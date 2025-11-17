# 20. ベストプラクティス

## タスク設計

### 1. タスクを小さく保つ

```python
# Good: 小さなタスク
@app.task
def process_user(user_id):
    user = get_user(user_id)
    return process(user)

# Avoid: 大きすぎるタスク
@app.task
def process_all_users():
    for user in User.objects.all():  # 数万件
        process(user)
```

### 2. べき等性を保つ

タスクを複数回実行しても結果が同じになるように設計。

```python
# Good: べき等
@app.task
def update_cache(key, value):
    cache.set(key, value)  # 何度実行しても同じ結果

# Avoid: べき等でない
@app.task
def increment_counter(key):
    cache.incr(key)  # 実行回数で結果が変わる
```

### 3. タスクに状態を持たせない

```python
# Good: ステートレス
@app.task
def send_email(user_id, message):
    user = get_user(user_id)
    send_mail(user.email, message)

# Avoid: グローバル状態に依存
counter = 0

@app.task
def stateful_task():
    global counter
    counter += 1  # 複数ワーカーで競合
```

## パフォーマンス

### 1. 結果が不要な場合は保存しない

```python
@app.task(ignore_result=True)
def send_notification(user_id):
    # 結果を保存しない
    send_push(user_id)
```

### 2. バッチ処理

```python
# Good: バッチ処理
@app.task
def process_batch(item_ids):
    items = Item.objects.filter(id__in=item_ids)
    for item in items:
        process(item)

# Avoid: 個別処理
for item_id in item_ids:
    process_item.delay(item_id)  # タスクが多すぎる
```

### 3. プリフェッチの調整

```python
# 長時間タスク
app.conf.worker_prefetch_multiplier = 1

# 短時間タスク
app.conf.worker_prefetch_multiplier = 4
```

## 信頼性

### 1. Late ACKを有効化

```python
app.conf.task_acks_late = True
app.conf.task_reject_on_worker_lost = True
```

### 2. リトライポリシー

```python
@app.task(
    bind=True,
    autoretry_for=(Exception,),
    retry_kwargs={'max_retries': 3},
    retry_backoff=True
)
def reliable_task(self):
    # 自動リトライ
    pass
```

### 3. タイムアウト設定

```python
@app.task(
    time_limit=300,  # ハードリミット
    soft_time_limit=240  # ソフトリミット
)
def bounded_task():
    pass
```

## セキュリティ

### 1. Pickleを避ける

```python
app.conf.task_serializer = 'json'
app.conf.accept_content = ['json']
```

### 2. 環境変数で機密情報

```python
import os
app.conf.broker_url = os.getenv('CELERY_BROKER_URL')
```

### 3. レート制限

```python
@app.task(rate_limit='10/m')
def api_call():
    pass
```

## モニタリング

### 1. ログを適切に設定

```python
import logging
logger = logging.getLogger(__name__)

@app.task
def monitored_task():
    logger.info('Task started')
    # 処理
    logger.info('Task completed')
```

### 2. メトリクスを収集

```python
@app.task(bind=True)
def tracked_task(self):
    start = time.time()
    try:
        result = perform_work()
        duration = time.time() - start
        metrics.timing('task.duration', duration)
        return result
    except Exception:
        metrics.incr('task.error')
        raise
```

## よくあるアンチパターン

### ❌ 1. データベースオブジェクトを直接渡す

```python
# Bad
user = User.objects.get(id=123)
task.delay(user)  # シリアライズできない

# Good
task.delay(user.id)  # IDを渡す
```

### ❌ 2. タスク内でタスクを同期実行

```python
# Bad
@app.task
def parent_task():
    result = child_task.delay().get()  # ブロッキング

# Good
from celery import chain
workflow = chain(parent_task.s(), child_task.s())
```

### ❌ 3. 無限ループ

```python
# Bad
@app.task
def infinite_task():
    while True:  # 終わらない
        do_work()

# Good
@app.task
def bounded_task(iterations):
    for i in range(iterations):
        do_work()
```

## まとめ

- ✅ タスクは小さく、べき等に
- ✅ 結果が不要なら`ignore_result=True`
- ✅ Late ACKで信頼性向上
- ✅ JSONシリアライザを使用
- ✅ ログとメトリクスで監視

## 次のステップ

運用編はこれで完了です！次は「[プロジェクト編](../05_projects/21_email_system.md)」で実践的なプロジェクトに取り組みます。
