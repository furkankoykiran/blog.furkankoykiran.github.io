---
title: "Python ile E-posta Otomasyonu"
date: "2025-11-08 09:00:00 +0300"
categories: [Python, Automation]
tags: [python, email, smtp, automation, notification, alert, jinja2, html-email, mailgun, sendgrid]
image:
  src: /assets/img/posts/python-smtp-email-workflow.png
  alt: "Python SMTP E-posta Workflow Diyagramı"
---

E-posta otomasyonu, modern uygulamalarda kullanıcı bildirimleri, raporlama, alert sistemleri ve pazarlama kampanyaları için vazgeçilmez bir özelliktir. Python, SMTP protokolü ve çeşitli e-posta servisleri ile güçlü otomasyon çözümleri sunar. Bu yazıda, Python ile e-posta gönderimi, HTML şablonları, toplu e-posta, error handling ve best practice'leri ele alacağız.

## SMTP Protokolü ve Python smtplib

SMTP (Simple Mail Transfer Protocol), e-posta göndermek için kullanılan standart protokoldür. Python'ın built-in `smtplib` modülü ile kolayca e-posta gönderebilirsiniz.

### Temel E-posta Gönderimi

```python
import smtplib
from email.message import EmailMessage

def send_simple_email():
    """Basit metin e-postası gönder"""
    # Email mesajı oluştur
    msg = EmailMessage()
    msg['Subject'] = 'Test E-postası'
    msg['From'] = 'gonderici@example.com'
    msg['To'] = 'alici@example.com'
    msg.set_content('Bu bir test e-postasıdır.')
    
    # SMTP sunucusuna bağlan ve gönder
    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()  # TLS encryption başlat
        smtp.login('gonderici@example.com', 'uygulama_sifresi')
        smtp.send_message(msg)
        print("E-posta başarıyla gönderildi!")

# Gmail için uygulama şifresi oluşturma:
# https://myaccount.google.com/apppasswords
```

### Çoklu Alıcı ve CC/BCC

```python
from email.message import EmailMessage
import smtplib

def send_multi_recipient_email():
    """Çoklu alıcıya e-posta gönder"""
    msg = EmailMessage()
    msg['Subject'] = 'Önemli Duyuru'
    msg['From'] = 'admin@company.com'
    
    # Birden fazla alıcı
    msg['To'] = 'user1@example.com, user2@example.com'
    
    # CC (Carbon Copy) - Görünür kopya
    msg['Cc'] = 'manager@company.com'
    
    # BCC (Blind Carbon Copy) - Gizli kopya
    # BCC başlıkta görünmez, send_message parametresinde belirtilir
    bcc_recipients = ['secret@example.com', 'audit@company.com']
    
    msg.set_content("""
    Merhaba Takım,
    
    Bu hafta sonu planlı bakım çalışması olacaktır.
    
    Saygılarımızla,
    IT Ekibi
    """)
    
    # Tüm alıcıları birleştir
    all_recipients = (
        msg['To'].split(', ') + 
        msg['Cc'].split(', ') + 
        bcc_recipients
    )
    
    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('admin@company.com', 'app_password')
        smtp.send_message(msg, to_addrs=all_recipients)
    
    print(f"E-posta {len(all_recipients)} kişiye gönderildi")
```

## HTML E-posta ve Template Rendering

Profesyonel e-postalar için HTML içerik ve template engine kullanımı önemlidir.

![Jinja2 Email Template Rendering](/assets/img/posts/jinja2-email-template-rendering.jpg)

### Jinja2 ile HTML Template

