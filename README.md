# Celery学習カリキュラム＆技術詳細解説

Celeryを基礎から実践的に学び、さらに内部実装まで深く理解するための包括的な学習教材です。

## 📚 コンテンツ

### 1. 学習カリキュラム（初心者〜実践者向け）

体系的にCeleryを学ぶための24セクション構成のカリキュラム。

**📖 [カリキュラムを見る](./docs/curriculum/00_index.md)**

## 📚 カリキュラム構成

### レベル1: 基礎編（4-6時間）
1. [Celeryの概要](./docs/curriculum/01_basics/01_what_is_celery.md)
2. [環境構築](./docs/curriculum/01_basics/02_environment_setup.md)
3. [最初のタスク](./docs/curriculum/01_basics/03_first_task.md)
4. [タスクの基本](./docs/curriculum/01_basics/04_task_basics.md)

### レベル2: 実践編（8-10時間）
5. [タスクの実行方法](./docs/curriculum/02_practical/05_task_execution.md)
6. [タスクの状態管理](./docs/curriculum/02_practical/06_task_states.md)
7. [ワーカーの理解](./docs/curriculum/02_practical/07_workers.md)
8. [メッセージブローカー](./docs/curriculum/02_practical/08_message_brokers.md)
9. [Celeryの設定](./docs/curriculum/02_practical/09_configuration.md)

### レベル3: 応用編（10-12時間）
10. [タスクのスケジューリング](./docs/curriculum/03_advanced/10_scheduling.md)
11. [タスクチェーンとワークフロー](./docs/curriculum/03_advanced/11_workflows.md)
12. [エラーハンドリング](./docs/curriculum/03_advanced/12_error_handling.md)
13. [タスクの優先度とルーティング](./docs/curriculum/03_advanced/13_routing.md)
14. [パフォーマンス最適化](./docs/curriculum/03_advanced/14_performance.md)
15. [セキュリティ](./docs/curriculum/03_advanced/15_security.md)

### レベル4: 運用編（6-8時間）
16. [モニタリング](./docs/curriculum/04_operations/16_monitoring.md)
17. [ロギングとデバッグ](./docs/curriculum/04_operations/17_logging.md)
18. [テスト](./docs/curriculum/04_operations/18_testing.md)
19. [デプロイメント](./docs/curriculum/04_operations/19_deployment.md)
20. [ベストプラクティス](./docs/curriculum/04_operations/20_best_practices.md)

### レベル5: 実践プロジェクト（8-10時間）
21. [プロジェクト1: メール送信システム](./docs/curriculum/05_projects/21_email_system.md)
22. [プロジェクト2: 画像処理パイプライン](./docs/curriculum/05_projects/22_image_pipeline.md)
23. [プロジェクト3: データ処理バッチシステム](./docs/curriculum/05_projects/23_batch_system.md)
24. [プロジェクト4: Webスクレイピングシステム](./docs/curriculum/05_projects/24_scraping_system.md)

**総学習時間**: 約36-46時間

### 2. 技術詳細解説（上級者向け）

Celeryの内部実装と技術的原理を図解付きで詳しく解説。

**🔬 [技術詳細を見る](./docs/technical-deep-dive/00_index.md)**

#### 主要トピック
1. **[メッセージブローカーの内部動作](./docs/technical-deep-dive/01_message_broker_internals.md)**
   - Redis/RabbitMQのアーキテクチャ
   - データ構造とプロトコル
   - パフォーマンス特性

2. **[タスク実行の詳細フロー](./docs/technical-deep-dive/02_task_execution_flow.md)**
   - 送信から結果取得までの完全フロー
   - 状態遷移とリトライ
   - タイムアウト実装

3. **[ワーカーアーキテクチャ](./docs/technical-deep-dive/03_worker_architecture.md)**
   - プロセス構造
   - Prefork/Gevent Pool
   - プリフェッチとACK

## 🎯 学習パス

### パス1: 初級者（Celery初心者）
```
学習カリキュラム
├─ レベル1: 基礎編（全て）
├─ レベル2: 実践編（前半）
└─ 簡単なプロジェクトで実践
```

### パス2: 中級者（Pythonの経験あり）
```
学習カリキュラム
├─ レベル1: 基礎編（流し読み）
├─ レベル2: 実践編（全て）
├─ レベル3: 応用編（必要な部分）
└─ プロジェクト1-2で実践
```

