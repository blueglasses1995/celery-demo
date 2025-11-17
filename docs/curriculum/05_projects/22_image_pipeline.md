# プロジェクト2: 画像処理パイプライン

## プロジェクト概要

画像アップロード後、複数サイズのサムネイルを生成し、最適化してストレージにアップロードするパイプラインを構築します。

## 学習目標

- 画像処理のワークフロー
- chainを使ったパイプライン
- 進捗トラッキング
- エラーリカバリ

## 実装

### タスク定義

```python
# tasks/image_tasks.py
from celery import chain, group
from celery_app import app
from PIL import Image
import io

@app.task
def download_image(image_id):
    """画像をダウンロード"""
    image = ImageModel.objects.get(id=image_id)
    image_data = download_from_url(image.source_url)
    return {'image_id': image_id, 'data': image_data}

@app.task(bind=True)
def generate_thumbnails(self, image_info):
    """複数サイズのサムネイルを生成"""
    sizes = [(150, 150), (300, 300), (600, 600), (1200, 1200)]
    image_data = image_info['data']
    image_id = image_info['image_id']

    thumbnails = []
    for i, size in enumerate(sizes):
        # 進捗更新
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i + 1,
                'total': len(sizes),
                'size': size
            }
        )

        # サムネイル生成
        thumbnail = create_thumbnail(image_data, size)
        thumbnails.append({
            'size': size,
            'data': thumbnail
        })

    return {
        'image_id': image_id,
        'thumbnails': thumbnails
    }

@app.task
def optimize_images(thumbnail_info):
    """画像を最適化"""
    optimized = []
    for thumb in thumbnail_info['thumbnails']:
        # 画質を保ちつつファイルサイズを削減
        optimized_data = optimize_image(
            thumb['data'],
            quality=85,
            format='WEBP'
        )
        optimized.append({
            'size': thumb['size'],
            'data': optimized_data
        })

    return {
        'image_id': thumbnail_info['image_id'],
        'thumbnails': optimized
    }

@app.task
def upload_to_storage(image_info):
    """ストレージにアップロード"""
    image_id = image_info['image_id']
    urls = []

    for thumb in image_info['thumbnails']:
        # S3にアップロード
        url = upload_to_s3(
            data=thumb['data'],
            key=f'images/{image_id}/{thumb["size"][0]}x{thumb["size"][1]}.webp'
        )
        urls.append({
            'size': thumb['size'],
            'url': url
        })

    # データベースを更新
    ImageModel.objects.filter(id=image_id).update(
        thumbnails=urls,
        status='processed'
    )

    return {
        'image_id': image_id,
        'thumbnails': urls,
        'status': 'completed'
    }

@app.task
def process_image_pipeline(image_id):
    """画像処理パイプライン"""
    workflow = chain(
        download_image.s(image_id),
        generate_thumbnails.s(),
        optimize_images.s(),
        upload_to_storage.s()
    )
    return workflow.apply_async()
```

### 使用例

```python
# views.py
def upload_image_view(request):
    if request.method == 'POST':
        image_file = request.FILES['image']

        # 画像モデルを作成
        image = ImageModel.objects.create(
            original_file=image_file,
            status='uploading'
        )

        # パイプラインを開始
        result = process_image_pipeline.delay(image.id)

        return JsonResponse({
            'image_id': image.id,
            'task_id': result.id,
            'status': 'processing'
        })

def image_status(request, task_id):
    """処理状態を確認"""
    from celery.result import AsyncResult

    result = AsyncResult(task_id)

    if result.state == 'PROGRESS':
        return JsonResponse({
            'state': result.state,
            'current': result.info.get('current', 0),
            'total': result.info.get('total', 0),
            'size': result.info.get('size'),
        })
    elif result.state == 'SUCCESS':
        return JsonResponse({
            'state': result.state,
            'result': result.info
        })
    else:
        return JsonResponse({
            'state': result.state,
            'status': str(result.info)
        })
```

## まとめ

- ✅ chainでパイプライン構築
- ✅ 進捗トラッキング
- ✅ 画像処理のベストプラクティス

## 次のプロジェクト

次のプロジェクト「[データ処理バッチシステム](./23_batch_system.md)」では、大量データの処理を学びます。