```python
from jinja2 import Template
from email.message import EmailMessage
from email.utils import make_msgid
import smtplib

# HTML template
EMAIL_TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            color: #333;
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
        }
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            text-align: center;
            border-radius: 10px 10px 0 0;
        }
        .content {
            background: #f9f9f9;
            padding: 30px;
            border-radius: 0 0 10px 10px;
        }
        .button {
            display: inline-block;
            padding: 12px 30px;
            background: #667eea;
            color: white;
            text-decoration: none;
            border-radius: 5px;
            margin: 20px 0;
        }
        .footer {
            text-align: center;
            margin-top: 20px;
            color: #888;
            font-size: 12px;
        }
        .stats {
            display: flex;
            justify-content: space-around;
            margin: 20px 0;
        }
        .stat-box {
            text-align: center;
            padding: 15px;
            background: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }
        .stat-number {
            font-size: 24px;
            font-weight: bold;
            color: #667eea;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>{{ title }}</h1>
        <p>{{ subtitle }}</p>
    </div>
    
    <div class="content">
        <p>Merhaba {{ user_name }},</p>
        
        <p>{{ message }}</p>
        
        {% if stats %}
        <div class="stats">
            {% for stat in stats %}
            <div class="stat-box">
                <div class="stat-number">{{ stat.value }}</div>
                <div>{{ stat.label }}</div>
            </div>
            {% endfor %}
        </div>
        {% endif %}
        
        {% if action_url %}
        <center>
            <a href="{{ action_url }}" class="button">{{ action_text }}</a>
        </center>
        {% endif %}
        
        <p>İyi çalışmalar dileriz!</p>
    </div>
    
    <div class="footer">
        <p>Bu e-posta {{ company_name }} tarafından gönderilmiştir.</p>
        <p>{{ year }} © Tüm hakları saklıdır.</p>
    </div>
</body>
</html>
"""

def send_html_email(to_email, context):
    """HTML template ile e-posta gönder"""
    # Template render et
    template = Template(EMAIL_TEMPLATE)
    html_content = template.render(**context)
    
    # Email mesajı oluştur
    msg = EmailMessage()
    msg['Subject'] = context['title']
    msg['From'] = 'noreply@company.com'
    msg['To'] = to_email
    
    # Plain text fallback
    msg.set_content(f"""
    {context['title']}
    
    Merhaba {context['user_name']},
    
    {context['message']}
    
    {context.get('action_url', '')}
    """)
    
    # HTML içerik ekle
    msg.add_alternative(html_content, subtype='html')
    
    # E-postayı gönder
    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('sender@example.com', 'app_password')
        smtp.send_message(msg)

# Kullanım örneği
context = {
    'title': 'Haftalık Rapor',
    'subtitle': '1-7 Kasım 2025',
    'user_name': 'Ahmet Yılmaz',
    'message': 'Bu hafta harika bir performans sergiledsiniz!',
    'stats': [
        {'value': '127', 'label': 'Tamamlanan Görev'},
        {'value': '98%', 'label': 'Başarı Oranı'},
        {'value': '4.8', 'label': 'Ortalama Puan'}
    ],
    'action_url': 'https://dashboard.company.com',
    'action_text': 'Dashboard\'a Git',
    'company_name': 'ABC Şirketi',
    'year': 2025
}

send_html_email('ahmet@example.com', context)
```

## Ek Dosya (Attachment) Gönderimi

```python
from email.message import EmailMessage
import smtplib
from pathlib import Path
import mimetypes

def send_email_with_attachments(
    to_email,
    subject,
    body,
    attachments=None
):
    """Ek dosyalarla e-posta gönder"""
    msg = EmailMessage()
    msg['Subject'] = subject
    msg['From'] = 'sender@example.com'
    msg['To'] = to_email
    msg.set_content(body)
    
    # Ek dosyaları ekle
    if attachments:
        for file_path in attachments:
            path = Path(file_path)
            
            # MIME type'ı belirle
            mime_type, _ = mimetypes.guess_type(file_path)
            if mime_type is None:
                mime_type = 'application/octet-stream'
            
            maintype, subtype = mime_type.split('/', 1)
            
            # Dosyayı oku ve ekle
            with open(file_path, 'rb') as f:
                msg.add_attachment(
                    f.read(),
                    maintype=maintype,
                    subtype=subtype,
                    filename=path.name
                )
    
    # Gönder
    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('sender@example.com', 'app_password')
        smtp.send_message(msg)
    
    print(f"E-posta gönderildi: {len(attachments or [])} ek dosya")

# Kullanım
send_email_with_attachments(
    to_email='client@example.com',
    subject='Aylık Rapor',
    body='Ekteki raporları inceleyebilirsiniz.',
    attachments=[
        'reports/november_sales.pdf',
        'reports/november_analytics.xlsx',
        'reports/summary.docx'
    ]
)

# In-memory attachment (pandas DataFrame örneği)
import io
import pandas as pd

def send_dataframe_as_excel(to_email, df, filename='report.xlsx'):
    """DataFrame'i Excel eki olarak gönder"""
    msg = EmailMessage()
    msg['Subject'] = 'Veri Raporu'
    msg['From'] = 'analytics@company.com'
    msg['To'] = to_email
    msg.set_content('Ekteki Excel dosyasını inceleyebilirsiniz.')
    
    # DataFrame'i memory'de Excel'e çevir
    excel_buffer = io.BytesIO()
    with pd.ExcelWriter(excel_buffer, engine='openpyxl') as writer:
        df.to_excel(writer, index=False, sheet_name='Data')
    
    excel_buffer.seek(0)
    
    # Excel dosyasını ekle
    msg.add_attachment(
        excel_buffer.read(),
        maintype='application',
        subtype='vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        filename=filename
    )
    
    # Gönder
    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('analytics@company.com', 'app_password')
        smtp.send_message(msg)
```

