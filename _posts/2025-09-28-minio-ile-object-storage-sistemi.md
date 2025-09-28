---
title: "MinIO ile Object Storage Sistemi"
date: 2025-09-28 09:00:00 +0300
categories: [Storage, Cloud]
tags: [minio, object-storage, s3, python, cloud-storage, backup]
image:
  path: /assets/img/posts/minio-architecture-diagram.png
  alt: "MinIO Architecture"
---

Modern uygulamalarda veri depolama ihtiyaçları giderek artıyor. Resimler, videolar, log dosyaları, yedekler ve büyük veri setleri gibi yapılandırılmamış verileri saklamak için object storage (nesne depolama) ideal bir çözümdür. MinIO, AWS S3 uyumlu, yüksek performanslı ve self-hosted bir object storage sistemidir.

## Object Storage Nedir?

Object storage, verileri nesneler (objects) olarak saklayan bir depolama mimarisidir. Her nesne:
- **Data**: Dosyanın kendisi
- **Metadata**: Dosya hakkında bilgiler
- **Unique ID**: Benzersiz tanımlayıcı

Traditional dosya sistemlerinden farklı olarak, object storage düz bir namespace kullanır ve sınırsız ölçeklenebilir.

### Object Storage vs Block Storage vs File Storage

```
File Storage (NFS, SMB):
└── Root
    ├── Folder1
    │   └── file.txt
    └── Folder2
        └── image.jpg

Block Storage (SAN, iSCSI):
[Block 1][Block 2][Block 3][Block 4]...

Object Storage (S3, MinIO):
bucket/prefix/object-key-12345
bucket/prefix/object-key-67890
```

## MinIO Nedir?

MinIO, Kubernetes-native, yüksek performanslı object storage sistemidir. AWS S3 API'si ile tam uyumlu olduğu için S3 kullanan uygulamalar MinIO'ya kolayca geçiş yapabilir.

### MinIO'nun Avantajları

- **S3 Uyumlu**: AWS SDK'ları ve araçları çalışır
- **Yüksek performans**: Okuma/yazma hızı
- **Self-hosted**: Kendi sunucunuzda çalışır, vendor lock-in yok
- **Erasure coding**: Veri güvenliği ve dayanıklılık
- **Kubernetes entegrasyonu**: Cloud-native mimari
- **Açık kaynak**: Apache License 2.0

![MinIO S3 Compatible Storage](/assets/img/posts/minio-s3-compatible-storage.svg)
_MinIO S3 uyumlu object storage_

## MinIO Kurulumu

### Docker ile Tek Sunucu Kurulumu

```bash
# MinIO dizinlerini oluştur
mkdir -p ~/minio/data
mkdir -p ~/minio/config

# MinIO container'ı çalıştır
docker run -d \
  --name minio \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9001:9001 \
  -e "MINIO_ROOT_USER=minioadmin" \
  -e "MINIO_ROOT_PASSWORD=minioadmin123" \
  -v ~/minio/data:/data \
  minio/minio server /data --console-address ":9001"

# Web Console: http://localhost:9001
# API Endpoint: http://localhost:9000
```

### Docker Compose ile Production Kurulum

```yaml
# docker-compose.yml - Production MinIO
version: '3.8'

services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"  # API
      - "9001:9001"  # Console
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-minioadmin123}
      MINIO_BROWSER_REDIRECT_URL: https://minio-console.example.com
      MINIO_SERVER_URL: https://minio.example.com
      # Prometheus metrics
      MINIO_PROMETHEUS_AUTH_TYPE: public
    volumes:
      - minio-data:/data
    networks:
      - storage
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # Nginx reverse proxy (isteğe bağlı)
  nginx:
    image: nginx:alpine
    container_name: minio-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - minio
    networks:
      - storage

volumes:
  minio-data:
    driver: local

networks:
  storage:
    driver: bridge
```

```bash
# .env dosyası
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=your-secure-password-here
```

### Distributed MinIO (Yüksek Erişilebilirlik)

