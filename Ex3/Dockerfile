FROM python:3.11-slim
WORKDIR /app
COPY flashsale.py .
RUN pip install --no-cache-dir flask gunicorn
CMD ["gunicorn","-b","0.0.0.0:5000","flashsale:app","--workers","1","--threads","2"]