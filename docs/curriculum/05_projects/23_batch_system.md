# プロジェクト3: データ処理バッチシステム

## プロジェクト概要

CSVファイルのインポート、データ変換、エラーレポートを含むバッチ処理システムを構築します。

## 実装

### タスク定義

```python
# tasks/batch_tasks.py
from celery import group, chord
from celery_app import app
import csv

@app.task(bind=True)
def import_csv(self, file_path):
    """CSVファイルをインポート"""
    with open(file_path, 'r') as f:
        reader = csv.DictReader(f)
        rows = list(reader)

    total = len(rows)
    batch_size = 100

    # バッチに分割
    batches = [
        rows[i:i + batch_size]
        for i in range(0, total, batch_size)
    ]

    # 各バッチを並列処理
    job = chord(
        process_batch.s(batch, i)
        for i, batch in enumerate(batches)
    )(generate_import_report.s())

    return job

@app.task
def process_batch(batch, batch_number):
    """バッチを処理"""
    results = {'success': [], 'errors': []}

    for row in batch:
        try:
            # データ変換
            cleaned_data = clean_data(row)
            validate_data(cleaned_data)

            # データベースに保存
            obj = MyModel.objects.create(**cleaned_data)
            results['success'].append(obj.id)

        except ValidationError as e:
            results['errors'].append({
                'row': row,
                'error': str(e)
            })

    return results

@app.task
def generate_import_report(batch_results):
    """インポートレポートを生成"""
    total_success = sum(len(r['success']) for r in batch_results)
    total_errors = sum(len(r['errors']) for r in batch_results)

    # エラー詳細を集約
    all_errors = []
    for r in batch_results:
        all_errors.extend(r['errors'])

    report = {
        'total_processed': total_success + total_errors,
        'success': total_success,
        'errors': total_errors,
        'error_details': all_errors
    }

    # レポートを保存
    save_report(report)
    notify_admin(report)

    return report
```

## まとめ

- ✅ バッチ処理の実装
- ✅ エラーハンドリング
- ✅ レポート生成

## 次のプロジェクト

次のプロジェクト「[Webスクレイピングシステム](./24_scraping_system.md)」では、分散クローリングを学びます。