## Toplu E-posta Gönderimi

![Email Automation Architecture](/assets/img/posts/email-automation-python-architecture.png)

```python
import smtplib
from email.message import EmailMessage
from concurrent.futures import ThreadPoolExecutor, as_completed
import time
from dataclasses import dataclass
from typing import List, Dict
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class EmailRecipient:
    """E-posta alıcı bilgileri"""
    email: str
    name: str
    custom_data: Dict = None

class BulkEmailSender:
    """Toplu e-posta gönderme sınıfı"""
    
    def __init__(
        self,
        smtp_host='smtp.gmail.com',
        smtp_port=587,
        username=None,
        password=None,
        rate_limit=10  # Saniyede max e-posta
    ):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port
        self.username = username
        self.password = password
        self.rate_limit = rate_limit
        self.sent_count = 0
        self.failed_count = 0
    
    def send_single_email(
        self,
        recipient: EmailRecipient,
        subject: str,
        template: str
    ) -> bool:
        """Tek bir e-posta gönder"""
        try:
            # Template'i render et
            from jinja2 import Template
            content = Template(template).render(
                name=recipient.name,
                email=recipient.email,
                **(recipient.custom_data or {})
            )
            
            # Email oluştur
            msg = EmailMessage()
            msg['Subject'] = subject
            msg['From'] = self.username
            msg['To'] = recipient.email
            msg.add_alternative(content, subtype='html')
            
            # Gönder
            with smtplib.SMTP(self.smtp_host, self.smtp_port) as smtp:
                smtp.starttls()
                smtp.login(self.username, self.password)
                smtp.send_message(msg)
            
            self.sent_count += 1
            logger.info(f"✓ E-posta gönderildi: {recipient.email}")
            return True
            
        except Exception as e:
            self.failed_count += 1
            logger.error(f"✗ Hata ({recipient.email}): {str(e)}")
            return False
    
    def send_bulk(
        self,
        recipients: List[EmailRecipient],
        subject: str,
        template: str,
        max_workers=5
    ) -> Dict:
        """Toplu e-posta gönder"""
        logger.info(f"Toplu e-posta başlıyor: {len(recipients)} alıcı")
        start_time = time.time()
        
        # Rate limiting için delay hesapla
        delay_between_emails = 1.0 / self.rate_limit
        
        results = []
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            # Tüm görevleri submit et
            future_to_recipient = {
                executor.submit(
                    self.send_single_email,
                    recipient,
                    subject,
                    template
                ): recipient
                for recipient in recipients
            }
            
            # Sonuçları topla
            for future in as_completed(future_to_recipient):
                recipient = future_to_recipient[future]
                try:
                    success = future.result()
                    results.append({
                        'email': recipient.email,
                        'success': success
                    })
                    
                    # Rate limiting
                    time.sleep(delay_between_emails)
                    
                except Exception as e:
                    logger.error(f"Unexpected error: {str(e)}")
                    results.append({
                        'email': recipient.email,
                        'success': False
                    })
        
        elapsed = time.time() - start_time
        
        summary = {
            'total': len(recipients),
            'sent': self.sent_count,
            'failed': self.failed_count,
            'elapsed_seconds': elapsed,
            'emails_per_second': len(recipients) / elapsed,
            'results': results
        }
        
        logger.info(f"""
        Toplu e-posta tamamlandı:
        - Gönderilen: {self.sent_count}
        - Başarısız: {self.failed_count}
        - Süre: {elapsed:.2f} saniye
        - Hız: {summary['emails_per_second']:.2f} e-posta/saniye
        """)
        
        return summary

# Kullanım örneği
if __name__ == '__main__':
    # Alıcı listesi
    recipients = [
        EmailRecipient(
            email='user1@example.com',
            name='Ahmet Yılmaz',
            custom_data={'subscription': 'Premium', 'score': 95}
        ),
        EmailRecipient(
            email='user2@example.com',
            name='Ayşe Demir',
            custom_data={'subscription': 'Basic', 'score': 87}
        ),
        # ... daha fazla alıcı
    ]
    
    # HTML template
    template = """
    <html>
    <body>
        <h2>Merhaba {{ name }}!</h2>
        <p>{{ subscription }} aboneliğiniz için teşekkürler.</p>
        <p>Bu ayki puanınız: <strong>{{ score }}</strong></p>
    </body>
    </html>
    """
    
    # Toplu gönder
    sender = BulkEmailSender(
        username='sender@example.com',
        password='app_password',
        rate_limit=10  # 10 e-posta/saniye
    )
    
    results = sender.send_bulk(
        recipients=recipients,
        subject='Aylık Performans Raporu',
        template=template,
        max_workers=3
    )
```