```yaml
# docker-compose-distributed.yml - 4 node MinIO cluster
version: '3.8'

services:
  minio1:
    image: minio/minio:latest
    hostname: minio1
    volumes:
      - minio1-data:/data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    command: server http://minio{1...4}/data --console-address ":9001"
    networks:
      - minio-cluster

  minio2:
    image: minio/minio:latest
    hostname: minio2
    volumes:
      - minio2-data:/data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    command: server http://minio{1...4}/data --console-address ":9001"
    networks:
      - minio-cluster

  minio3:
    image: minio/minio:latest
    hostname: minio3
    volumes:
      - minio3-data:/data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    command: server http://minio{1...4}/data --console-address ":9001"
    networks:
      - minio-cluster

  minio4:
    image: minio/minio:latest
    hostname: minio4
    volumes:
      - minio4-data:/data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    command: server http://minio{1...4}/data --console-address ":9001"
    networks:
      - minio-cluster
    ports:
      - "9000:9000"
      - "9001:9001"

volumes:
  minio1-data:
  minio2-data:
  minio3-data:
  minio4-data:

networks:
  minio-cluster:
    driver: bridge
```

## Python ile MinIO Kullanımı

### Temel Kurulum

```bash
# MinIO Python SDK
pip install minio
```

### MinIO Client Sınıfı

