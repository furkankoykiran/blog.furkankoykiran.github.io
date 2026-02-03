---
title: "Systemd ile Python Servislerinizi Ölümsüzleştirin"
description: "Scriptlerinizi 'python main.py' diye çalıştırmaktan vazgeçin. Systemd ile otomatik başlatma, log yönetimi ve crash recovery."
date: "2025-01-12 10:00:00 +0300"
categories: [DevOps, Linux, Python]
tags: [systemd, python, linux, service-management, daemon]
image:
  path: /assets/img/posts/systemd-components-architecture.png
  alt: "Systemd Architecture and Unit Files"
---


Bir Python botu veya API yazdınız. Sunucuda `nohup python main.py &` komutunu çalıştırıp SSH'tan çıktınız.
İki gün sonra müşteri aradı: "Servis çalışmıyor!"
Meğer sunucu güncelleme alıp yeniden başlamış. Veya kod küçük bir hata yüzünden "Crash" olmuş.
Profesyonel üretim (production) ortamlarında scriptler elle çalıştırılmaz.
Linux'un kalbi olan **Systemd**, servislerinizi yönetmek, izlemek ve hayatta tutmak (Keep Alive) için vardır.
Bu yazıda, Python uygulamalarınızı nasıl "Daemon" haline getirip ölümsüzleştireceğinizi, Watchdog entegrasyonunu ve Cron yerine Timer kullanımını anlatacağım.

![Systemd Architecture](/assets/img/posts/systemd-components-architecture.png)
*Linux boot süreci ve Systemd servis hiyerarşisi.*

## 1. Unit Dosyasının Anatomisi

Systemd'e ne yapacağını söyleyen dosyalara "Unit File" denir. Genelde `/etc/systemd/system/` altında `.service` uzantısı ile dururlar.
İdeal bir Python servisi şöyle görünür:

```ini
# /etc/systemd/system/myservice.service

[Unit]
Description=My Critical Python Service
# Redis ve Network hazır olmadan başlama
After=network.target redis-server.service
Requires=redis-server.service
Documentation=https://wiki.mysite.com/bot

[Service]
Type=notify
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp

# Sanal ortamdaki Python'u kullanın
ExecStart=/opt/myapp/venv/bin/python main.py

# Çevresel Değişkenler (Secrets)
EnvironmentFile=/opt/myapp/.env

# Restart Stratejisi
Restart=on-failure
RestartSec=5s

# Watchdog Entegrasyonu (90s cevap gelmezse öldür)
WatchdogSec=90s

# Standart Çıktı -> Journald
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

## 2. Watchdog: "Sadece Çalışması Yetmez, Cevap Vermeli"

Çoğu kişi `Restart=always` ayarını yeterli sanır.
Ama ya kodunuz **Deadlock** olursa? Process çalışıyor görünür (PID var), ama aslında donmuştur.
Systemd bunu anlayamaz.
Çözüm: **Watchdog**.
Uygulamanız Systemd'e periyodik olarak "Ben iyiyim" sinyali (Heartbeat) göndermelidir. Göndermezse Systemd onu öldürür ve yeniden başlatır.

Python'da `sdnotify` kütüphanesi ile:

```python
import time
import sdnotify

n = sdnotify.SystemdNotifier()
n.notify("READY=1") # Başladım!

while True:
    try:
        # Ana iş mantığı
        do_heavy_work()
        
        # Systemd'e "Yaşıyorum" de
        n.notify("WATCHDOG=1")
    except Exception as e:
        # Hata durumunu bildir
        n.notify(f"STATUS=Hata oluştu: {str(e)}")
        time.sleep(5)
