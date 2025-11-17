# 19. デプロイメント

## 学習目標

- プロセス管理ツール（Supervisor、systemd）を使う
- Dockerでのデプロイ
- クラウド環境でのデプロイ

## Supervisorでのデプロイ

### インストール

```bash
sudo apt-get install supervisor
```

### 設定ファイル

```ini
; /etc/supervisor/conf.d/celery.conf
[program:celery-worker]
command=/path/to/venv/bin/celery -A celery_app worker --loglevel=info
directory=/path/to/project
user=celery
numprocs=1
stdout_logfile=/var/log/celery/worker.log
stderr_logfile=/var/log/celery/worker_err.log
autostart=true
autorestart=true
startsecs=10
stopwaitsecs=600

[program:celery-beat]
command=/path/to/venv/bin/celery -A celery_app beat --loglevel=info
directory=/path/to/project
user=celery
numprocs=1
stdout_logfile=/var/log/celery/beat.log
stderr_logfile=/var/log/celery/beat_err.log
autostart=true
autorestart=true
startsecs=10
```

### 起動

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start celery-worker
sudo supervisorctl start celery-beat
```

## systemdでのデプロイ

### ワーカーサービス

```ini
# /etc/systemd/system/celery-worker.service
[Unit]
Description=Celery Worker
After=network.target

[Service]
Type=forking
User=celery
Group=celery
EnvironmentFile=/etc/celery/celery.conf
WorkingDirectory=/path/to/project
ExecStart=/path/to/venv/bin/celery -A celery_app worker \
    --loglevel=info \
    --logfile=/var/log/celery/worker.log \
    --pidfile=/var/run/celery/worker.pid \
    --detach
ExecStop=/path/to/venv/bin/celery -A celery_app control shutdown
Restart=always

[Install]
WantedBy=multi-user.target
```

### Beatサービス

```ini
# /etc/systemd/system/celery-beat.service
[Unit]
Description=Celery Beat
After=network.target

[Service]
Type=simple
User=celery
Group=celery
WorkingDirectory=/path/to/project
ExecStart=/path/to/venv/bin/celery -A celery_app beat \
    --loglevel=info \
    --logfile=/var/log/celery/beat.log \
    --schedule=/var/run/celery/celerybeat-schedule
Restart=always

[Install]
WantedBy=multi-user.target
```

### 起動

```bash
sudo systemctl daemon-reload
sudo systemctl enable celery-worker celery-beat
sudo systemctl start celery-worker celery-beat
sudo systemctl status celery-worker
```

## Dockerでのデプロイ

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["celery", "-A", "celery_app", "worker", "--loglevel=info"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  celery-worker:
    build: .
    command: celery -A celery_app worker --loglevel=info
    volumes:
      - .:/app
    depends_on:
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

  celery-beat:
    build: .
    command: celery -A celery_app beat --loglevel=info
    volumes:
      - .:/app
    depends_on:
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0

  flower:
    build: .
    command: celery -A celery_app flower
    ports:
      - "5555:5555"
    depends_on:
      - redis
      - celery-worker
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
```

### 起動

```bash
docker-compose up -d
```

## Kubernetesでのデプロイ

### ワーカーDeployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: celery-worker
  template:
    metadata:
      labels:
        app: celery-worker
    spec:
      containers:
      - name: worker
        image: myapp:latest
        command: ["celery", "-A", "celery_app", "worker", "--loglevel=info"]
        env:
        - name: CELERY_BROKER_URL
          valueFrom:
            secretKeyRef:
              name: celery-secrets
              key: broker-url
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## まとめ

- ✅ Supervisorまたはsystemdでプロセス管理
- ✅ Dockerで簡単デプロイ
- ✅ Kubernetesでスケーラブルな運用
- ✅ 環境変数で設定を管理

## 次のステップ

次のセクション「[ベストプラクティス](./20_best_practices.md)」では、実践的なノウハウを学びます。