```python
# minio_client.py - MinIO wrapper sınıfı
from minio import Minio
from minio.error import S3Error
from io import BytesIO
from typing import Optional, List, Dict
from datetime import timedelta
import os

class MinIOClient:
    """MinIO object storage client wrapper"""
    
    def __init__(
        self,
        endpoint: str = "localhost:9000",
        access_key: str = "minioadmin",
        secret_key: str = "minioadmin123",
        secure: bool = False
    ):
        """
        MinIO client başlat
        
        Args:
            endpoint: MinIO sunucu adresi
            access_key: Access key
            secret_key: Secret key
            secure: HTTPS kullan (True/False)
        """
        self.client = Minio(
            endpoint,
            access_key=access_key,
            secret_key=secret_key,
            secure=secure
        )
        self.endpoint = endpoint
    
    def create_bucket(self, bucket_name: str) -> bool:
        """Bucket oluştur"""
        try:
            if not self.client.bucket_exists(bucket_name):
                self.client.make_bucket(bucket_name)
                print(f"✅ Bucket oluşturuldu: {bucket_name}")
                return True
            else:
                print(f"ℹ️  Bucket zaten var: {bucket_name}")
                return False
        except S3Error as e:
            print(f"❌ Bucket oluşturma hatası: {e}")
            return False
    
    def list_buckets(self) -> List[str]:
        """Tüm bucket'ları listele"""
        try:
            buckets = self.client.list_buckets()
            return [bucket.name for bucket in buckets]
        except S3Error as e:
            print(f"❌ Bucket listeleme hatası: {e}")
            return []
    
    def upload_file(
        self, 
        bucket_name: str, 
        object_name: str, 
        file_path: str,
        content_type: Optional[str] = None
    ) -> bool:
        """Dosya yükle"""
        try:
            # Content type otomatik tespit
            if content_type is None:
                import mimetypes
                content_type, _ = mimetypes.guess_type(file_path)
                content_type = content_type or "application/octet-stream"
            
            # Dosya boyutu
            file_size = os.path.getsize(file_path)
            
            # Upload
            self.client.fput_object(
                bucket_name,
                object_name,
                file_path,
                content_type=content_type
            )
            
            print(f"✅ Dosya yüklendi: {object_name} ({file_size} bytes)")
            return True
        
        except S3Error as e:
            print(f"❌ Yükleme hatası: {e}")
            return False
    
    def upload_data(
        self,
        bucket_name: str,
        object_name: str,
        data: bytes,
        content_type: str = "application/octet-stream"
    ) -> bool:
        """Bytes data yükle"""
        try:
            data_stream = BytesIO(data)
            data_size = len(data)
            
            self.client.put_object(
                bucket_name,
                object_name,
                data_stream,
                length=data_size,
                content_type=content_type
            )
            
            print(f"✅ Data yüklendi: {object_name} ({data_size} bytes)")
            return True
        
        except S3Error as e:
            print(f"❌ Yükleme hatası: {e}")
            return False
    
    def download_file(
        self,
        bucket_name: str,
        object_name: str,
        file_path: str
    ) -> bool:
        """Dosya indir"""
        try:
            self.client.fget_object(bucket_name, object_name, file_path)
            print(f"✅ Dosya indirildi: {file_path}")
            return True
        
        except S3Error as e:
            print(f"❌ İndirme hatası: {e}")
            return False
    
    def get_object_data(self, bucket_name: str, object_name: str) -> Optional[bytes]:
        """Object verisini al"""
        try:
            response = self.client.get_object(bucket_name, object_name)
            data = response.read()
            response.close()
            response.release_conn()
            return data
        
        except S3Error as e:
            print(f"❌ Okuma hatası: {e}")
            return None
    
    def list_objects(
        self,
        bucket_name: str,
        prefix: Optional[str] = None,
        recursive: bool = True
    ) -> List[Dict]:
        """Bucket'taki objeleri listele"""
        try:
            objects = self.client.list_objects(
                bucket_name,
                prefix=prefix,
                recursive=recursive
            )
            
            result = []
            for obj in objects:
                result.append({
                    "name": obj.object_name,
                    "size": obj.size,
                    "last_modified": obj.last_modified,
                    "etag": obj.etag
                })
            
            return result
        
        except S3Error as e:
            print(f"❌ Listeleme hatası: {e}")
            return []
    
    def delete_object(self, bucket_name: str, object_name: str) -> bool:
        """Object sil"""
        try:
            self.client.remove_object(bucket_name, object_name)
            print(f"✅ Silindi: {object_name}")
            return True
        
        except S3Error as e:
            print(f"❌ Silme hatası: {e}")
            return False
    
    def delete_objects(self, bucket_name: str, object_names: List[str]) -> bool:
        """Birden fazla object sil"""
        try:
            errors = self.client.remove_objects(
                bucket_name,
                object_names
            )
            
            error_list = list(errors)
            if error_list:
                print(f"⚠️  Bazı dosyalar silinemedi: {len(error_list)}")
                return False
            
            print(f"✅ {len(object_names)} dosya silindi")
            return True
        
        except S3Error as e:
            print(f"❌ Silme hatası: {e}")
            return False
    
    def get_presigned_url(
        self,
        bucket_name: str,
        object_name: str,
        expires: timedelta = timedelta(hours=1)
    ) -> Optional[str]:
        """Geçici paylaşım linki oluştur"""
        try:
            url = self.client.presigned_get_object(
                bucket_name,
                object_name,
                expires=expires
            )
            return url
        
        except S3Error as e:
            print(f"❌ URL oluşturma hatası: {e}")
            return None
    
    def get_object_stat(self, bucket_name: str, object_name: str) -> Optional[Dict]:
        """Object metadata'sını al"""
        try:
            stat = self.client.stat_object(bucket_name, object_name)
            return {
                "size": stat.size,
                "last_modified": stat.last_modified,
                "content_type": stat.content_type,
                "etag": stat.etag,
                "metadata": stat.metadata
            }
        
        except S3Error as e:
            print(f"❌ Stat hatası: {e}")
            return None
    
    def copy_object(
        self,
        source_bucket: str,
        source_object: str,
        dest_bucket: str,
        dest_object: str
    ) -> bool:
        """Object kopyala"""
        try:
            from minio.commonconfig import CopySource
            
            self.client.copy_object(
                dest_bucket,
                dest_object,
                CopySource(source_bucket, source_object)
            )
            
            print(f"✅ Kopyalandı: {source_object} -> {dest_object}")
            return True
        
        except S3Error as e:
            print(f"❌ Kopyalama hatası: {e}")
            return False

# Kullanım
if __name__ == "__main__":
    # Client oluştur
    minio_client = MinIOClient(
        endpoint="localhost:9000",
        access_key="minioadmin",
        secret_key="minioadmin123",
        secure=False
    )
    
    # Bucket oluştur
    minio_client.create_bucket("my-bucket")
    
    # Dosya yükle
    minio_client.upload_file(
        "my-bucket",
        "test.txt",
        "/path/to/local/file.txt"
    )
    
    # Objeleri listele
    objects = minio_client.list_objects("my-bucket")
    for obj in objects:
        print(f"{obj['name']}: {obj['size']} bytes")
```

### FastAPI ile Dosya Upload Sistemi

