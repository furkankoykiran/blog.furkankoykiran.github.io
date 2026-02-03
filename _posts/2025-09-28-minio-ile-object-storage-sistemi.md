---
title: "MinIO ve Object Storage: Disksiz Mimariye Geçiş"
description: "Dosyaları sunucu diskinde saklamanın riskleri, MinIO ile S3 uyumlu mimari, Erasure Coding ve Docker ile production kurulumu."
date: "2025-09-28 11:00:00 +0300"
categories: [DevOps, Storage, Cloud Native, Architecture]
tags: [minio, s3, python, docker, kubernetes, distributed-systems]
image:
  path: /assets/img/posts/minio-architecture-diagram.png
  alt: "MinIO High Performance Object Storage Architecture"
---

Geleneksel web geliştirmede "dosya yükleme" (File Upload) işlemi genelde şöyle başlar: Formdan gelen dosyayı al, `/var/www/uploads/user_123.jpg` yoluna kaydet.

Bu yöntem, tek bir sunucuda çalışan hobi projeleri için harikadır. Ancak projeniz büyüyüp de yük dengeleyici (Load Balancer) arkasına **ikinci sunucuyu** eklediğiniz gün, o dosya sistemi başınıza bela olur.

Kullanıcı A fotoğrafını Sunucu-1'e yükler. Sayfayı yenilediğinde Load Balancer onu Sunucu-2'ye gönderir.
Sonuç: **404 Not Found.**

Çözüm? Uygulamanızı **Stateless** (Durumsuz) hale getirmek ve dosyaları **Object Storage** mimarisine taşımaktır. İşte bu noktada AWS S3'ün açık kaynak alternatifi **MinIO** devreye girer.

![MinIO Architecture](/assets/img/posts/minio-architecture-diagram.png)
*Object Storage mimarisi: Uygulama sunucuları dosyaları tutmaz, sadece yönetir.*

## 1. Object Storage Nedir ve Neden Lazım?

Dosya sistemi (File System), hiyerarşiktir (Klasör -> Dosya). Veritabanı gibidir ama metadata tutamaz. Object Storage ise düzdür (Flat). Her dosya bir "Obje"dir ve yanına bir kimlik kartı (Metadata) iliştirilir.

**AWS S3 varken neden MinIO?**
1.  **Maliyet:** AWS S3 faturası bandwidth ile şişer. MinIO'yu kendi sunucunuza (On-Premise) kurmak bedavadır.
2.  **Gizlilik (GDPR/KVKK):** Finansal verileri veya sağlık verilerini yurtdışına çıkaramazsınız. MinIO ile veriniz kendi veri merkezinizde kalır.
3.  **Hız:** Aynı veri merkezindeki sunucular arası 10Gbps hız varken, internet üzerinden S3'e gitmek yavaştır.

![MinIO vs S3](/assets/img/posts/minio-s3-compatible-storage.svg)

## 2. Docker ile Production-Ready Kurulum

MinIO'yu sadece "çalıştırmak" kolaydır ama production için "kalıcı" (persistent) hale getirmek gerekir.

```yaml
# docker-compose.yml
version: '3.8'
services:
  minio:
    image: minio/minio:RELEASE.2024-01-31T20-20-33Z
    container_name: minio_server
    command: server /data --console-address ":9001"
    restart: always # Sunucu kapanırsa otomatik başlat
    environment:
      MINIO_ROOT_USER: ${MINIO_USER}    # .env dosyasından oku!
      MINIO_ROOT_PASSWORD: ${MINIO_PASS}
    ports:
      - "9000:9000" # API Portu
      - "9001:9001" # Web Arayüzü (Console)
    volumes:
      # Verileri host makinede sakla, container silinse de veri gitmez
      - ./minio_data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
```

Tarayıcıdan `localhost:9001` adresine girdiğinizde sizi çok şık bir arayüz karşılar. Buradan "Bucket" (Kova) oluşturabilirsiniz.

## 3. Python (Boto3) Entegrasyonu

MinIO'nun en güzel yanı, **%100 S3 API Uyumlu** olmasıdır. Yani MinIO için özel bir kütüphane öğrenmenize gerek yoktur. AWS için yazılmış kodlar (boto3), sadece bir satırlık değişiklikle MinIO ile çalışır.

```python
import boto3
from botocore.client import Config

# S3 Client oluştururken 'endpoint_url' veriyoruz.
# Boto3 bunu görünce AWS yerine bizim sunucuya gidiyor.
s3 = boto3.client('s3',
                  endpoint_url='http://localhost:9000',
                  aws_access_key_id='admin',
                  aws_secret_access_key='password123',
                  config=Config(signature_version='s3v4'),
                  region_name='us-east-1')

def upload_securely(file_path, bucket):
    # Dosyayı yükle
    s3.upload_file(file_path, bucket, file_path)
    
    # Presigned URL (İmzalı Link) oluştur
    # Bu link 1 saat geçerli olacak, sonra patlayacak.
    url = s3.generate_presigned_url(
        ClientMethod='get_object',
        Params={'Bucket': bucket, 'Key': file_path},
        ExpiresIn=3600
    )
    return url

# Kullanıcıya dosyayı değil, linki veriyoruz!
# Trafik bizim uygulama sunucumuzdan değil, MinIO'dan akıyor.
print(upload_securely("fatura.pdf", "finance-docs"))
```

