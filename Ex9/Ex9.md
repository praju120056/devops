# Exercise 9: Jenkins Multi-Stage Pipeline - Deploying a Python Application

## Problem Statement

Automate the CI/CD pipeline for a Python Flask application using a Jenkins **Pipeline** with multiple stages:

1. **Build:** prepare the environment and dependencies.
2. **Test:** run unit tests to make sure the application works as expected.
3. **Deploy:** deploy the application by copying it to a mock deployment directory.
4. **Run Application / Test Application:** start the deployed app and run the tests again.

**Environment:** Windows, VS Code integrated terminal (Command Prompt), Docker Desktop, Jenkins (Docker) on port **8082**

---

## Setup Notes

- Jenkins is exposed on host port **8082** (`-p 8082:8080`), so it is accessed at `http://localhost:8082`.
- The application files and the `Jenkinsfile` are stored in the `Ex9` sub-directory of the existing exercises repository (`https://github.com/PraneethUpadhya195/devops.git`).
- Because the files are in a sub-directory, the Jenkinsfile runs its steps inside `dir('Ex9')`, and the job's **Script Path** is `Ex9/Jenkinsfile`.
- The `jenkins/jenkins:lts` image does not include Python, so Python 3, pip, venv and Flask are installed inside the Jenkins container (Step 2).

---

## Solution (Brief)

1. Make sure Jenkins is running on port 8082.
2. Install Python 3 and Flask inside the Jenkins container.
3. Create the Flask app, its unit test, and the `Jenkinsfile` in the `Ex9` folder and push them to GitHub.
4. Create a Pipeline job that reads the `Jenkinsfile` from the Git repository (Pipeline script from SCM).
5. Run the pipeline and verify that all stages pass.

---

## Files Created (in the `Ex9` folder)

```cmd
mkdir Ex9
cd Ex9
```

```
Ex9/
├── app.py
├── requirements.txt
├── test_app.py
└── Jenkinsfile
```

**app.py**

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, Jenkins Multi-Stage Pipeline!"

if __name__ == "__main__":
    app.run(debug=True)
```

**requirements.txt**

```
flask==2.1.2
```

**test_app.py**

```python
import unittest
from app import app

class TestApp(unittest.TestCase):
    def test_home(self):
        tester = app.test_client()
        response = tester.get("/")
        print(response.data.decode("utf-8"))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.data.decode("utf-8"), "Hello, Jenkins Multi-Stage Pipeline!")

if __name__ == "__main__":
    unittest.main()
```

**Jenkinsfile**

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Creating virtual environment and installing dependencies...'
                dir('Ex9') {
                    sh 'python3 --version'
                }
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                dir('Ex9') {
                    sh 'python3 -m unittest discover -s .'
                }
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                dir('Ex9') {
                    sh '''
                    mkdir -p python-app-deploy
                    cp app.py python-app-deploy/
                    '''
                }
            }
        }
        stage('Run Application') {
            steps {
                echo 'Running application...'
                dir('Ex9') {
                    sh '''
                    nohup python3 python-app-deploy/app.py > python-app-deploy/app.log 2>&1 &
                    echo $! > python-app-deploy/app.pid
                    '''
                }
            }
        }
        stage('Test Application') {
            steps {
                echo 'Testing application...'
                dir('Ex9') {
                    sh 'python3 test_app.py'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for more details.'
        }
    }
}
```

---

## Commands Executed and Output

### Step 1: Make sure Jenkins is running on port 8082

```cmd
docker ps
```

```
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS         PORTS                                                                                          NAMES
a1ae4dbe87a9   jenkins/jenkins:lts   "/usr/bin/tini -- /u…"   7 minutes ago   Up 7 minutes   0.0.0.0:50000->50000/tcp, [::]:50000->50000/tcp, 0.0.0.0:8082->8080/tcp, [::]:8082->8080/tcp   jenkins
```

If the container exists but is stopped, start it with `docker start jenkins`. If it does not exist, create it:

```cmd
docker run -d --name jenkins -p 8082:8080 -p 50000:50000 jenkins/jenkins:lts
```

Open `http://localhost:8082` and log in. The **Pipeline** plugin is included in the suggested plugins.

### Step 2: Install Python and Flask inside the Jenkins container

The pipeline fails with missing `python3` and `pip` errors unless they are installed. Run the install as the `root` user inside the Jenkins container:

```cmd
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y python3 python3-pip python3-venv python3-flask"
```


Verify the installation:

```cmd
docker exec jenkins python3 --version
```

```
Python 3.13.5
```

```cmd
docker exec jenkins python3 -c "import flask; print('Flask OK')"
```

```
Flask OK
```

### Step 3: Push the application and Jenkinsfile to GitHub