## Email Service Provider Entegrasyonları

Production'da SMTP yerine özel e-posta servisleri kullanmak daha güvenilirdir.

### SendGrid Entegrasyonu

```python
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail, Email, To, Content

class SendGridEmailer:
    """SendGrid ile e-posta gönderimi"""
    
    def __init__(self, api_key):
        self.client = SendGridAPIClient(api_key)
    
    def send_simple(self, to_email, subject, html_content):
        """Basit e-posta gönder"""
        message = Mail(
            from_email='noreply@company.com',
            to_emails=to_email,
            subject=subject,
            html_content=html_content
        )
        
        try:
            response = self.client.send(message)
            return {
                'status_code': response.status_code,
                'success': 200 <= response.status_code < 300
            }
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    def send_with_template(
        self,
        to_email,
        template_id,
        dynamic_data
    ):
        """SendGrid template ile gönder"""
        message = Mail(
            from_email='noreply@company.com',
            to_emails=to_email
        )
        
        message.template_id = template_id
        message.dynamic_template_data = dynamic_data
        
        try:
            response = self.client.send(message)
            return {'success': True, 'status': response.status_code}
        except Exception as e:
            return {'success': False, 'error': str(e)}

# Kullanım
sendgrid = SendGridEmailer(api_key='SG.xxx')

result = sendgrid.send_simple(
    to_email='user@example.com',
    subject='Hoş Geldiniz!',
    html_content='<h1>Aramıza hoş geldiniz!</h1>'
)
```

### Mailgun Entegrasyonu

