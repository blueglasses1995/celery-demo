# 8. メッセージブローカー

## 学習目標

- Redis、RabbitMQの特徴と違いを理解する
- 各ブローカーの設定方法を習得する
- 結果バックエンドの選択と設定を理解する
- ブローカーのベストプラクティスを学ぶ

## メッセージブローカーの役割

ブローカーは、タスクメッセージを保管・配送する中央ハブです。

```
[Producer] → [Broker] → [Worker]
              ↓
          [Queue]
```

## Redis

### 特徴

**利点:**
- セットアップが簡単
- 高速（メモリベース）
- Pythonとの相性が良い
- 結果バックエンドとしても使用可能

**欠点:**
- メモリベースなので永続性に限界
- 優先度キューのサポートが限定的
- 高度な機能が少ない

### 接続設定

```python
# celery_app.py
app = Celery(
    'myapp',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'  # 別のDB番号を使用
)
```

### 認証付きRedis

```python
# パスワード付き
broker_url = 'redis://:password@localhost:6379/0'

# ユーザー名とパスワード
broker_url = 'redis://username:password@localhost:6379/0'
```

### Redis Sentinel

高可用性構成:

```python
app.conf.broker_url = 'sentinel://localhost:26379;sentinel://localhost:26380'
app.conf.broker_transport_options = {
    'master_name': 'mymaster',
    'sentinel_kwargs': {
        'password': 'sentinel_password'
    },
    'password': 'redis_password'
}
```

### Redis Cluster

```python
from redis import RedisCluster

app.conf.broker_url = 'redis://localhost:7000/0'
app.conf.broker_transport_options = {
    'cluster_nodes': [
        {'host': 'localhost', 'port': 7000},
        {'host': 'localhost', 'port': 7001},
        {'host': 'localhost', 'port': 7002},
    ]
}
```

### Redis設定オプション

```python
app.conf.update(
    # 可視性タイムアウト（タスク処理中のタイムアウト）
    broker_transport_options={
        'visibility_timeout': 3600,  # 1時間
    },

    # 接続プール設定
    broker_pool_limit=10,

    # 接続タイムアウト
    broker_connection_timeout=30,

    # 再接続
    broker_connection_retry=True,
    broker_connection_max_retries=10,
)
```

## RabbitMQ

### 特徴

**利点:**
- 高い信頼性
- 高度な機能（優先度キュー、メッセージTTL等）
- 永続化が得意
- モニタリングツールが充実

**欠点:**
- セットアップがやや複雑
- Redisより遅い
- メモリ使用量が多い

### 接続設定

```python
# celery_app.py
app = Celery(
    'myapp',
    broker='amqp://guest:guest@localhost:5672//',
    backend='rpc://'  # RPC結果バックエンド
)
```

### カスタム仮想ホストとユーザー

```python
# カスタムvhost
broker_url = 'amqp://myuser:mypass@localhost:5672/myvhost'

# SSLを使用
broker_url = 'amqps://myuser:mypass@rabbitmq.example.com:5671//'
broker_use_ssl = {
    'keyfile': '/path/to/key.pem',
    'certfile': '/path/to/cert.pem',
    'ca_certs': '/path/to/ca.pem',
    'cert_reqs': ssl.CERT_REQUIRED
}
```

### RabbitMQ設定オプション

```python
app.conf.update(
    # 接続設定
    broker_connection_retry=True,
    broker_connection_retry_on_startup=True,
    broker_connection_max_retries=10,

    # プリフェッチ設定
    worker_prefetch_multiplier=4,

    # ACK設定
    task_acks_late=True,
    task_reject_on_worker_lost=True,

    # ハートビート
    broker_heartbeat=30,
)
```

### 優先度キュー

```python
# 設定
app.conf.task_queue_max_priority = 10

# タスクに優先度を設定
@app.task
def high_priority_task():
    pass

high_priority_task.apply_async(priority=9)
```

### メッセージTTL

```python
# キュー全体のTTL設定
from kombu import Queue, Exchange

app.conf.task_queues = (
    Queue(
        'default',
        Exchange('default'),
        routing_key='default',
        queue_arguments={'x-message-ttl': 60000}  # 60秒
    ),
)

# 個別タスクのexpires
task.apply_async(args=[data], expires=300)  # 5分
```

## Amazon SQS

クラウドネイティブなメッセージキュー。

### 設定

```bash
pip install celery[sqs]
pip install boto3
```

```python
app = Celery(
    'myapp',
    broker='sqs://',
    backend='s3://my-bucket/'
)

app.conf.update(
    broker_transport_options={
        'region': 'us-east-1',
        'queue_name_prefix': 'celery-'
    },

    # AWS認証情報（環境変数推奨）
    # AWS_ACCESS_KEY_ID
    # AWS_SECRET_ACCESS_KEY
)
```