```
Bu kod sayesinde, uygulamanız sonsuz döngüde sıkışsa bile Systemd bunu fark eder ve restart atar.

## 3. Güvenlik: Sandbox İçinde Çalıştırma

Systemd'in modern sürümleri, Docker benzeri izolasyon özellikleri sunar.
Servisinizin dosya sistemine yazmasını veya kritik dizinleri görmesini engelleyebilirsiniz.

```ini
[Service]
# /usr, /boot gibi yerleri Read-Only yapar
ProtectSystem=full
# /home, /root ve /run/user gizlenir
ProtectHome=true
# /tmp için özel izole bir alan yaratır
PrivateTmp=true
# Process yeni yetki (sudo) alamaz
NoNewPrivileges=true
# Sadece IPv4/IPv6 soketleri açabilir
RestrictAddressFamilies=AF_INET AF_INET6
```
Bu satırlar, uygulamanızda RCE (Remote Code Execution) açığı olsa bile saldırganın sistemde gezmesini imkansız hale getirir.

## 4. Cron Yerine Systemd Timers

Zamanlanmış görevler (Cronjob) için "Crontab" eskide kaldı.
Systemd Timer'lar çok daha yeteneklidir:
1.  Logları `journalctl` ile görebilirsiniz.
2.  Biri bitmeden diğeri başlamaz (Overlapping engelleme).
3.  Servis bağımlılıklarını yönetebilirsiniz.

İki dosya gerekir: `.service` (ne yapacak) ve `.timer` (ne zaman yapacak).

**myscript.timer:**
```ini
[Unit]
Description=Run myscript every day at 3am

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=600 # 10 dk rastgele gecikme (Yükü dağıtmak için)
Persistent=true # Makine kapalıysa açılınca çalıştır

[Install]
WantedBy=timers.target
```
`systemctl enable --now myscript.timer` dediğinizde modern bir cronjob'ınız olur.

## 5. Log Yönetimi ve Analiz

Log dosyalarıyla (app.log) uğraşmayın. Standart çıktıya (`print`) yazın, Systemd yakalasın.

```bash
# Canlı log akışı
journalctl -u myservice -f

# Sadece "Error" seviyesindeki loglar
journalctl -u myservice -p err

# Belirli bir zaman aralığı
journalctl -u myservice --since "1 hour ago"

# JSON formatında çıktı (Log toplayıcılar için)
journalctl -u myservice -o json
```
Bu loglar binary formatta sıkıştırılarak saklanır, diskte yer kaplamaz ve bozulmaz.

## 6. Socket Activation (İleri Seviye)

Systemd'in en "cool" özelliği **Socket Activation**dır.
Servisiniz (Gunicorn vb.) başlamadan Systemd 80 portunu dinler.
İlk istek geldiği AN servisi başlatır ve soketi ona (File Descriptor olarak) verir.

Avantajları:
1.  **Sıfır Downtime Update:** Servisi kapatıp açarken soket Systemd'de olduğu için bağlantılar kopmaz, kuyrukta bekler.
2.  **On-Demand Start:** Hiç kullanılmayan admin panelleri RAM tüketmez, istek gelince uyanır.

```pop
# myservice.socket
[Socket]
ListenStream=8000
```
Python tarafında bu soketi dinlemek için `socket.fromfd()` kullanılır.

![Systemd Security](/assets/img/posts/systemd-unit-file-visual-guide.png)
*Sandboxing ve yetki kısıtlama mekanizmaları.*

## Sonuç

Docker ve Kubernetes çağında bile Systemd, Linux sunucu yönetiminin temel taşıdır.
Bir Python scriptini Systemd servisine çevirmek 5 dakikanızı alır, ama size kazandırdığı huzur (Peace of Mind) paha biçilemez.
`Watchdog` ile donmaları yakalamak, `Timer` ile cron'u modernize etmek ve `Security` özellikleri ile "Hardening" yapmak, Senior bir DevOps/Backend mühendisinin yetkinlik setinde mutlaka olmalıdır.
Gece 3'te servisin çöküp otomatik başladığını ve müşterinin hiçbir şey hissetmediğini bilmek, iyi bir uyku için en iyi reçetedir.