Jenkins reads the `Jenkinsfile` from GitHub, so the `Ex9` folder must be pushed before the pipeline is run. From the root of the repository:

```cmd
git add Ex9
git commit -m "Add Exercise 9 Jenkins multi-stage pipeline"
git push
```

### Step 4: Create the Pipeline job

1. On the Jenkins dashboard, click **New Item**.
2. Enter the name `Python-MultiStage-Pipeline`, select **Pipeline**, and click **OK**.

### Step 5: Configure the job

Under **Pipeline**:
- **Definition:** Pipeline script from SCM
- **SCM:** Git
- **Repository URL:** `https://github.com/PraneethUpadhya195/devops.git`
- **Credentials:** none for a public repository
- **Branch Specifier:** `*/main`
- **Script Path:** `Ex9/Jenkinsfile`

Click **Save**.

### Step 6: Run the pipeline

On the job page, click **Build Now**. Open the build number in **Build History**, then click **Console Output**:

```
Started by user admin

Obtained Ex9/Jenkinsfile from git https://github.com/PraneethUpadhya195/devops.git
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins
 in /var/jenkins_home/workspace/Python-Multistage-Pipeline
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
Cloning the remote Git repository
Cloning repository https://github.com/PraneethUpadhya195/devops.git
 > git init /var/jenkins_home/workspace/Python-Multistage-Pipeline # timeout=10
Fetching upstream changes from https://github.com/PraneethUpadhya195/devops.git
 > git --version # timeout=10
 > git --version # 'git version 2.47.3'
 > git fetch --tags --force --progress -- https://github.com/PraneethUpadhya195/devops.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git config remote.origin.url https://github.com/PraneethUpadhya195/devops.git # timeout=10
 > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
Avoid second fetch
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision 87f92c7c49c4f5de64d477cbf78a1d83926ac92e (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 87f92c7c49c4f5de64d477cbf78a1d83926ac92e # timeout=10
Commit message: "Temp Ex9"
First time build. Skipping changelog.
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Build)
[Pipeline] echo
Creating virtual environment and installing dependencies...
[Pipeline] dir
Running in /var/jenkins_home/workspace/Python-Multistage-Pipeline/Ex9
[Pipeline] {
[Pipeline] sh
+ python3 --version
Python 3.13.5
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Test)
[Pipeline] echo
Running tests...
[Pipeline] dir
Running in /var/jenkins_home/workspace/Python-Multistage-Pipeline/Ex9
[Pipeline] {
[Pipeline] sh
+ python3 -m unittest discover -s .
.
----------------------------------------------------------------------
Ran 1 test in 0.009s

OK
Hello, Jenkins Multi-Stage Pipeline!
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] echo
Deploying application...
[Pipeline] dir
Running in /var/jenkins_home/workspace/Python-Multistage-Pipeline/Ex9
[Pipeline] {
[Pipeline] sh
+ mkdir -p python-app-deploy
+ cp app.py python-app-deploy/
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Run Application)
[Pipeline] echo
Running application...
[Pipeline] dir
Running in /var/jenkins_home/workspace/Python-Multistage-Pipeline/Ex9
[Pipeline] {
[Pipeline] sh
+ echo 2547
+ nohup python3 python-app-deploy/app.py
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Test Application)
[Pipeline] echo
Testing application...
[Pipeline] dir
Running in /var/jenkins_home/workspace/Python-Multistage-Pipeline/Ex9
[Pipeline] {
[Pipeline] sh
+ python3 test_app.py
.
----------------------------------------------------------------------
Ran 1 test in 0.004s

OK
Hello, Jenkins Multi-Stage Pipeline!
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Declarative: Post Actions)
[Pipeline] echo
Pipeline completed successfully!
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS

```

### Step 7: Verify the deployed files

The Deploy and Run Application stages create the `python-app-deploy` folder in the job workspace. List its contents inside the Jenkins container:

```cmd
docker exec jenkins ls -l /var/jenkins_home/workspace/Python-MultiStage-Pipeline/Ex9/python-app-deploy
```

```
total 12
-rw-r--r-- 1 jenkins jenkins 319 Oct 17 15:17 app.log
-rw-r--r-- 1 jenkins jenkins   5 Oct 17 15:17 app.pid
-rw-r--r-- 1 jenkins jenkins 178 Oct 17 15:17 app.py
```

---

## Result

The Jenkins pipeline `Python-MultiStage-Pipeline` (running on port 8082) pulled the `Jenkinsfile` and application from GitHub and ran the Build, Test, Deploy, Run Application, and Test Application stages successfully. The unit test passed, `app.py` was copied to the `python-app-deploy` directory, and the pipeline ended with `Pipeline completed successfully!` and `Finished: SUCCESS`.