```python
# fastapi_minio.py - FastAPI ile dosya yükleme
from fastapi import FastAPI, UploadFile, File, HTTPException, Query
from fastapi.responses import StreamingResponse
from minio_client import MinIOClient
from datetime import timedelta
from typing import List
import io

app = FastAPI(title="MinIO File Upload API")

# MinIO client
minio_client = MinIOClient(
    endpoint="localhost:9000",
    access_key="minioadmin",
    secret_key="minioadmin123"
)

# Varsayılan bucket
DEFAULT_BUCKET = "uploads"
minio_client.create_bucket(DEFAULT_BUCKET)

@app.post("/upload")
async def upload_file(
    file: UploadFile = File(...),
    bucket: str = Query(DEFAULT_BUCKET),
    prefix: str = Query("")
):
    """Dosya yükle"""
    try:
        # Dosya adı
        object_name = f"{prefix}/{file.filename}" if prefix else file.filename
        
        # Dosyayı oku
        contents = await file.read()
        
        # MinIO'ya yükle
        success = minio_client.upload_data(
            bucket,
            object_name,
            contents,
            content_type=file.content_type or "application/octet-stream"
        )
        
        if not success:
            raise HTTPException(status_code=500, detail="Upload failed")
        
        return {
            "success": True,
            "bucket": bucket,
            "object_name": object_name,
            "size": len(contents),
            "content_type": file.content_type
        }
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/upload/multiple")
async def upload_multiple_files(
    files: List[UploadFile] = File(...),
    bucket: str = Query(DEFAULT_BUCKET),
    prefix: str = Query("")
):
    """Birden fazla dosya yükle"""
    results = []
    
    for file in files:
        try:
            object_name = f"{prefix}/{file.filename}" if prefix else file.filename
            contents = await file.read()
            
            success = minio_client.upload_data(
                bucket,
                object_name,
                contents,
                content_type=file.content_type or "application/octet-stream"
            )
            
            results.append({
                "filename": file.filename,
                "success": success,
                "size": len(contents)
            })
        
        except Exception as e:
            results.append({
                "filename": file.filename,
                "success": False,
                "error": str(e)
            })
    
    return {"results": results}

@app.get("/download/{bucket}/{object_name:path}")
async def download_file(bucket: str, object_name: str):
    """Dosya indir"""
    try:
        # Dosya verisini al
        data = minio_client.get_object_data(bucket, object_name)
        
        if data is None:
            raise HTTPException(status_code=404, detail="File not found")
        
        # Metadata al
        stat = minio_client.get_object_stat(bucket, object_name)
        
        return StreamingResponse(
            io.BytesIO(data),
            media_type=stat.get("content_type", "application/octet-stream"),
            headers={
                "Content-Disposition": f"attachment; filename={object_name.split('/')[-1]}"
            }
        )
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/share/{bucket}/{object_name:path}")
async def get_share_link(
    bucket: str,
    object_name: str,
    expires_hours: int = Query(1, ge=1, le=168)
):
    """Paylaşım linki oluştur"""
    try:
        url = minio_client.get_presigned_url(
            bucket,
            object_name,
            expires=timedelta(hours=expires_hours)
        )
        
        if url is None:
            raise HTTPException(status_code=404, detail="File not found")
        
        return {
            "url": url,
            "expires_in_hours": expires_hours
        }
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/list/{bucket}")
async def list_files(
    bucket: str,
    prefix: str = Query(""),
    recursive: bool = Query(True)
):
    """Dosyaları listele"""
    try:
        objects = minio_client.list_objects(bucket, prefix, recursive)
        return {
            "bucket": bucket,
            "prefix": prefix,
            "count": len(objects),
            "objects": objects
        }
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/delete/{bucket}/{object_name:path}")
async def delete_file(bucket: str, object_name: str):
    """Dosya sil"""
    try:
        success = minio_client.delete_object(bucket, object_name)
        
        if not success:
            raise HTTPException(status_code=500, detail="Delete failed")
        
        return {"success": True, "message": f"Deleted: {object_name}"}
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/buckets")
async def list_buckets():
    """Bucket'ları listele"""
    try:
        buckets = minio_client.list_buckets()
        return {"buckets": buckets}
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/buckets/{bucket_name}")
async def create_bucket(bucket_name: str):
    """Bucket oluştur"""
    try:
        success = minio_client.create_bucket(bucket_name)
        return {"success": success, "bucket": bucket_name}
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

![MinIO Object Locking](/assets/img/posts/minio-object-locking.svg)
_MinIO object locking ve versioning_

## İleri Seviye Özellikler

### Object Versioning

```python
# versioning.py - Object versioning yönetimi
from minio import Minio
from minio.versioningconfig import VersioningConfig, ENABLED

