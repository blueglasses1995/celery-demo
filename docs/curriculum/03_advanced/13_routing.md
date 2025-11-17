# 13. タスクの優先度とルーティング

## 学習目標

- 複数のキューを使い分ける
- タスクを適切なキューにルーティングする
- 優先度を設定する

## キューの概念

```
[Producer] → [高優先度キュー] → [専用ワーカー]
           → [通常キュー]     → [通常ワーカー]
           → [低優先度キュー] → [バッチワーカー]
```

## キューの設定

```python
from kombu import Queue, Exchange

app.conf.task_queues = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('high_priority', Exchange('high'), routing_key='high'),
    Queue('low_priority', Exchange('low'), routing_key='low'),
)

app.conf.task_default_queue = 'default'
```

## タスクのルーティング

### 静的ルーティング

```python
app.conf.task_routes = {
    'myapp.tasks.send_email': {'queue': 'high_priority'},
    'myapp.tasks.cleanup': {'queue': 'low_priority'},
    'myapp.tasks.*': {'queue': 'default'},
}
```

### 動的ルーティング

```python
def route_task(name, args, kwargs, options, task=None, **kw):
    if 'urgent' in kwargs:
        return {'queue': 'high_priority'}
    elif 'batch' in kwargs:
        return {'queue': 'low_priority'}
    return {'queue': 'default'}

app.conf.task_routes = (route_task,)
```

### 実行時にキューを指定

```python
# 高優先度キューに送信
send_email.apply_async(args=[user_id], queue='high_priority')

# 低優先度キューに送信
cleanup.apply_async(queue='low_priority')
```

## ワーカーのキュー指定

```bash
# 高優先度キューのみ処理
celery -A celery_app worker -Q high_priority -c 4

# 複数キューを処理
celery -A celery_app worker -Q high_priority,default -c 8

# すべてのキューを処理
celery -A celery_app worker -c 4
```

## 優先度

```python
# RabbitMQで優先度を有効化
app.conf.task_queue_max_priority = 10

# タスクに優先度を設定
task.apply_async(args=[data], priority=9)  # 最高優先度
task.apply_async(args=[data], priority=0)  # 最低優先度
```

## まとめ

- ✅ 複数キューでタスクを分類
- ✅ ルーティングで適切なワーカーに振り分け
- ✅ 優先度で実行順序を制御

## 次のステップ

次のセクション「[パフォーマンス最適化](./14_performance.md)」では、最適化手法を学びます。
