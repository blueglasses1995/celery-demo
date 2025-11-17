# 9. Celeryの設定

## 学習目標

- 設定ファイルの作成と構造化を理解する
- 重要な設定項目を把握する
- 環境別の設定管理を習得する
- ベストプラクティスを学ぶ

## 設定方法

### 1. 直接設定

```python
# celery_app.py
from celery import Celery

app = Celery('myapp')
app.conf.broker_url = 'redis://localhost:6379/0'
app.conf.result_backend = 'redis://localhost:6379/1'
app.conf.task_serializer = 'json'
```

### 2. update()で一括設定

```python
app.conf.update(
    broker_url='redis://localhost:6379/0',
    result_backend='redis://localhost:6379/1',
    task_serializer='json',
    result_serializer='json',
    timezone='Asia/Tokyo',
    enable_utc=True,
)
```

### 3. 設定ファイルから読み込み

```python
# config.py
broker_url = 'redis://localhost:6379/0'
result_backend = 'redis://localhost:6379/1'
task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
timezone = 'Asia/Tokyo'
enable_utc = True

# celery_app.py
app.conf.config_from_object('config')
```

### 4. クラスベースの設定

```python
# config.py
class Config:
    broker_url = 'redis://localhost:6379/0'
    result_backend = 'redis://localhost:6379/1'
    task_serializer = 'json'
    result_serializer = 'json'
    accept_content = ['json']
    timezone = 'Asia/Tokyo'
    enable_utc = True

class DevelopmentConfig(Config):
    broker_url = 'redis://localhost:6379/0'
    task_always_eager = True  # 同期実行（テスト用）

class ProductionConfig(Config):
    broker_url = 'redis://production-redis:6379/0'
    worker_prefetch_multiplier = 1
    task_acks_late = True

# celery_app.py
import os
env = os.getenv('ENV', 'development')

if env == 'production':
    app.conf.config_from_object('config.ProductionConfig')
else:
    app.conf.config_from_object('config.DevelopmentConfig')
```

## 重要な設定項目

### ブローカー設定

```python
# ブローカーURL
broker_url = 'redis://localhost:6379/0'

# 接続リトライ
broker_connection_retry = True
broker_connection_retry_on_startup = True
broker_connection_max_retries = 10

# 接続プール
broker_pool_limit = 10

# タイムアウト
broker_connection_timeout = 30
```

### 結果バックエンド設定

```python
# 結果バックエンドURL
result_backend = 'redis://localhost:6379/1'

# 結果の有効期限（秒）
result_expires = 3600  # 1時間

# 拡張結果
result_extended = True

# バックエンドのタイムアウト
result_backend_transport_options = {
    'socket_timeout': 5.0
}
```

### タスク設定

```python
# タスクのシリアライザ
task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']

# タスクのタイムリミット
task_time_limit = 3600  # 1時間（ハードリミット）
task_soft_time_limit = 3000  # 50分（ソフトリミット）

# タスクのACK
task_acks_late = True  # タスク完了後にACK
task_reject_on_worker_lost = True  # ワーカー停止時に拒否

# 結果を保存しない
task_ignore_result = False  # デフォルト: False

# タスク実行開始を追跡
task_track_started = True
```

### ワーカー設定

```python
# プリフェッチ設定
worker_prefetch_multiplier = 4  # デフォルト: 4

# ワーカープールの実装
worker_pool = 'prefork'  # prefork, gevent, eventlet, threads

# 並行処理数
worker_concurrency = 4

# タスク実行後にワーカーを再起動
worker_max_tasks_per_child = 1000

# メモリ制限
worker_max_memory_per_child = 200000  # KB (200MB)
```

### タイムゾーン設定

```python
# タイムゾーン
timezone = 'Asia/Tokyo'

# UTCを有効化
enable_utc = True
```

### セキュリティ設定

```python
# メッセージ署名
task_serializer = 'json'
accept_content = ['json']  # pickleを無効化

# タスク名のホワイトリスト
imports = ('myapp.tasks',)
```

## 環境別設定管理

### .envファイルの使用

```bash
# .env
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/1
CELERY_TIMEZONE=Asia/Tokyo
```

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    broker_url = os.getenv('CELERY_BROKER_URL')
    result_backend = os.getenv('CELERY_RESULT_BACKEND')
    timezone = os.getenv('CELERY_TIMEZONE', 'UTC')
```

### 環境別設定ファイル

```python
# config/development.py
broker_url = 'redis://localhost:6379/0'
task_always_eager = True  # 同期実行
task_eager_propagates = True  # エラーを伝播

# config/production.py
broker_url = 'redis://production:6379/0'
worker_prefetch_multiplier = 1
task_acks_late = True
task_reject_on_worker_lost = True
result_expires = 3600