def enable_versioning(client: Minio, bucket_name: str):
    """Bucket için versioning aktif et"""
    config = VersioningConfig(ENABLED)
    client.set_bucket_versioning(bucket_name, config)
    print(f"✅ Versioning aktif: {bucket_name}")

def list_object_versions(client: Minio, bucket_name: str, prefix: str = ""):
    """Object versiyonlarını listele"""
    versions = client.list_objects(
        bucket_name,
        prefix=prefix,
        include_version=True
    )
    
    for version in versions:
        print(f"Object: {version.object_name}")
        print(f"  Version ID: {version.version_id}")
        print(f"  Is Latest: {version.is_latest}")
        print(f"  Size: {version.size}")
        print(f"  Last Modified: {version.last_modified}")
        print()

def restore_object_version(
    client: Minio,
    bucket_name: str,
    object_name: str,
    version_id: str
):
    """Eski versiyonu geri yükle"""
    from minio.commonconfig import CopySource
    
    # Eski versiyonu kopyala (yeni versiyon olarak)
    source = CopySource(bucket_name, object_name, version_id=version_id)
    client.copy_object(bucket_name, object_name, source)
    print(f"✅ Versiyon geri yüklendi: {version_id}")
```

### Lifecycle Management

```python
# lifecycle.py - Otomatik yaşam döngüsü yönetimi
from minio import Minio
from minio.lifecycleconfig import (
    LifecycleConfig,
    Rule,
    Expiration,
    Transition,
    Filter
)
from datetime import timedelta

def set_lifecycle_rules(client: Minio, bucket_name: str):
    """Lifecycle kuralları tanımla"""
    
    # Kural 1: 30 günden eski temp dosyaları sil
    rule1 = Rule(
        rule_id="delete-temp-files",
        status="Enabled",
        rule_filter=Filter(prefix="temp/"),
        expiration=Expiration(days=30)
    )
    
    # Kural 2: 90 günden eski logları sil
    rule2 = Rule(
        rule_id="delete-old-logs",
        status="Enabled",
        rule_filter=Filter(prefix="logs/"),
        expiration=Expiration(days=90)
    )
    
    # Kural 3: Backup'ları 1 yıl sonra sil
    rule3 = Rule(
        rule_id="delete-old-backups",
        status="Enabled",
        rule_filter=Filter(prefix="backups/"),
        expiration=Expiration(days=365)
    )
    
    # Kuralları uygula
    config = LifecycleConfig([rule1, rule2, rule3])
    client.set_bucket_lifecycle(bucket_name, config)
    print(f"✅ Lifecycle kuralları uygulandı: {bucket_name}")

def get_lifecycle_rules(client: Minio, bucket_name: str):
    """Mevcut lifecycle kurallarını göster"""
    config = client.get_bucket_lifecycle(bucket_name)
    
    for rule in config.rules:
        print(f"Rule ID: {rule.rule_id}")
        print(f"  Status: {rule.status}")
        print(f"  Prefix: {rule.rule_filter.prefix if rule.rule_filter else 'all'}")
        if rule.expiration:
            print(f"  Expiration: {rule.expiration.days} days")
        print()
```

### Bucket Policy (Erişim Kontrolü)

```python
# policy.py - Bucket erişim politikaları
from minio import Minio
import json

def set_public_read_policy(client: Minio, bucket_name: str, prefix: str = ""):
    """Public okuma erişimi ver"""
    
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {"AWS": ["*"]},
                "Action": ["s3:GetObject"],
                "Resource": [f"arn:aws:s3:::{bucket_name}/{prefix}*"]
            }
        ]
    }
    
    client.set_bucket_policy(bucket_name, json.dumps(policy))
    print(f"✅ Public read policy uygulandı: {bucket_name}/{prefix}")

