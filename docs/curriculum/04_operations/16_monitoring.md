# 16. モニタリング

## 学習目標

- Flowerでワーカーとタスクを監視する
- イベントとメトリクスを収集する
- アラート設定を行う

## Flower - Celeryモニタリングツール

### インストールと起動

```bash
pip install flower

# 起動
celery -A celery_app flower

# ポートを指定
celery -A celery_app flower --port=5555

# ブラウザで http://localhost:5555 にアクセス
```

### 基本認証

```bash
celery -A celery_app flower --basic_auth=user:password
```

### Flowerで確認できる情報

- ワーカーの状態とリソース使用状況
- 実行中のタスク
- タスクの履歴と統計
- タスクの成功率・失敗率
- タスクの実行時間

## イベントモニタリング

### イベントの有効化

```bash
# ワーカー起動時にイベントを有効化
celery -A celery_app worker --events
```

### コードでイベントを監視

```python
from celery_app import app

def monitor_events():
    state = app.events.State()

    def on_task_received(event):
        state.event(event)
        task = state.tasks.get(event['uuid'])
        print(f'Task received: {task.name}')

    def on_task_succeeded(event):
        state.event(event)
        task = state.tasks.get(event['uuid'])
        print(f'Task succeeded: {task.name} - {task.runtime}s')

    def on_task_failed(event):
        state.event(event)
        task = state.tasks.get(event['uuid'])
        print(f'Task failed: {task.name} - {task.exception}')

    with app.connection() as connection:
        recv = app.events.Receiver(connection, handlers={
            'task-received': on_task_received,
            'task-succeeded': on_task_succeeded,
            'task-failed': on_task_failed,
        })
        recv.capture(limit=None, timeout=None, wakeup=True)

if __name__ == '__main__':
    monitor_events()
```

## メトリクス収集

### Prometheusとの統合

```bash
pip install celery-exporter
```

```bash
celery-exporter --broker-url=redis://localhost:6379/0
```

Prometheusで `http://localhost:9540/metrics` をスクレイプ。

### カスタムメトリクス

```python
import statsd

statsd_client = statsd.StatsClient('localhost', 8125)

@app.task
def monitored_task():
    with statsd_client.timer('task.execution_time'):
        result = perform_work()
        statsd_client.incr('task.success')
        return result
```

## ヘルスチェック

```python
from celery_app import app

def check_celery_health():
    """Celeryの健全性をチェック"""
    try:
        # ワーカーの状態を確認
        inspect = app.control.inspect()
        stats = inspect.stats()

        if not stats:
            return {'status': 'unhealthy', 'reason': 'No workers available'}

        active = inspect.active()
        return {
            'status': 'healthy',
            'workers': len(stats),
            'active_tasks': sum(len(tasks) for tasks in active.values())
        }
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}
```

## まとめ

- ✅ Flowerでリアルタイムモニタリング
- ✅ イベントでタスクを追跡
- ✅ Prometheusでメトリクス収集
- ✅ ヘルスチェックで異常検知

## 次のステップ

次のセクション「[ロギングとデバッグ](./17_logging.md)」では、デバッグ手法を学びます。