### パス3: 上級者（内部実装を理解したい）
```
学習カリキュラム
├─ レベル1-2: 復習
├─ レベル3-4: 重点的に学習
└─ 技術詳細解説で深掘り
```

## 🚀 スタートガイド

### 学習カリキュラムから始める

1. [カリキュラムインデックス](./docs/curriculum/00_index.md)を開く
2. 自分のレベルに合った学習パスを選択
3. 各セクションを順番に進める
4. 実際にコードを書いて実践する

### 技術詳細から始める（上級者向け）

1. [技術詳細インデックス](./docs/technical-deep-dive/00_index.md)を開く
2. 興味のあるトピックを選択
3. 図解を見ながら理解を深める
4. 実験環境で動作を確認する

## 📖 ドキュメント構成

```
celery-demo/
├── README.md                          # このファイル
└── docs/
    ├── curriculum/                    # 学習カリキュラム
    │   ├── 00_index.md               # カリキュラム全体インデックス
    │   ├── 01_basics/                # レベル1: 基礎編 (4ファイル)
    │   ├── 02_practical/             # レベル2: 実践編 (5ファイル)
    │   ├── 03_advanced/              # レベル3: 応用編 (6ファイル)
    │   ├── 04_operations/            # レベル4: 運用編 (5ファイル)
    │   └── 05_projects/              # レベル5: プロジェクト編 (4ファイル)
    └── technical-deep-dive/          # 技術詳細解説
        ├── 00_index.md               # 技術詳細インデックス
        ├── 01_message_broker_internals.md
        ├── 02_task_execution_flow.md
        └── 03_worker_architecture.md
```

## ✨ 特徴

### 学習カリキュラムの特徴
- ✅ 体系的な24セクション構成
- ✅ 実践的なコード例
- ✅ 4つの実プロジェクトで実践
- ✅ 段階的な学習パス

### 技術詳細解説の特徴
- ✅ テキストベースの詳細図解
- ✅ 内部実装の徹底解説
- ✅ パフォーマンスチューニングに役立つ
- ✅ トラブルシューティングに活用可能

## 🔧 前提知識

### 必須
- Python 3.7以上の基本文法
- 関数とデコレータの理解
- 基本的なコマンドライン操作

### 推奨
- 仮想環境の使い方
- 基本的なネットワークの知識
- Web開発の経験

### あると良い（技術詳細向け）
- マルチプロセス/マルチスレッドの理解
- Redis/RabbitMQの基礎知識
- プロセス間通信（IPC）の知識

## 📝 実験環境

カリキュラムや技術詳細を実際に試すための環境：

```bash
# Python仮想環境の作成
python3 -m venv venv
source venv/bin/activate

# Celeryのインストール
pip install "celery[redis]"

# Redisの起動（Docker）
docker run -d -p 6379:6379 redis:latest

# Celeryワーカーの起動
celery -A celery_app worker --loglevel=info

# Flowerでモニタリング
pip install flower
celery -A celery_app flower
```

## 🔗 リソース

- [Celery公式ドキュメント](https://docs.celeryproject.org/)
- [Celery GitHub](https://github.com/celery/celery)
- [Stack Overflow - Celeryタグ](https://stackoverflow.com/questions/tagged/celery)

## 📊 学習時間の目安

| レベル | 内容 | 時間 |
|--------|------|------|
| レベル1 | 基礎編 | 4-6時間 |
| レベル2 | 実践編 | 8-10時間 |
| レベル3 | 応用編 | 10-12時間 |
| レベル4 | 運用編 | 6-8時間 |
| レベル5 | プロジェクト編 | 8-10時間 |
| 技術詳細 | 内部実装の理解 | 6-8時間 |
| **合計** | | **42-54時間** |

## 🎓 学習後のスキル

このカリキュラムを完了すると、以下ができるようになります：

✅ Celeryを使った非同期タスク処理システムの設計・実装
✅ ワーカーとブローカーの適切な設定
✅ エラーハンドリングとリトライ戦略の実装
✅ 複雑なワークフローの構築
✅ 本番環境でのデプロイと運用
✅ パフォーマンスチューニング
✅ 内部実装の理解に基づいたトラブルシューティング

---

**Happy Learning! 🚀**
