# Celery 技術詳細解説

## 概要

このディレクトリには、Celeryの内部実装と技術的な仕組みを詳しく解説したドキュメントが含まれています。

テキストベースの図解を多用して、Celeryがどのように動作しているのかを深く理解できるようになっています。

## 目次

### 1. [メッセージブローカーの内部動作](./01_message_broker_internals.md)

Celeryの心臓部であるメッセージブローカーの仕組みを徹底解説。

**内容:**
- Redis と RabbitMQ のアーキテクチャ比較
- メッセージキューの内部データ構造
- AMQPプロトコルの詳細
- Exchange、Queue、Bindingの仕組み
- メッセージの永続化メカニズム
- 優先度キューの実装
- パフォーマンス特性の比較

**詳細図解:**
- Redisのデータ構造（LIST、ZSET）
- RabbitMQの内部コンポーネント
- メッセージフロー
- Visibility Timeout
- パフォーマンスベンチマーク

### 2. [タスク実行の詳細フロー](./02_task_execution_flow.md)

タスクがどのように実行されるのか、ブローカーからワーカー、結果保存までの完全なフローを解説。

**内容:**
- Phase 1: タスク送信（Producer）
- Phase 2: タスク取得（Consumer）
- Phase 3: タスク実行（Worker Pool）
- Phase 4: 結果保存とACK
- Phase 5: 結果取得（Client）
- タスク状態の遷移
- リトライメカニズムの詳細
- タイムアウトの実装

**詳細図解:**
- タスクメッセージの構造
- 実行フローの全体像
- 状態遷移図
- リトライフロー
- シグナルハンドリング

### 3. [ワーカーアーキテクチャ](./03_worker_architecture.md)

Celeryワーカーの内部構造とプロセスモデルを徹底解説。

**内容:**
- ワーカープロセスの階層構造
- Prefork Poolの実装詳細
- Gevent Poolの実装詳細
- プリフェッチとACKメカニズム
- QoS（Quality of Service）管理
- Early ACK vs Late ACK
- Visibility Timeout

**詳細図解:**
- プロセス階層
- Prefork Poolのメモリレイアウト
- Gevent Poolのイベントループ
- プリフェッチフロー
- ACKタイムライン

### 4. [マイクロサービスアーキテクチャ設計パターン](./04_microservices_architecture.md)

Celeryを使ったマイクロサービスアーキテクチャの実践的な設計パターンを徹底解説。

**内容:**
- Pattern 1: タスクベース・マイクロサービス
- Pattern 2: イベント駆動アーキテクチャ（Pub/Sub）
- Pattern 3: Sagaパターン（分散トランザクション）
- Pattern 4: CQRS + Event Sourcing
- Redis vs RabbitMQの選択ガイド
- パターン選択マトリクス
- 実装のベストプラクティス

**詳細図解:**
- 各パターンのアーキテクチャ図
- サービス間通信フロー
- イベントフロー
- Saga実行シーケンス
- CQRS/Event Sourcingのデータフロー
- ブローカー選択の決定木

## 学習の進め方

### 推奨順序

1. **メッセージブローカーの内部動作** - Celeryの基盤を理解
2. **タスク実行の詳細フロー** - タスクのライフサイクルを追跡
3. **ワーカーアーキテクチャ** - ワーカーの内部を深掘り
4. **マイクロサービスアーキテクチャ設計パターン** - 実践的なアーキテクチャ設計を学ぶ

### 学習のヒント

- 各ドキュメントの図解を丁寧に読む
- 実際のコードと対応させながら理解する
- 実験: ログを有効にして実際の動作を確認
- デバッガを使って内部の動きを追跡

## 前提知識

これらのドキュメントを読む前に、以下の知識があると理解が深まります：

### 必須
- Pythonの基本（マルチプロセス、マルチスレッド）
- Celeryの基本的な使い方（[学習カリキュラム](../curriculum/00_index.md)参照）

### 推奨
- ネットワークプログラミングの基礎
- プロセス間通信（IPC）
- UNIXシグナル
- イベント駆動プログラミング

### あると良い
- Redis/RabbitMQの基本的な使い方
- AMQPプロトコルの知識
- epoll/select等のI/O多重化

## 用語集

### コアコンセプト

- **Broker**: メッセージキュー（Redis/RabbitMQ）
- **Backend**: 結果ストレージ
- **Worker**: タスクを実行するプロセス
- **Pool**: ワーカープロセスの集合
- **Prefetch**: 先読みするメッセージ数
- **ACK**: メッセージ確認応答

### プロセスモデル

- **Prefork**: マルチプロセスモデル（デフォルト）
- **Gevent**: グリーンスレッドモデル
- **Eventlet**: Geventと類似
- **Threads**: マルチスレッドモデル

### メッセージング

- **AMQP**: Advanced Message Queuing Protocol
- **Exchange**: メッセージのルーティングポイント
- **Queue**: メッセージの保管場所
- **Binding**: ExchangeとQueueの関連付け
- **Routing Key**: ルーティングの鍵

### タイミング

- **ETA**: 実行予定時刻
- **Countdown**: 遅延秒数
- **Expires**: 有効期限
- **Visibility Timeout**: 可視性タイムアウト

## 実験環境の構築

技術詳細を実際に確認するための環境構築:

```bash
# Redisを起動（デバッグログ有効）
redis-server --loglevel verbose

# Celeryワーカーを起動（デバッグモード）
celery -A celery_app worker --loglevel=debug

# イベントを監視
celery -A celery_app events

# Flowerでリアルタイム監視
celery -A celery_app flower
```

## デバッグテクニック

### ログを詳細に

```python
# celery_app.py
import logging
logging.basicConfig(level=logging.DEBUG)

app.conf.worker_log_format = '''
[%(asctime)s: %(levelname)s/%(processName)s] %(message)s
'''
```

### 内部状態の確認

```python
# ワーカーの状態を確認
celery -A celery_app inspect stats
celery -A celery_app inspect active
celery -A celery_app inspect registered

# コネクションプールの状態
celery -A celery_app shell
>>> app.connection_for_read().pool
```

### トレースの有効化

```python
# タスク実行のトレース
app.conf.task_track_started = True

# イベントの送信
app.conf.worker_send_task_events = True
app.conf.task_send_sent_event = True
```

## 参考リソース

### 公式ドキュメント
- [Celery Internals](https://docs.celeryproject.org/en/stable/internals/guide.html)
- [Kombu Documentation](https://docs.celeryproject.org/projects/kombu/)
- [Billiard Documentation](https://github.com/celery/billiard)

### ソースコード
- [Celery GitHub](https://github.com/celery/celery)
- [Kombu GitHub](https://github.com/celery/kombu)
- [Py-AMQP GitHub](https://github.com/celery/py-amqp)

### プロトコル仕様
- [AMQP 0-9-1 Specification](https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf)
- [Redis Protocol](https://redis.io/topics/protocol)

## フィードバック

技術詳細についての質問や追加してほしい内容があれば、GitHubのIssueで教えてください。

---

**Happy Deep Diving! 🔬**
