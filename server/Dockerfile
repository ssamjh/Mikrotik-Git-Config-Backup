FROM python:3.14-slim

# git is the only system dependency
RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py git_ops.py ./

# /data/repo  — mounted volume; git working tree lives here
RUN mkdir -p /data/repo

EXPOSE 8080

# Docker sends SIGTERM on `stop`; uvicorn shuts down on it (see app.py).
STOPSIGNAL SIGTERM

CMD ["python", "app.py"]