def set_user_access_policy(
    client: Minio,
    bucket_name: str,
    user_arn: str,
    actions: list
):
    """Belirli kullanıcıya erişim ver"""
    
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {"AWS": [user_arn]},
                "Action": actions,
                "Resource": [f"arn:aws:s3:::{bucket_name}/*"]
            }
        ]
    }
    
    client.set_bucket_policy(bucket_name, json.dumps(policy))
    print(f"✅ Kullanıcı erişimi tanımlandı")

def remove_bucket_policy(client: Minio, bucket_name: str):
    """Bucket policy'sini kaldır"""
    client.delete_bucket_policy(bucket_name)
    print(f"✅ Policy kaldırıldı: {bucket_name}")
```

## Backup ve Restore Sistemi

```python
# backup_system.py - Otomatik backup sistemi
from minio_client import MinIOClient
from pathlib import Path
from datetime import datetime
import tarfile
import tempfile
import schedule
import time

class BackupSystem:
    """MinIO destekli backup sistemi"""
    
    def __init__(self, minio_client: MinIOClient, bucket_name: str = "backups"):
        self.client = minio_client
        self.bucket_name = bucket_name
        self.client.create_bucket(bucket_name)
    
    def backup_directory(self, directory: str, backup_name: str = None):
        """Dizini tar.gz olarak yedekle"""
        if backup_name is None:
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            backup_name = f"backup_{timestamp}.tar.gz"
        
        # Geçici tar.gz dosyası oluştur
        with tempfile.NamedTemporaryFile(delete=False, suffix=".tar.gz") as tmp:
            tmp_path = tmp.name
        
        # Dizini sıkıştır
        print(f"📦 Sıkıştırılıyor: {directory}")
        with tarfile.open(tmp_path, "w:gz") as tar:
            tar.add(directory, arcname=Path(directory).name)
        
        # MinIO'ya yükle
        print(f"⬆️  Yükleniyor: {backup_name}")
        success = self.client.upload_file(
            self.bucket_name,
            backup_name,
            tmp_path,
            content_type="application/gzip"
        )
        
        # Geçici dosyayı sil
        Path(tmp_path).unlink()
        
        if success:
            print(f"✅ Backup tamamlandı: {backup_name}")
        
        return success
    
    def list_backups(self):
        """Mevcut backup'ları listele"""
        backups = self.client.list_objects(self.bucket_name)
        
        print("\n📁 Mevcut Backup'lar:")
        print("-" * 70)
        for backup in backups:
            size_mb = backup['size'] / (1024 * 1024)
            print(f"{backup['name']:<40} {size_mb:>8.2f} MB  {backup['last_modified']}")
        print("-" * 70)
        
        return backups
    
    def restore_backup(self, backup_name: str, restore_path: str):
        """Backup'ı geri yükle"""
        # Geçici dosyaya indir
        with tempfile.NamedTemporaryFile(delete=False, suffix=".tar.gz") as tmp:
            tmp_path = tmp.name
        
        print(f"⬇️  İndiriliyor: {backup_name}")
        success = self.client.download_file(
            self.bucket_name,
            backup_name,
            tmp_path
        )
        
        if not success:
            return False
        
        # Arşivi aç
        print(f"📂 Açılıyor: {restore_path}")
        with tarfile.open(tmp_path, "r:gz") as tar:
            tar.extractall(restore_path)
        
        # Geçici dosyayı sil
        Path(tmp_path).unlink()
        
        print(f"✅ Restore tamamlandı: {restore_path}")
        return True
    
    def delete_old_backups(self, keep_days: int = 30):
        """Eski backup'ları sil"""
        from datetime import timedelta
        
        cutoff_date = datetime.now() - timedelta(days=keep_days)
        backups = self.client.list_objects(self.bucket_name)
        
        deleted_count = 0
        for backup in backups:
            if backup['last_modified'] < cutoff_date:
                self.client.delete_object(self.bucket_name, backup['name'])
                deleted_count += 1
        
        print(f"✅ {deleted_count} eski backup silindi")
        return deleted_count
    
    def schedule_backup(self, directory: str, time_str: str = "02:00"):
        """Zamanlanmış backup"""
        def backup_job():
            print(f"\n⏰ Zamanlanmış backup başladı: {datetime.now()}")
            self.backup_directory(directory)
        
        schedule.every().day.at(time_str).do(backup_job)
        
        print(f"📅 Backup zamanlandı: Her gün {time_str}")
        print("⏳ Bekliyor... (Ctrl+C ile dur)")
        
        try:
            while True:
                schedule.run_pending()
                time.sleep(60)
        except KeyboardInterrupt:
            print("\n🛑 Backup servisi durduruldu")