```python
import requests

class MailgunEmailer:
    """Mailgun ile e-posta gönderimi"""
    
    def __init__(self, api_key, domain):
        self.api_key = api_key
        self.domain = domain
        self.base_url = f'https://api.mailgun.net/v3/{domain}'
    
    def send_email(
        self,
        to_email,
        subject,
        html_content,
        text_content=None,
        attachments=None
    ):
        """E-posta gönder"""
        data = {
            'from': f'System <noreply@{self.domain}>',
            'to': to_email,
            'subject': subject,
            'html': html_content
        }
        
        if text_content:
            data['text'] = text_content
        
        files = None
        if attachments:
            files = [
                ('attachment', open(f, 'rb'))
                for f in attachments
            ]
        
        response = requests.post(
            f'{self.base_url}/messages',
            auth=('api', self.api_key),
            data=data,
            files=files
        )
        
        return {
            'success': response.status_code == 200,
            'response': response.json()
        }
    
    def send_bulk(self, recipients, subject, html_content):
        """Toplu e-posta (recipient variables)"""
        # Mailgun bulk gönderim
        data = {
            'from': f'Newsletter <noreply@{self.domain}>',
            'to': [r['email'] for r in recipients],
            'subject': subject,
            'html': html_content,
            'recipient-variables': {
                r['email']: r['data']
                for r in recipients
            }
        }
        
        response = requests.post(
            f'{self.base_url}/messages',
            auth=('api', self.api_key),
            data=data
        )
        
        return response.json()

# Kullanım
mailgun = MailgunEmailer(
    api_key='key-xxx',
    domain='mg.company.com'
)

result = mailgun.send_email(
    to_email='user@example.com',
    subject='Test E-posta',
    html_content='<h1>Merhaba Dünya!</h1>',
    attachments=['report.pdf']
)
```

## Error Handling ve Retry Logic

```python
import smtplib
from email.message import EmailMessage
import time
from functools import wraps
import logging

logger = logging.getLogger(__name__)

class EmailSendError(Exception):
    """E-posta gönderme hatası"""
    pass

def retry_on_failure(max_retries=3, delay=2, backoff=2):
    """Retry decorator"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            retries = 0
            current_delay = delay
            
            while retries < max_retries:
                try:
                    return func(*args, **kwargs)
                except (smtplib.SMTPException, ConnectionError) as e:
                    retries += 1
                    if retries >= max_retries:
                        logger.error(
                            f"Failed after {max_retries} retries: {str(e)}"
                        )
                        raise EmailSendError(
                            f"Could not send email after {max_retries} attempts"
                        )
                    
                    logger.warning(
                        f"Attempt {retries} failed: {str(e)}. "
                        f"Retrying in {current_delay}s..."
                    )
                    time.sleep(current_delay)
                    current_delay *= backoff
            
        return wrapper
    return decorator

class RobustEmailSender:
    """Hata yönetimi olan e-posta gönderici"""
    
    def __init__(self, smtp_config):
        self.smtp_config = smtp_config
    
    @retry_on_failure(max_retries=3, delay=2, backoff=2)
    def send_email(self, msg):
        """Retry logic ile e-posta gönder"""
        try:
            with smtplib.SMTP(
                self.smtp_config['host'],
                self.smtp_config['port'],
                timeout=30
            ) as smtp:
                smtp.starttls()
                smtp.login(
                    self.smtp_config['username'],
                    self.smtp_config['password']
                )
                smtp.send_message(msg)
                
            logger.info(f"Email sent successfully to {msg['To']}")
            return True
            
        except smtplib.SMTPAuthenticationError:
            logger.error("SMTP authentication failed")
            raise
        
        except smtplib.SMTPRecipientsRefused:
            logger.error(f"Recipient refused: {msg['To']}")
            raise
        
        except smtplib.SMTPServerDisconnected:
            logger.warning("SMTP server disconnected, will retry")
            raise ConnectionError("SMTP disconnected")
        
        except Exception as e:
            logger.error(f"Unexpected error: {str(e)}")
            raise
    
    def send_with_fallback(self, msg, fallback_config=None):
        """Ana sunucu başarısız olursa fallback sunucu dene"""
        try:
            return self.send_email(msg)
        except Exception as e:
            if fallback_config:
                logger.warning(
                    f"Primary SMTP failed, trying fallback: {str(e)}"
                )
                # Geçici olarak fallback config kullan
                original_config = self.smtp_config
                self.smtp_config = fallback_config
                try:
                    return self.send_email(msg)
                finally:
                    self.smtp_config = original_config
            else:
                raise

# Kullanım
smtp_config = {
    'host': 'smtp.gmail.com',
    'port': 587,
    'username': 'sender@example.com',
    'password': 'app_password'
}

fallback_config = {
    'host': 'smtp.sendgrid.net',
    'port': 587,
    'username': 'apikey',
    'password': 'SG.xxx'
}

sender = RobustEmailSender(smtp_config)

msg = EmailMessage()
msg['Subject'] = 'Test'
msg['From'] = 'sender@example.com'
msg['To'] = 'recipient@example.com'
msg.set_content('Test message')

try:
    sender.send_with_fallback(msg, fallback_config)
except EmailSendError as e:
    logger.error(f"Could not send email: {str(e)}")
```

