---
title: "Zarif Kapanış (Graceful Shutdown): Fişi Çekmeyin"
description: "Container'larınız ölürken veri kaybediyor musunuz? SIGTERM sinyallerini yakalayıp, DB bağlantılarını ve açık işlemleri güvenle kapatma sanatı."
date: "2025-02-15 14:00:00 +0300"
categories: [Backend, DevOps, Python]
tags: [graceful-shutdown, signal-handling, kubernetes, python, docker]
image:
  path: /assets/img/posts/graceful-shutdown-flow.png
  alt: "Graceful Shutdown Workflow"
---


Bilgisayarınızı kapatırken fişi mi çekersiniz, yoksa "Bilgisayarı Kapat" menüsünü mü kullanırsınız?
Fişi çekerseniz (Hard Shutdown) açık dosyalar bozulur, kaydedilmemiş veriler uçar ve veritabanı transactionları yarıda kalır.
Yazılımda da durum birebir aynıdır.

Kubernetes bir pod'u ölçeklendirirken (scale-down) veya yeni bir versiyon deploy ederken (rollout), eski pod'ları "öldürür".
Eğer uygulamanız bu yaklaşan ölümü (Pending Doom) hissetmez ve hazırlık yapmazsa;
*   O an işlenmekte olan ödeme isteği başarısız olur.
*   Veritabanına yarım yazılmış (Corrupted) veri girer.
*   Kullanıcı "502 Bad Gateway" veya "Connection Reset" hatası görür.

Bu yazıda, Python uygulamalarımıza nasıl **"Ölmeden önce son sözünü söyleme"** yeteneği (Graceful Shutdown) kazandıracağımızı ve Kubernetes/Docker dünyasındaki tuzakları inceleyeceğiz.

## 1. Sinyalleri Anlamak: İşletim Dili

Linux kernel, süreçlerle (Process) sinyaller aracılığıyla konuşur. Bir DevOps/Backend mühendisi şu üçlüyü adını bildiği gibi bilmelidir:

1.  **SIGINT (Signal Interrupt - 2):** Terminalde `Ctrl+C` bastığınızda giden "nazik" durdurma isteğidir.
2.  **SIGTERM (Signal Terminate - 15):** Kubernetes'in veya Docker'ın "Seni birazdan kapatacağım, toparlan" dediği sinyaldir. Yakalanabilir (Catchable).
3.  **SIGKILL (Signal Kill - 9):** `kill -9`. "Sorgusuz sualsiz öl". Uygulamanın bunu yakalama, log yazma veya vedalaşma şansı yoktur. Fişin çekilmesidir.

Hedefimiz: **SIGTERM** geldiğinde yeni istek almayı durdurmak ("Dükkan Kapandı" tabelası asmak), içerideki müşterilerin (isteklerin) işini bitirmek ve sonra kepenkleri indirmektir.

## 2. Python ile Profesyonel Sinyal Yakalama

Basit scriptlerde `try-except KeyboardInterrupt` yeterlidir ama production servislerinde bir `SignalInterceptor` sınıfı yazmak en temizidir.

```python
import signal
import time
import sys
import logging

class GracefulKiller:
    kill_now = False
    
    def __init__(self):
        signal.signal(signal.SIGINT, self.exit_gracefully)
        signal.signal(signal.SIGTERM, self.exit_gracefully)
    
    def exit_gracefully(self, signum, frame):
        logging.info(f"Sinyal alındı: {signum}. Kapanış süreci başlıyor...")
        self.kill_now = True

killer = GracefulKiller()

# Ana Döngü (Main Loop)
while not killer.kill_now:
    logging.info("İş yapılıyor...")
    process_data() # Uzun süren işlem
    time.sleep(1)

logging.info("Döngü durdu. Kaynaklar temizleniyor...")
db.close()
sys.exit(0)
```
Bu yapı sayesinde, bir işlem (transaction) *ortasında* sinyal gelse bile, döngünün başa dönmesini bekler ve öyle çıkarız. Veri bütünlüğü korunur.

## 3. Web Sunucularında (FastAPI/Gunicorn) Durum

Kendi döngünüzü yazmazsınız, Gunicorn veya Uvicorn sunucusu sizin yerinize çalışır.
Burada önemli olan Framework'ün sağladığı "Lifespan" (Yaşam Döngüsü) olaylarıdır.

```python
# FastAPI Lifespan (Python 3.7+)
from contextlib import asynccontextmanager
from fastapi import FastAPI
import asyncio

@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- Startup ---
    print("DB Bağlanıyor...")
    yield
    # --- Shutdown (SIGTERM Geldiğinde) ---
    print("Yeni istek alımı durduruldu. Mevcutlar bekleniyor...")
    # Bekleyen işleri bitir (Drain)
    await close_db_connections()
    print("Bye bye!")

app = FastAPI(lifespan=lifespan)
```
Kubernetes SIGTERM gönderdiğinde, Uvicorn bunu yakalar, yeni bağlantıları reddeder ve `shutdown` bloğunu çalıştırır.

