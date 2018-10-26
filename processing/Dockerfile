FROM python:3.7

ENV PYTHONUNBUFFERED 1
COPY requirements_processing.txt .
RUN pip install -r requirements_processing.txt
COPY processing.py .
EXPOSE 5672

CMD ["bash", "wait_db.sh"]
CMD ["python", "processing.py"]
