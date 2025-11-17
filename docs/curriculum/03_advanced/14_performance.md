# 14. パフォーマンス最適化

## 学習目標

- タスクの圧縮とシリアライゼーション最適化
- プリフェッチ設定の調整
- ワーカーのチューニング

## シリアライゼーション最適化

### JSON (デフォルト)

```python
app.conf.task_serializer = 'json'
app.conf.result_serializer = 'json'
```

安全だが、やや遅い。

### MessagePack

```bash
pip install msgpack
```

```python
app.conf.task_serializer = 'msgpack'
app.conf.result_serializer = 'msgpack'
app.conf.accept_content = ['msgpack', 'json']
```

JSONより高速。

## 圧縮

```python
# タスクの圧縮
app.conf.task_compression = 'gzip'

# 結果の圧縮
app.conf.result_compression = 'gzip'
```

大きなペイロードで効果的。

## プリフェッチ設定

```python
# 一度に1タスクのみ取得（長時間タスク向け）
app.conf.worker_prefetch_multiplier = 1

# 複数タスクを先読み（短時間タスク向け）
app.conf.worker_prefetch_multiplier = 4  # デフォルト
```

## ワーカーの最適化

### 並行処理数

```bash
# CPU密集型
celery -A app worker --concurrency=4  # CPUコア数

# I/O密集型
celery -A app worker --pool=gevent --concurrency=100
```

### メモリ管理

```bash
celery -A app worker --max-tasks-per-child=1000
```

### タスクの最適化

```python
# 結果を保存しない
@app.task(ignore_result=True)
def log_event(data):
    save_log(data)

# 結果の有効期限を短く
app.conf.result_expires = 300  # 5分
```

## データベースクエリの最適化

```python
# 悪い例
@app.task
def process_users():
    for user in User.objects.all():  # N+1問題
        process_user(user)

# 良い例
@app.task
def process_users():
    users = User.objects.select_related('profile').all()
    for user in users:
        process_user(user)
```

## まとめ

- ✅ MessagePackで高速化
- ✅ 圧縮で帯域幅を削減
- ✅ プリフェッチでスループット向上
- ✅ メモリリークを防ぐ

## 次のステップ

次のセクション「[セキュリティ](./15_security.md)」では、セキュリティ対策を学びます。