## 結果バックエンド

### Redis Backend

```python
app.conf.result_backend = 'redis://localhost:6379/1'
```

**利点:**
- 高速
- セットアップ簡単

**欠点:**
- メモリベース
- 長期保存には不向き

### RPC Backend (RabbitMQ)

```python
app.conf.result_backend = 'rpc://'
```

**利点:**
- RabbitMQ使用時に追加設定不要
- 軽量

**欠点:**
- 一時的な結果のみ
- タスク完了後すぐに消える

### Database Backend

```python
# SQLite
app.conf.result_backend = 'db+sqlite:///results.db'

# PostgreSQL
app.conf.result_backend = 'db+postgresql://user:pass@localhost/celery'

# MySQL
app.conf.result_backend = 'db+mysql://user:pass@localhost/celery'
```

**利点:**
- 永続化
- 複雑なクエリが可能

**欠点:**
- 遅い
- データベース負荷

### MongoDB Backend

```python
app.conf.result_backend = 'mongodb://localhost:27017/celery'
```

### S3 Backend

```python
app.conf.result_backend = 's3://mybucket/'
```

### 結果バックエンドの選択ガイド

| ユースケース | 推奨バックエンド |
|------------|----------------|
| 一時的な結果のみ | RPC (RabbitMQ) |
| 高速な結果取得 | Redis |
| 長期保存が必要 | Database |
| 監査ログとして | Database, MongoDB |
| 大きな結果 | S3 |
| 結果不要 | ignore_result=True |

## ブローカーの比較

### Redis vs RabbitMQ

| 特徴 | Redis | RabbitMQ |
|------|-------|----------|
| セットアップ | ★★★★★ 簡単 | ★★★☆☆ やや複雑 |
| 速度 | ★★★★★ 高速 | ★★★★☆ 高速 |
| 信頼性 | ★★★☆☆ 良い | ★★★★★ 優秀 |
| 永続性 | ★★★☆☆ 限定的 | ★★★★★ 優秀 |
| メモリ使用 | ★★★★☆ 効率的 | ★★★☆☆ やや多い |
| 優先度キュー | ★★☆☆☆ 限定的 | ★★★★★ 完全 |
| モニタリング | ★★★☆☆ 基本的 | ★★★★★ 充実 |

### 選択ガイド

**Redisを選ぶ場合:**
- 開発環境・小規模
- シンプルなタスクキュー
- 高速性が最優先
- セットアップの簡単さ重視

**RabbitMQを選ぶ場合:**
- 本番環境・大規模
- 高い信頼性が必要
- 優先度キューが必要
- 複雑なルーティングが必要

## 高可用性構成

### Redis Sentinel

```python
app.conf.broker_url = 'sentinel://localhost:26379'
app.conf.broker_transport_options = {
    'master_name': 'mymaster',
    'sentinel_kwargs': {
        'socket_timeout': 0.1
    }
}
```

### RabbitMQ クラスター

```python
app.conf.broker_url = 'amqp://user:pass@host1:5672;amqp://user:pass@host2:5672//'
```

## モニタリング

### RabbitMQ Management

```bash
# Management pluginを有効化
rabbitmq-plugins enable rabbitmq_management

# http://localhost:15672 にアクセス
```

確認できる情報:
- キューの長さ
- メッセージレート
- 接続数
- メモリ使用量

### Redis Monitor

```bash
# Redis CLIでモニタリング
redis-cli monitor

# キーの確認
redis-cli keys "celery*"

# キューの長さ
redis-cli llen celery
```

## トラブルシューティング

### 接続エラー

```python
# 接続のリトライ設定
app.conf.update(
    broker_connection_retry=True,
    broker_connection_retry_on_startup=True,
    broker_connection_max_retries=10,
)
```

### タスクの喪失

```python
# Late ACKを有効化
app.conf.update(
    task_acks_late=True,
    task_reject_on_worker_lost=True,
)
```

### パフォーマンス問題

```python
# プリフェッチ設定
app.conf.update(
    worker_prefetch_multiplier=1,  # 一度に1タスクのみ
)
```

## まとめ

- ✅ Redis: 高速・シンプル、開発環境向き
- ✅ RabbitMQ: 高信頼・高機能、本番環境向き
- ✅ 結果バックエンドは用途に応じて選択
- ✅ 高可用性にはSentinel/クラスター構成

## 次のステップ

次のセクション「[Celeryの設定](./09_configuration.md)」では、設定管理について学びます。