## 4. Erasure Coding: RAID'in Emekliliği

Eskiden disk bozulmalarına karşı RAID yapardık. MinIO'da ise **Erasure Coding** vardır.

MinIO, bir dosyayı (Object) matematiksel olarak parçalara böler (Data + Parity). 4 diskiniz varsa ve 2'si tamamen yansa bile (evet, fiziksel olarak yansa bile), MinIO o dosyayı matematiksel hesaplamalarla (Reed-Solomon algoritması) geri oluşturur.

Bu özellik, donanım arızalarında sizi kurtarır. `Bit Rot` koruması sayesinde, diskin zamanla manyetik özelliğini kaybetmesi sonucu oluşan veri bozulmalarını (silent corruption) bile onarır.

## 5. İleri Seviye: Object Locking (WORM)

Bazı sektörlerde (Finans, Sağlık, Hukuk) yasal zorunluluklar vardır: "Bu fatura 5 yıl boyunca SİLİNEMEZ ve DEĞİŞTİRİLEMEZ olmalıdır."

MinIO'da **Object Locking** açarsanız, `root` kullanıcısı bile o dosyayı süresi dolmadan silemez. Buna **WORM (Write Once, Read Many)** denir.

![MinIO Object Locking](/assets/img/posts/minio-object-locking.svg)
*Yasal uyumluluk için kilitleme mekanizması.*

## 6. Sıkça Sorulan Sorular & Hatalar

### "MinIO'yu Veritabanı Gibi Kullanabilir miyim?"
Hayır. MinIO bir blob storage'dır. İçinde arama yapamazsınız (SQL `WHERE name LIKE '%...%'` çalışmaz). Sadece dosyanın tamamını okuyabilirsiniz. Metadata araması yapabilirsiniz ama sınırlıdır.

### "Performans Sorunu Yaşar mıyım?"
MinIO, Go ile yazılmıştır ve Assembly optimizasyonları kullanır. NVMe diskler üzerinde saniyede 183 GB okuma hızlarına ulaşabilir. Darboğaz genelde ağ (Network) kartınızdadır.

### "Donanım Tavsiyesi Nedir?"
MinIO CPU dostudur, asıl yük Disk I/O üzerindedir.
*   **Disk:** Mümkünse NVMe SSD kullanın. Eğer HDD kullanacaksanız, çok sayıda diski paralel bağlayın.
*   **Network:** 10Gbps veya 25Gbps kartlar önerilir. 1Gbps ağ kartı, NVMe disklerin hızına yetişemez ve darboğaz oluşturur.
*   **RAM:** MinIO caching için RAM kullanır. Ne kadar çok RAM, o kadar çok performans (Cache Hit).

### MinIO Client (mc) İpuçları
GUI güzeldir ama terminal (CLI) candır. Production'da hayat kurtaran komutlar:
*   `mc admin info myminio`: Sunucu sağlık durumu ve uptime bilgisi.
*   `mc du myminio/photos`: Hangi klasör kaç GB yer kaplıyor?
*   `mc diff local/dir myminio/bucket`: Yerel klasör ile sunucu arasındaki farkları gösterir.
*   `mc watch myminio/bucket`: Kovadaki değişiklikleri canlı izler (Log tail gibi).

## 7. Terimler Sözlüğü (Glossary)

*   **Bucket:** Klasör benzeri mantıksal yapı. Dosyalar bucket içine atılır.
*   **Tenant:** Çok kiracılı mimaride her müşteriye ayrılan izole ortam.
*   **Presigned URL:** Geçici süreliğine yetki verilmiş erişim linki.
*   **Multipart Upload:** Büyük dosyaları (örn: 5GB) küçük parçalara bölerek paralel yükleme tekniği.

## 6. Veri Yaşam Döngüsü (Lifecycle Policies)

Disk doluluğu sorununu kökten çözmek için politikalar belirleyebilirsiniz.
"Log dosyalarını 30 gün tut, sonra sil" veya "Eski faturaları 1 yıl sonra ucuz diske (Tiering) taşı".

MinIO Client (`mc`) ile bunu yapmak bir satırdır:

```bash
# 'logs' kovasındaki 30 günden eski dosyaları otomatik sil
mc ilm add myminio/logs --expiry-days 30
```

Bu sayede "Disk doldu, logları kim silecek?" derdi biter.

## 7. Güvenlik ve Şifreleme (SSE)

MinIO, verileri diske yazmadan önce şifreleyebilir (Encryption at Rest).
3 farklı standart vardır:
1.  **SSE-S3:** Anahtarları MinIO yönetir. (En kolayı)
2.  **SSE-KMS:** Anahtarları harici bir kasa (Vault/KMS) yönetir. (En güvenlisi)
3.  **SSE-C:** Anahtarı istemci gönderir.

Production ortamında SSE-S3 açmak, diskleriniz çalınsa bile verilerin okunamamasını garanti eder.

## 8. Sıkça Sorulan Sorular & Hatalar Söz

Sunucunuzun diskine bağımlı kalmak, bir DevOps mühendisi için en büyük günahtır. Uygulamanızı özgürleştirmek için bugün MinIO'yu deneyin. Hem AWS faturasından kurtulun, hem de verinize gerçekten sahip olun.
