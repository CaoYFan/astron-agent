FROM python:3.12-slim
WORKDIR /tool
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY passwords.py .
USER 65534:65534
ENTRYPOINT ["python", "/tool/passwords.py"]
