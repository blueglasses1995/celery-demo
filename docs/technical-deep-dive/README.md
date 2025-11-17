# Celery技術詳細解説

Celeryの内部実装と技術的原理を、テキストベースの図解を使って詳しく解説したドキュメント集です。

## 📚 コンテンツ

### [インデックス](./00_index.md)
全ドキュメントの目次と学習ガイド

### 技術詳細ドキュメント

1. **[メッセージブローカーの内部動作](./01_message_broker_internals.md)**
   - Redis/RabbitMQのアーキテクチャ
   - メッセージキューの内部構造
   - AMQPプロトコル
   - パフォーマンス比較

2. **[タスク実行の詳細フロー](./02_task_execution_flow.md)**
   - タスク送信からレスポンスまでの完全フロー
   - 状態遷移の詳細
   - リトライメカニズム
   - タイムアウトの実装

3. **[ワーカーアーキテクチャ](./03_worker_architecture.md)**
   - プロセス階層構造
   - Prefork/Gevent Poolの実装
   - プリフェッチとACK
   - QoS管理

## 🎯 対象読者

- Celeryの内部実装に興味がある開発者
- パフォーマンスチューニングをしたい人
- トラブルシューティングに役立てたい人
- Celeryコントリビューターを目指す人

## 📖 使い方

1. [インデックスページ](./00_index.md)から始める
2. 興味のあるトピックを選んで読む
3. 図解を見ながら理解を深める
4. 実際の環境で確認・実験する

## 🔬 実験のすすめ

ドキュメントを読むだけでなく、実際に動作を確認することを強く推奨します：

```bash
# デバッグログを有効にして起動
celery -A celery_app worker --loglevel=debug

# 内部状態を確認
celery -A celery_app inspect stats

# Flowerで可視化
celery -A celery_app flower
```

## 🔗 関連リソース

- [学習カリキュラム](../curriculum/00_index.md) - 基礎から学ぶ
- [Celery公式ドキュメント](https://docs.celeryproject.org/)
- [ソースコード](https://github.com/celery/celery)
