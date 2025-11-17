# 18. テスト

## 学習目標

- タスクのユニットテストを書く
- インテグレーションテストを実装する
- モックとスタブを活用する

## ユニットテスト

### 同期実行でのテスト

```python
# conftest.py (pytest)
import pytest
from celery import Celery

@pytest.fixture(scope='session')
def celery_config():
    return {
        'broker_url': 'memory://',
        'result_backend': 'cache+memory://',
        'task_always_eager': True,  # 同期実行
        'task_eager_propagates': True,  # 例外を伝播
    }

# test_tasks.py
from tasks import add, send_email

def test_add():
    result = add.apply(args=(4, 6))
    assert result.get() == 10

def test_send_email(mocker):
    # メール送信をモック
    mock_send = mocker.patch('tasks.send_mail')

    send_email.apply(args=['user@example.com', 'Hello'])

    mock_send.assert_called_once()
```

### pytest-celeryの使用

```bash
pip install pytest-celery
```

```python
# test_tasks.py
import pytest

@pytest.mark.celery(result_backend='redis://')
def test_task_with_redis():
    from tasks import process_data
    result = process_data.delay({'key': 'value'})
    assert result.get() == expected_result
```

## モックの使用

### 外部APIのモック

```python
from unittest.mock import patch
from tasks import fetch_api_data

def test_fetch_api_data():
    with patch('tasks.requests.get') as mock_get:
        mock_get.return_value.json.return_value = {'data': 'test'}

        result = fetch_api_data.apply(args=['https://api.example.com'])

        assert result.get() == {'data': 'test'}
        mock_get.assert_called_once_with('https://api.example.com')
```

### データベースのモック

```python
from unittest.mock import MagicMock

def test_update_user(mocker):
    mock_user = MagicMock()
    mocker.patch('tasks.User.objects.get', return_value=mock_user)

    result = update_user.apply(args=[123, {'name': 'Alice'}])

    assert result.get()['status'] == 'success'
    mock_user.update.assert_called_once()
```

## インテグレーションテスト

### 実際のワーカーでのテスト

```python
def test_task_integration():
    """実際のワーカーでタスクを実行"""
    from tasks import long_task

    result = long_task.delay(10)

    # 完了を待つ
    value = result.get(timeout=30)

    assert value == expected_value
    assert result.successful()
```

## フィクスチャの活用

```python
# conftest.py
import pytest
from celery_app import app

@pytest.fixture
def celery_app():
    app.conf.update(
        broker_url='redis://localhost:6379/15',  # テスト用DB
        result_backend='redis://localhost:6379/15',
    )
    return app

@pytest.fixture
def celery_worker(celery_app):
    """テスト用ワーカーを起動"""
    with celery_app.Worker() as worker:
        yield worker
```

## カバレッジ

```bash
pip install pytest-cov

# カバレッジを測定
pytest --cov=tasks tests/
```

## まとめ

- ✅ `task_always_eager=True`で同期テスト
- ✅ モックで外部依存を分離
- ✅ pytest-celeryでテストを簡素化
- ✅ カバレッジで品質を維持

## 次のステップ

次のセクション「[デプロイメント](./19_deployment.md)」では、本番環境へのデプロイを学びます。