# celery_app.py
import os
env = os.getenv('ENV', 'development')
app.conf.config_from_object(f'config.{env}')
```

## 実践的な設定例

### 開発環境

```python
# config/development.py
# ブローカー
broker_url = 'redis://localhost:6379/0'

# 結果バックエンド
result_backend = 'redis://localhost:6379/1'
result_expires = 600  # 10分

# タスク
task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
task_track_started = True
task_always_eager = False  # False推奨（実環境に近い）

# ワーカー
worker_prefetch_multiplier = 4
worker_max_tasks_per_child = 100

# タイムゾーン
timezone = 'Asia/Tokyo'
enable_utc = True

# ログ
worker_log_format = '[%(asctime)s: %(levelname)s/%(processName)s] %(message)s'
```

### 本番環境

```python
# config/production.py
import os

# ブローカー
broker_url = os.getenv('CELERY_BROKER_URL')
broker_connection_retry = True
broker_connection_max_retries = 10
broker_pool_limit = 50

# 結果バックエンド
result_backend = os.getenv('CELERY_RESULT_BACKEND')
result_expires = 3600
result_extended = True

# タスク
task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
task_track_started = True
task_time_limit = 3600
task_soft_time_limit = 3000
task_acks_late = True
task_reject_on_worker_lost = True

# ワーカー
worker_prefetch_multiplier = 1  # 長時間タスク用
worker_max_tasks_per_child = 1000
worker_max_memory_per_child = 300000  # 300MB
worker_disable_rate_limits = False

# タイムゾーン
timezone = 'Asia/Tokyo'
enable_utc = True

# セキュリティ
imports = ('myapp.tasks',)

# モニタリング
worker_send_task_events = True
task_send_sent_event = True
```

### テスト環境

```python
# config/test.py
# 同期実行
task_always_eager = True
task_eager_propagates = True

# メモリバックエンド
broker_url = 'memory://'
result_backend = 'cache+memory://'

# タスク
task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
```

## キューとルーティング設定

### キューの定義

```python
from kombu import Queue, Exchange

# キューの定義
task_queues = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('high_priority', Exchange('high_priority'), routing_key='high'),
    Queue('low_priority', Exchange('low_priority'), routing_key='low'),
)

# デフォルトキュー
task_default_queue = 'default'
task_default_exchange = 'default'
task_default_routing_key = 'default'
```

### ルーティング

```python
# タスクごとのルーティング
task_routes = {
    'myapp.tasks.send_email': {'queue': 'high_priority'},
    'myapp.tasks.cleanup': {'queue': 'low_priority'},
    'myapp.tasks.*': {'queue': 'default'},
}

# または関数で動的ルーティング
def route_task(name, args, kwargs, options, task=None, **kw):
    if ':high' in name:
        return {'queue': 'high_priority'}
    return {'queue': 'default'}

task_routes = (route_task,)
```

## Beat（スケジューラー）設定

```python
from celery.schedules import crontab

beat_schedule = {
    # 毎日午前2時に実行
    'cleanup-daily': {
        'task': 'myapp.tasks.cleanup',
        'schedule': crontab(hour=2, minute=0),
    },

    # 5分ごとに実行
    'check-status': {
        'task': 'myapp.tasks.check_status',
        'schedule': 300.0,  # 秒
    },

    # 毎週月曜日の午前9時
    'weekly-report': {
        'task': 'myapp.tasks.generate_report',
        'schedule': crontab(hour=9, minute=0, day_of_week=1),
        'args': ('weekly',),
    },
}

beat_schedule_filename = '/var/run/celery/beat-schedule'
```

## ロギング設定

```python
# ログフォーマット
worker_log_format = '[%(asctime)s: %(levelname)s/%(processName)s] %(message)s'
worker_task_log_format = '[%(asctime)s: %(levelname)s/%(processName)s][%(task_name)s(%(task_id)s)] %(message)s'

# ログレベル
worker_log_color = True
worker_redirect_stdouts = True
worker_redirect_stdouts_level = 'INFO'
```

## パフォーマンス設定

```python
# 最適化されたJSONシリアライザ
task_serializer = 'json'

# 圧縮
task_compression = 'gzip'
result_compression = 'gzip'

# プリフェッチ最適化
worker_prefetch_multiplier = 1  # 長時間タスク
# worker_prefetch_multiplier = 4  # 短時間タスク

# レート制限を無効化（パフォーマンス向上）
worker_disable_rate_limits = True  # レート制限が不要な場合
```

## まとめ

- ✅ 設定は`config.py`で一元管理
- ✅ 環境別に設定ファイルを分ける
- ✅ 環境変数で機密情報を管理
- ✅ 本番環境では`task_acks_late=True`を推奨
- ✅ キューとルーティングで処理を分散

## 次のステップ

実践編はこれで完了です！次は「[応用編](../03_advanced/10_scheduling.md)」に進み、高度な機能を学びます。