## Production Best Practices

### 1. Email Queue Sistemi

```python
import redis
import json
from datetime import datetime
import uuid

class EmailQueue:
    """Redis ile e-posta kuyruğu"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.queue_key = 'email:queue'
        self.processing_key = 'email:processing'
        self.failed_key = 'email:failed'
    
    def enqueue(self, email_data):
        """E-postayı kuyruğa ekle"""
        email_id = str(uuid.uuid4())
        email_job = {
            'id': email_id,
            'data': email_data,
            'created_at': datetime.utcnow().isoformat(),
            'attempts': 0
        }
        
        self.redis.lpush(
            self.queue_key,
            json.dumps(email_job)
        )
        
        return email_id
    
    def dequeue(self):
        """Kuyruktan e-posta al"""
        # BRPOPLPUSH: atomik dequeue + processing kuyruğuna ekle
        item = self.redis.brpoplpush(
            self.queue_key,
            self.processing_key,
            timeout=5
        )
        
        if item:
            return json.loads(item)
        return None
    
    def mark_completed(self, email_job):
        """E-posta gönderimini tamamlandı işaretle"""
        self.redis.lrem(
            self.processing_key,
            1,
            json.dumps(email_job)
        )
    
    def mark_failed(self, email_job, error):
        """Başarısız e-postayı kaydet"""
        email_job['error'] = str(error)
        email_job['failed_at'] = datetime.utcnow().isoformat()
        
        self.redis.lpush(
            self.failed_key,
            json.dumps(email_job)
        )
        
        self.redis.lrem(
            self.processing_key,
            1,
            json.dumps(email_job)
        )
    
    def retry_failed(self, email_job, max_attempts=3):
        """Başarısız e-postayı tekrar dene"""
        email_job['attempts'] += 1
        
        if email_job['attempts'] < max_attempts:
            # Tekrar kuyruğa ekle
            self.redis.lpush(
                self.queue_key,
                json.dumps(email_job)
            )
            self.redis.lrem(
                self.processing_key,
                1,
                json.dumps(email_job)
            )
            return True
        else:
            # Max deneme aşıldı, failed'a taşı
            self.mark_failed(email_job, "Max attempts exceeded")
            return False

# Worker
import time

def email_worker(queue, sender):
    """E-posta gönderme worker'ı"""
    logger.info("Email worker started")
    
    while True:
        try:
            # Kuyruktan e-posta al
            email_job = queue.dequeue()
            
            if email_job:
                logger.info(f"Processing email: {email_job['id']}")
                
                try:
                    # E-postayı gönder
                    email_data = email_job['data']
                    msg = EmailMessage()
                    msg['Subject'] = email_data['subject']
                    msg['From'] = email_data['from']
                    msg['To'] = email_data['to']
                    msg.set_content(email_data['body'])
                    
                    sender.send_email(msg)
                    
                    # Başarılı, completed işaretle
                    queue.mark_completed(email_job)
                    logger.info(f"Email sent: {email_job['id']}")
                    
                except Exception as e:
                    logger.error(f"Failed to send email: {str(e)}")
                    
                    # Retry veya failed'a taşı
                    if not queue.retry_failed(email_job):
                        logger.error(
                            f"Email permanently failed: {email_job['id']}"
                        )
            
            time.sleep(0.1)  # Rate limiting
            
        except KeyboardInterrupt:
            logger.info("Worker stopped by user")
            break
        
        except Exception as e:
            logger.error(f"Worker error: {str(e)}")
            time.sleep(1)
```

### 2. Email Analytics ve Tracking