# Kullanım
if __name__ == "__main__":
    minio_client = MinIOClient()
    backup_system = BackupSystem(minio_client)
    
    # Manuel backup
    backup_system.backup_directory("/path/to/important/data")
    
    # Backup'ları listele
    backup_system.list_backups()
    
    # Eski backup'ları temizle
    backup_system.delete_old_backups(keep_days=30)
    
    # Zamanlanmış backup (her gün saat 02:00)
    # backup_system.schedule_backup("/path/to/data", "02:00")
```

## MinIO Client (mc) CLI Kullanımı

```bash
# MinIO Client kurulumu
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Alias tanımla
mc alias set myminio http://localhost:9000 minioadmin minioadmin123

# Bucket oluştur
mc mb myminio/mybucket

# Dosya yükle
mc cp file.txt myminio/mybucket/

# Dizin senkronize et
mc mirror /local/path myminio/mybucket/ --watch

# Dosya listele
mc ls myminio/mybucket/

# Dosya indir
mc cp myminio/mybucket/file.txt /local/path/

# Bucket policy ayarla
mc policy set public myminio/mybucket

# İstatistikler
mc stat myminio/mybucket/file.txt

# Disk kullanımı
mc du myminio/mybucket
```

## Best Practices

### 1. Güvenlik

```python
# Güvenli bağlantı
minio_client = MinIOClient(
    endpoint="minio.example.com",
    access_key=os.getenv("MINIO_ACCESS_KEY"),
    secret_key=os.getenv("MINIO_SECRET_KEY"),
    secure=True  # HTTPS kullan
)

# Bucket encryption
from minio.sseconfig import Rule, SSEConfig
rule = Rule(default_sse=True)
config = SSEConfig(rule)
client.set_bucket_encryption(bucket_name, config)
```

### 2. Performans

```python
# Multipart upload (büyük dosyalar için)
client.fput_object(
    bucket_name,
    object_name,
    file_path,
    part_size=10*1024*1024  # 10MB part'lar
)

# Paralel upload
from concurrent.futures import ThreadPoolExecutor

def upload_files_parallel(files: List[str], bucket: str):
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = [
            executor.submit(minio_client.upload_file, bucket, f, f)
            for f in files
        ]
        results = [f.result() for f in futures]
    return results
```

### 3. Monitoring

```python
# Health check
def check_minio_health(client: Minio) -> bool:
    try:
        client.list_buckets()
        return True
    except:
        return False

# Disk usage monitoring
def get_bucket_size(client: MinIOClient, bucket: str) -> int:
    total_size = 0
    objects = client.list_objects(bucket)
    for obj in objects:
        total_size += obj['size']
    return total_size
```

## Sonuç

MinIO, modern uygulamalar için güçlü ve esnek bir object storage çözümüdür. Bu yazıda öğrendiklerinizle:

- MinIO kurulumu ve konfigürasyonu yapabilirsiniz
- Python ile dosya yükleme/indirme sistemleri geliştirebilirsiniz
- FastAPI ile dosya upload API'si oluşturabilirsiniz
- Versioning ve lifecycle management kullanabilirsiniz
- Otomatik backup sistemleri kurabilirsiniz
- Production-ready storage çözümleri deploy edebilirsiniz

MinIO, AWS S3 uyumluluğu sayesinde vendor lock-in olmadan bulut depolama çözümleri sunmanızı sağlar. Kubernetes-native yapısı ile modern mikroservis mimarilerine mükemmel uyum gösterir.

## Kaynaklar

- [MinIO Official Documentation](https://min.io/docs/)
- [MinIO Python SDK](https://min.io/docs/minio/linux/developers/python/minio-py.html)
- [MinIO Docker Hub](https://hub.docker.com/r/minio/minio)
- [AWS S3 API Compatibility](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
- [Object Storage Best Practices](https://min.io/docs/minio/linux/operations/concepts.html)
