# 15. セキュリティ

## 学習目標

- メッセージの署名と検証
- 安全なシリアライゼーション
- セキュリティのベストプラクティス

## シリアライゼーションのセキュリティ

### Pickleを避ける

```python
# 危険: Pickleは任意コード実行のリスク
app.conf.task_serializer = 'pickle'  # ❌ 避ける

# 安全: JSONを使用
app.conf.task_serializer = 'json'  # ✅ 推奨
app.conf.accept_content = ['json']
```

### 受け入れるコンテンツタイプを制限

```python
app.conf.accept_content = ['json']  # JSONのみ受け入れる
```

## メッセージ署名

```python
# タスクメッセージに署名
app.conf.task_serializer = 'json'
app.conf.task_protocol = 2  # 新しいプロトコルを使用
```

## 認証とSSL

### RedisでSSL

```python
app.conf.broker_url = 'rediss://localhost:6379/0'  # SSL接続
app.conf.broker_use_ssl = {
    'ssl_cert_reqs': ssl.CERT_REQUIRED,
    'ssl_ca_certs': '/path/to/ca.pem',
}
```

### RabbitMQでSSL

```python
app.conf.broker_url = 'amqps://user:pass@localhost:5671//'
app.conf.broker_use_ssl = {
    'keyfile': '/path/to/key.pem',
    'certfile': '/path/to/cert.pem',
    'ca_certs': '/path/to/ca.pem',
    'cert_reqs': ssl.CERT_REQUIRED,
}
```

## タスクの制限

### タスクのホワイトリスト

```python
# 許可するタスクのみインポート
app.conf.imports = ('myapp.tasks',)
```

### レート制限

```python
@app.task(rate_limit='10/m')  # 1分間に10回まで
def api_call(url):
    return requests.get(url).json()
```

## 環境変数で機密情報を管理

```python
import os

app.conf.broker_url = os.getenv('CELERY_BROKER_URL')
app.conf.result_backend = os.getenv('CELERY_RESULT_BACKEND')
```

## まとめ

- ✅ Pickleは使わない（JSONを使用）
- ✅ SSL/TLSで通信を暗号化
- ✅ レート制限で悪用を防ぐ
- ✅ 環境変数で機密情報を管理

## 次のステップ

応用編はこれで完了です！次は「[運用編](../04_operations/16_monitoring.md)」に進みます。