```python
from datetime import datetime
import hashlib

class EmailTracker:
    """E-posta tracking ve analytics"""
    
    def __init__(self, database):
        self.db = database
    
    def generate_tracking_pixel(self, email_id):
        """Tracking pixel URL oluştur"""
        token = hashlib.md5(
            f"{email_id}{datetime.utcnow()}".encode()
        ).hexdigest()
        
        return f"https://track.company.com/pixel/{email_id}/{token}.gif"
    
    def generate_click_tracking_url(self, email_id, original_url):
        """Link tracking URL oluştur"""
        token = hashlib.md5(
            f"{email_id}{original_url}".encode()
        ).hexdigest()
        
        return f"https://track.company.com/click/{email_id}/{token}"
    
    def add_tracking_to_html(self, email_id, html_content):
        """HTML içeriğe tracking ekle"""
        # Tracking pixel ekle
        pixel_url = self.generate_tracking_pixel(email_id)
        tracking_pixel = f'<img src="{pixel_url}" width="1" height="1" />'
        
        # HTML'in sonuna ekle
        if '</body>' in html_content:
            html_content = html_content.replace(
                '</body>',
                f'{tracking_pixel}</body>'
            )
        else:
            html_content += tracking_pixel
        
        # Link'leri tracking ile değiştir
        # (Gerçek implementasyonda BeautifulSoup kullan)
        
        return html_content
    
    def record_open(self, email_id, user_agent, ip_address):
        """E-posta açılışını kaydet"""
        self.db.execute("""
            INSERT INTO email_opens (
                email_id, opened_at, user_agent, ip_address
            ) VALUES (?, ?, ?, ?)
        """, (email_id, datetime.utcnow(), user_agent, ip_address))
    
    def record_click(self, email_id, url, user_agent, ip_address):
        """Link tıklamasını kaydet"""
        self.db.execute("""
            INSERT INTO email_clicks (
                email_id, url, clicked_at, user_agent, ip_address
            ) VALUES (?, ?, ?, ?, ?)
        """, (email_id, url, datetime.utcnow(), user_agent, ip_address))
    
    def get_analytics(self, email_id):
        """E-posta analytics getir"""
        return {
            'sent_at': self.db.query(
                "SELECT sent_at FROM emails WHERE id = ?",
                (email_id,)
            ),
            'opens': self.db.query(
                "SELECT COUNT(*) FROM email_opens WHERE email_id = ?",
                (email_id,)
            ),
            'unique_opens': self.db.query(
                "SELECT COUNT(DISTINCT ip_address) FROM email_opens "
                "WHERE email_id = ?",
                (email_id,)
            ),
            'clicks': self.db.query(
                "SELECT COUNT(*) FROM email_clicks WHERE email_id = ?",
                (email_id,)
            ),
            'click_rate': self.db.query(
                "SELECT (COUNT(DISTINCT email_clicks.email_id) * 100.0 / "
                "COUNT(DISTINCT email_opens.email_id)) "
                "FROM email_clicks, email_opens "
                "WHERE email_clicks.email_id = ? AND email_opens.email_id = ?",
                (email_id, email_id)
            )
        }
```

## Sonuç

Python ile e-posta otomasyonu, modern uygulamaların vazgeçilmez bir parçasıdır. SMTP protokolü, HTML template'leri, toplu e-posta, error handling ve email service provider entegrasyonları ile profesyonel e-posta sistemleri oluşturabilirsiniz.

Bu yazıda ele aldığımız konular:
- SMTP protokolü ve smtplib kullanımı
- HTML e-posta ve Jinja2 template rendering
- Ek dosya (attachment) gönderimi
- Toplu e-posta gönderimi ve rate limiting
- SendGrid ve Mailgun entegrasyonları
- Error handling ve retry logic
- Email queue sistemi (Redis)
- Email tracking ve analytics
- Production best practices

Başarılı bir e-posta otomasyonu sistemi, reliability, deliverability ve analytics ile kullanıcı deneyimini artırır.

**Kaynaklar:**
- [Python smtplib Documentation](https://docs.python.org/3/library/smtplib.html)
- [Jinja2 Template Engine](https://jinja.palletsprojects.com/)
- [SendGrid Python Library](https://github.com/sendgrid/sendgrid-python)
- [Mailgun API Documentation](https://documentation.mailgun.com/)