## 4. Kubernetes: En Büyük Tuzak (PreStop Hook)

Kubernetes bir pod'u silerken aynı anda iki şey yapar:
1.  Pod'a `SIGTERM` gönderir.
2.  Pod'u `Service` (Load Balancer) endpoint listesinden çıkarır.

**Sorun:** Bu iki işlem asenkron çalışır. Bazen pod SIGTERM alır ve kapanmaya başlar, ama Load Balancer bunu henüz bilmediği için o ölü pod'a trafik göndermeye devam eder. Sonuç: Kullanıcı hatası.

**Çözüm:** `preStop` hook kullanmak.

```yaml
lifecycle:
  preStop:
    exec:
      # Şaka değil, gerçekten işe yarar.
      # Neden? K8s'in network tablolarını (iptables) güncellemesi zaman alır.
      command: ["/bin/sleep", "5"]
```
Bu 5 saniyelik uyku, uygulamanızın trafiği kesilmeden *önce* Load Balancer'dan düşmesini garantiler. Sonra SIGTERM gelir ve güvenle kapanırsınız.

## 5. Docker ve "Zombie Process" (PID 1 Sorunu)

Docker kullanırken `CMD ["sh", "-c", "python app.py"]` derseniz, PID 1 (Ana Süreç) `sh` olur. Python onun alt süreci (child) olur.
Linux'ta sinyaller sadece PID 1'e gider. `sh` kabuğu gelen SIGTERM'i çocuğuna (Python) iletmez.
Sonuç:
1.  Kubernetes "Kapan" der (SIGTERM).
2.  `sh` bunu yutar. Python çalışmaya devam eder.
3.  Kubernetes 30 saniye bekler.
4.  Kapanmayınca `SIGKILL` atar. (Fişi çeker). Veri kaybı!

**Çözüm:**
1.  **Exec Form Kullanın:** `CMD ["python", "app.py"]`. (Python PID 1 olur).
2.  **Tini Kullanın:** Dockerfile'da `ENTRYPOINT ["/usr/bin/tini", "--"] CMD [...]`. Tini, sinyalleri doğru şekilde ileten mini bir init sistemidir.

![Kubernetes Lifecycle](/assets/img/posts/pod-graceful-shutdown-diagram.png)
*Pod Lifecycle: Pending -> Running -> Terminating (PreStop -> SIGTERM) -> Killed.*

## 6. Workerlar (Celery/ARQ) Ne Olacak?

Web sunucusu kapanır, peki ya arkada 10 dakikadır video işleyen Celery worker?
Eğer SIGTERM gelince worker hemen kapanırsa, video bozulur.

Celery varsayılan olarak `Warm Shutdown` yapar. Yani elindeki işi bitirmeyi bekler.
Ancak Kubernetes `terminationGracePeriodSeconds: 30` (varsayılan) ayarı varsa ve işiniz 5 dakika sürüyorsa, K8s 30. saniyede kafasına sıkar.

**Çözüm:** Worker podları için `terminationGracePeriodSeconds` değerini işinizin maksimum süresine (örn: 3600s) yükseltin.

```yaml
spec:
  terminationGracePeriodSeconds: 3600 # 1 Saat bekle
```


## 7. Bonus: Canlı Debugging için SIGUSR1

Bazen uygulama çalışırken kilitlenir (Deadlock) ama CPU kullanmaz. Nerede takıldığını bulmak için uygulamayı kapatmak istemezsiniz.
Go dilinde `SIGQUIT` ile stack trace alabilirsiniz. Python'da bunu biz ekleyebiliriz:

```python
import traceback

def dump_stack_trace(signum, frame):
    print(f"Sinyal {signum} alındı. Stack trace basılıyor:")
    code = []
    for threadId, stack in sys._current_frames().items():
        code.append(f"\n# ThreadID: {threadId}")
        for filename, lineno, name, line in traceback.extract_stack(stack):
            code.append(f"File: \"{filename}\", line {lineno}, in {name}")
            if line:
                code.append(f"  {line.strip()}")
    print("\n".join(code))

signal.signal(signal.SIGUSR1, dump_stack_trace)
```
Artık çalışan pod'a `kill -s SIGUSR1 <pid>` attığınızda, loglarda uygulamanın o an tam olarak hangi satırda olduğunu görebilirsiniz. Hayat kurtarır.

## Sonuç: Nezaket Hayat Kurtarır


Graceful Shutdown, opsiyonel bir özellik değil, **Distributed Systems 101** konusudur.
Deploy sırasında "acaba bir isteği düşürdük mü?" stresi yaşamak istemiyorsanız:
1.  Sinyalleri dinleyin (SIGTERM).
2.  Kubernetes'te `preStop` hook (sleep) kullanın.
3.  Docker'da PID 1 sorununa (Tini) dikkat edin.
4.  Workerlar için Grace Period süresini uzatın.

İyi bir yazılım, partiden ayrılırken ortalığı dağıtıp kaçmaz; vedalaşır, kapıyı yavaşça çeker ve öyle gider.

