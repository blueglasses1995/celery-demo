# 17. ロギングとデバッグ

## 学習目標

- ログの設定と管理
- デバッグ手法の習得
- トラブルシューティング

## ロギング設定

### 基本的なログ設定

```python
# celery_app.py
from celery.signals import after_setup_logger
import logging

@after_setup_logger.connect
def setup_loggers(logger, *args, **kwargs):
    formatter = logging.Formatter(
        '[%(asctime)s: %(levelname)s/%(processName)s] %(message)s'
    )

    # ファイルハンドラ
    fh = logging.FileHandler('/var/log/celery/worker.log')
    fh.setFormatter(formatter)
    logger.addHandler(fh)

    # コンソールハンドラ
    ch = logging.StreamHandler()
    ch.setFormatter(formatter)
    logger.addHandler(ch)
```

### タスクごとのロギング

```python
import logging

logger = logging.getLogger(__name__)

@app.task(bind=True)
def logged_task(self):
    logger.info(f'Task started: {self.request.id}')
    try:
        result = perform_work()
        logger.info(f'Task completed: {result}')
        return result
    except Exception as exc:
        logger.error(f'Task failed: {exc}', exc_info=True)
        raise
```

## デバッグ手法

### デバッグモード

```bash
# デバッグレベルで起動
celery -A celery_app worker --loglevel=debug
```

### タスクのトレース

```python
app.conf.task_track_started = True  # タスク開始を記録

@app.task(bind=True)
def debug_task(self):
    print(f'Request: {self.request}')
    print(f'Task ID: {self.request.id}')
    print(f'Args: {self.request.args}')
    print(f'Kwargs: {self.request.kwargs}')
```

### 同期実行（テスト用）

```python
# settings.py
app.conf.task_always_eager = True  # タスクを同期実行
app.conf.task_eager_propagates = True  # 例外を伝播
```

## トラブルシューティング

### 一般的な問題と解決策

#### 1. タスクが実行されない

```bash
# ワーカーが起動しているか確認
celery -A celery_app inspect active

# タスクが登録されているか確認
celery -A celery_app inspect registered

# キュー名が正しいか確認
celery -A celery_app inspect active_queues
```

#### 2. メモリリーク

```bash
# max-tasks-per-childを設定
celery -A celery_app worker --max-tasks-per-child=100
```

#### 3. 接続エラー

```python
# リトライ設定を追加
app.conf.broker_connection_retry = True
app.conf.broker_connection_max_retries = 10
```

## まとめ

- ✅ 構造化ログで問題を追跡
- ✅ デバッグモードで詳細情報を取得
- ✅ `task_always_eager`でテスト
- ✅ `inspect`コマンドで状態確認

## 次のステップ

次のセクション「[テスト](./18_testing.md)」では、タスクのテスト方法を学びます。
