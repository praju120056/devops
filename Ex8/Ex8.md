# Exercise 8: Creating a "Hello World" Jenkins Job

## Problem Statement

Create a Jenkins Freestyle job that pulls a `hello-world.sh` script from a GitHub repository and runs it as a build step. Verify the result in the build's console output.

**Environment:** Windows, VS Code integrated terminal (Command Prompt), Docker Desktop, Jenkins (Docker) on port **8082**

---

## Setup Notes

- The script `hello-world.sh` is stored in the `Ex8` sub-directory of the existing exercises repository, so the build step uses the path `Ex8/hello-world.sh`.
- Jenkins is exposed on host port **8082** (`-p 8082:8080`), so it is accessed at `http://localhost:8082`.
- In the Jenkins job, the branch specifier is changed from `*/master` to `*/main`.

**hello-world.sh** (in `Ex8/`)

```bash
#!/bin/bash
echo "Hello, Jenkins!"
```

---

## Solution (Brief)

1. Run Jenkins in Docker on port 8082 and complete the initial setup.
2. Create a Freestyle job named `HelloWorld`.
3. Configure Git as the source (the exercises repository, branch `main`) and add an **Execute shell** build step that runs `sh Ex8/hello-world.sh`.
4. Build the job and verify `Hello, Jenkins!` in the console output.

---

## Commands Executed and Output

### Step 1: Run Jenkins on port 8082

Start Jenkins with host port 8082 mapped to Jenkins' port 8080 inside the container:

```cmd
docker run -d --name jenkins -p 8082:8080 -p 50000:50000 jenkins/jenkins:lts
```

```
eeba8e46d854a5ce99c387caa0cee56c88f957b8097c1caf08c3e153f7ccd17f
```

```cmd
docker ps
```

```
CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                                                          NAMES
eeba8e46d854   jenkins/jenkins:lts   "/usr/bin/tini -- /u…"   22 minutes ago   Up 22 minutes   0.0.0.0:50000->50000/tcp, [::]:50000->50000/tcp, 0.0.0.0:8082->8080/tcp, [::]:8082->8080/tcp   jenkins
```

Get the initial admin password (wait about a minute for Jenkins to start):

```cmd
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

```
Password :)
```

### Step 2: Access Jenkins

Open `http://localhost:8082`, unlock Jenkins with the password, install the suggested plugins, and create the admin user.

### Step 3: Confirm the script is available on GitHub

Jenkins pulls the script from GitHub, so the repository must contain `Ex8/hello-world.sh` on the `main` branch. List the branches of the remote repository:

```cmd
git ls-remote --heads https://github.com/PraneethUpadhya195/devops.git
```

```
69b92bfc49b8b83dc33d233cc6c41c6b2be0b53e        refs/heads/main
```

### Step 4: Create a new job

1. On the Jenkins dashboard, click **New Item**.
2. Enter the name `HelloWorld`.
3. Select **Freestyle project** and click **OK**.

### Step 5: Configure the job

- **Description:** `Hello World! Jenkins job.`
- **Source Code Management:** select **Git**
  - Repository URL: `https://github.com/PraneethUpadhya195/devops.git`
  - Credentials: none for a public repository (for a private repository, add a credential using your GitHub username and personal access token)
  - Branch Specifier: `*/main`
- **Build Steps:** **Add build step -> Execute shell**, with this command:

```
sh Ex8/hello-world.sh
```

Click **Save**.

### Step 6: Run the job

On the job page (Dashboard -> HelloWorld), click **Build Now**.

### Step 7: View the build output

Click the build number (e.g., `#1`) in **Build History**, then click **Console Output**:

```
Started by user admin

Running as SYSTEM
Building in workspace /var/jenkins_home/workspace/HelloWorld
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/HelloWorld/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/PraneethUpadhya195/devops.git # timeout=10
Fetching upstream changes from https://github.com/PraneethUpadhya195/devops.git
 > git --version # timeout=10
 > git --version # 'git version 2.47.3'
 > git fetch --tags --force --progress -- https://github.com/PraneethUpadhya195/devops.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision 69b92bfc49b8b83dc33d233cc6c41c6b2be0b53e (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 69b92bfc49b8b83dc33d233cc6c41c6b2be0b53e # timeout=10
Commit message: "Ex8 - .sh"
First time build. Skipping changelog.
[HelloWorld] $ /bin/sh -xe /tmp/jenkins5222641086954692838.sh
+ sh Ex8/hello-world.sh
Hello, Jenkins!
Finished: SUCCESS

```

---

## Result

The Jenkins `HelloWorld` Freestyle job (running on port 8082) pulled the repository from GitHub, executed `sh Ex8/hello-world.sh`, and the console output showed `Hello, Jenkins!` with `Finished: SUCCESS`.