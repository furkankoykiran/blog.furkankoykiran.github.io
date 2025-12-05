---
title: "API Kimlik Doğrulama Stratejileri: JWT, OAuth2 ve Güvenlik En İyi Uygulamaları"
description: "JWT, OAuth2, API Key gibi kimlik doğrulama yöntemleri. FastAPI ile güvenli API geliştirme, RBAC, rate limiting ve güvenlik best practices."
date: 2025-06-18 10:00:00 +0300
categories: [Security, API Development]
tags: [api, authentication, jwt, oauth, api-key, security, fastapi]
image:
  path: /assets/img/posts/api-jwt-authentication-flow.jpg
  alt: "API JWT Kimlik Doğrulama Akışı"
---

Modern web uygulamalarında API güvenliği, en kritik konulardan biridir. Yanlış yapılandırılmış bir kimlik doğrulama sistemi, veri sızıntılarına, yetkisiz erişimlere ve ciddi güvenlik açıklarına yol açabilir. Bu yazıda, API kimlik doğrulama stratejilerini, JWT token'larını, OAuth2 protokolünü ve güvenlik en iyi uygulamalarını detaylı olarak inceleyeceğiz.

## API Kimlik Doğrulama Nedir?

API kimlik doğrulama, bir kullanıcının veya uygulamanın kimliğini doğrulayarak API kaynaklarına erişim yetkisi verme sürecidir. Bu süreç, sistemin güvenliğini sağlamak için kritik öneme sahiptir. Doğru kimlik doğrulama stratejisi seçmek, uygulamanızın güvenliği, performansı ve ölçeklenebilirliği açısından hayati önem taşır.

### Temel Kimlik Doğrulama Yöntemleri

1. **API Key Authentication**: Basit, statik anahtar tabanlı kimlik doğrulama
2. **JWT (JSON Web Token)**: Token tabanlı, stateless kimlik doğrulama
3. **OAuth2**: Yetkilendirme framework'ü, üçüncü taraf erişim kontrolü
4. **Basic Authentication**: HTTP header'da kullanıcı adı/şifre gönderimi
5. **Session-based Authentication**: Sunucu tarafında oturum yönetimi

## JWT (JSON Web Token) ile Kimlik Doğrulama

JWT, modern web uygulamalarında en yaygın kullanılan kimlik doğrulama yöntemlerinden biridir. Stateless yapısı sayesinde ölçeklenebilir sistemler için idealdir.

### JWT Yapısı

JWT üç bölümden oluşur:

```python
# JWT Token Yapısı
# header.payload.signature

# Header (Algoritma ve Token Tipi)
{
  "alg": "HS256",
  "typ": "JWT"
}

# Payload (Kullanıcı Bilgileri ve Claims)
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622
}

# Signature (İmza)
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

### FastAPI ile JWT Implementasyonu

```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel

# Güvenlik yapılandırması
SECRET_KEY = "your-secret-key-keep-it-secret"  # Üretimde environment variable kullanın
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Password hashing için context
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# OAuth2 scheme tanımı
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

app = FastAPI()


# Pydantic modelleri
class Token(BaseModel):
    access_token: str
    token_type: str


class TokenData(BaseModel):
    username: Optional[str] = None


class User(BaseModel):
    username: str
    email: Optional[str] = None
    full_name: Optional[str] = None
    disabled: Optional[bool] = None


class UserInDB(User):
    hashed_password: str


# Örnek kullanıcı veritabanı (üretimde gerçek veritabanı kullanın)
fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW",
        "disabled": False,
    }
}


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Şifre doğrulama"""
    return pwd_context.verify(plain_password, hashed_password)


def get_password_hash(password: str) -> str:
    """Şifre hashleme"""
    return pwd_context.hash(password)


def get_user(db, username: str):
    """Kullanıcı bilgisi getirme"""
    if username in db:
        user_dict = db[username]
        return UserInDB(**user_dict)


def authenticate_user(fake_db, username: str, password: str):
    """Kullanıcı kimlik doğrulama"""
    user = get_user(fake_db, username)
    if not user:
        return False
    if not verify_password(password, user.hashed_password):
        return False
    return user


def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    """JWT access token oluşturma"""
    to_encode = data.copy()
    
    # Token süresini belirleme
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    # Expire claim ekleme
    to_encode.update({"exp": expire})
    
    # Token'ı encode etme
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt


async def get_current_user(token: str = Depends(oauth2_scheme)):
    """Token'dan mevcut kullanıcıyı getirme"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        # Token'ı decode etme
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        
        if username is None:
            raise credentials_exception
            
        token_data = TokenData(username=username)
        
    except JWTError:
        raise credentials_exception
    
    # Kullanıcı bilgisini getirme
    user = get_user(fake_users_db, username=token_data.username)
    if user is None:
        raise credentials_exception
        
    return user


async def get_current_active_user(current_user: User = Depends(get_current_user)):
    """Aktif kullanıcı kontrolü"""
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user


@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login endpoint - JWT token üretimi"""
    # Kullanıcı kimlik doğrulama
    user = authenticate_user(fake_users_db, form_data.username, form_data.password)
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # Access token oluşturma
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    
    return {"access_token": access_token, "token_type": "bearer"}


@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    """Mevcut kullanıcı bilgisi endpoint'i"""
    return current_user


@app.get("/protected-data")
async def get_protected_data(current_user: User = Depends(get_current_active_user)):
    """Korumalı veri endpoint'i"""
    return {
        "message": "This is protected data",
        "user": current_user.username,
        "data": ["sensitive", "information", "here"]
    }
```

### JWT Token Kullanımı

```python
import requests
from typing import Optional

class APIClient:
    """JWT tabanlı API client"""
    
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.access_token: Optional[str] = None
    
    def login(self, username: str, password: str) -> bool:
        """Login ve token alma"""
        try:
            response = requests.post(
                f"{self.base_url}/token",
                data={
                    "username": username,
                    "password": password
                }
            )
            response.raise_for_status()
            
            token_data = response.json()
            self.access_token = token_data["access_token"]
            return True
            
        except requests.RequestException as e:
            print(f"Login failed: {e}")
            return False
    
    def get_headers(self) -> dict:
        """Authorization header oluşturma"""
        if not self.access_token:
            raise Exception("Not authenticated. Please login first.")
        
        return {
            "Authorization": f"Bearer {self.access_token}"
        }
    
    def get_user_info(self) -> dict:
        """Kullanıcı bilgisi getirme"""
        response = requests.get(
            f"{self.base_url}/users/me",
            headers=self.get_headers()
        )
        response.raise_for_status()
        return response.json()
    
    def get_protected_data(self) -> dict:
        """Korumalı veri getirme"""
        response = requests.get(
            f"{self.base_url}/protected-data",
            headers=self.get_headers()
        )
        response.raise_for_status()
        return response.json()


# Kullanım örneği
if __name__ == "__main__":
    # Client oluşturma
    client = APIClient("http://localhost:8000")
    
    # Login işlemi
    if client.login("johndoe", "secret"):
        print("Login successful!")
        
        # Kullanıcı bilgisi
        user_info = client.get_user_info()
        print(f"User: {user_info}")
        
        # Korumalı veri
        protected_data = client.get_protected_data()
        print(f"Protected Data: {protected_data}")
```

## Refresh Token ile Token Yenileme

![API Refresh Token Akışı](/assets/img/posts/api-refresh-token-flow.png){: w="700" h="400" .shadow }
_API refresh token akış diyagramı_
_Refresh token kullanarak access token yenileme süreci_

Güvenlik ve kullanıcı deneyimi dengesini sağlamak için, kısa ömürlü access token'lar ve uzun ömürlü refresh token'lar kullanılır.

```python
from datetime import datetime, timedelta
from typing import Optional
from jose import jwt
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel
import secrets

app = FastAPI()

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15  # Kısa ömürlü
REFRESH_TOKEN_EXPIRE_DAYS = 7  # Uzun ömürlü

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Refresh token store (üretimde Redis kullanın)
refresh_token_store = {}


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str


class RefreshTokenRequest(BaseModel):
    refresh_token: str


def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """Access token oluşturma"""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode.update({
        "exp": expire,
        "type": "access"
    })
    
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def create_refresh_token(data: dict) -> str:
    """Refresh token oluşturma"""
    to_encode = data.copy()
    
    # Refresh token için unique ID
    jti = secrets.token_urlsafe(32)
    expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    
    to_encode.update({
        "exp": expire,
        "type": "refresh",
        "jti": jti
    })
    
    refresh_token = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    # Refresh token'ı store'a kaydetme
    refresh_token_store[jti] = {
        "username": data.get("sub"),
        "created_at": datetime.utcnow(),
        "expires_at": expire
    }
    
    return refresh_token


def verify_refresh_token(refresh_token: str) -> Optional[str]:
    """Refresh token doğrulama"""
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        
        # Token tipini kontrol etme
        if payload.get("type") != "refresh":
            return None
        
        # JTI kontrolü (token store'da var mı?)
        jti = payload.get("jti")
        if jti not in refresh_token_store:
            return None
        
        # Username'i döndürme
        username = payload.get("sub")
        return username
        
    except jwt.JWTError:
        return None


def revoke_refresh_token(refresh_token: str) -> bool:
    """Refresh token'ı iptal etme"""
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        jti = payload.get("jti")
        
        if jti in refresh_token_store:
            del refresh_token_store[jti]
            return True
            
        return False
        
    except jwt.JWTError:
        return False


@app.post("/token", response_model=TokenPair)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login endpoint - Token pair üretimi"""
    # Kullanıcı doğrulama (basitleştirilmiş)
    if form_data.username != "testuser" or form_data.password != "testpass":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password"
        )
    
    # Token pair oluşturma
    access_token = create_access_token(data={"sub": form_data.username})
    refresh_token = create_refresh_token(data={"sub": form_data.username})
    
    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }


@app.post("/token/refresh", response_model=TokenPair)
async def refresh_token_endpoint(request: RefreshTokenRequest):
    """Refresh token ile yeni token pair alma"""
    # Refresh token doğrulama
    username = verify_refresh_token(request.refresh_token)
    
    if not username:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid refresh token"
        )
    
    # Eski refresh token'ı iptal etme (rotation)
    revoke_refresh_token(request.refresh_token)
    
    # Yeni token pair oluşturma
    new_access_token = create_access_token(data={"sub": username})
    new_refresh_token = create_refresh_token(data={"sub": username})
    
    return {
        "access_token": new_access_token,
        "refresh_token": new_refresh_token,
        "token_type": "bearer"
    }


@app.post("/token/revoke")
async def revoke_token(request: RefreshTokenRequest):
    """Refresh token'ı iptal etme (logout)"""
    success = revoke_refresh_token(request.refresh_token)
    
    if not success:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Token revocation failed"
        )
    
    return {"message": "Token revoked successfully"}
```

### Token Yenileme Client Implementasyonu

```python
import requests
from datetime import datetime, timedelta
from typing import Optional
import threading
import time


class AutoRefreshAPIClient:
    """Otomatik token yenileme özellikli API client"""
    
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.access_token: Optional[str] = None
        self.refresh_token: Optional[str] = None
        self.token_expires_at: Optional[datetime] = None
        self._refresh_lock = threading.Lock()
    
    def login(self, username: str, password: str) -> bool:
        """Login ve token alma"""
        try:
            response = requests.post(
                f"{self.base_url}/token",
                data={"username": username, "password": password}
            )
            response.raise_for_status()
            
            self._update_tokens(response.json())
            
            # Otomatik yenileme thread'ini başlatma
            self._start_auto_refresh()
            
            return True
            
        except requests.RequestException as e:
            print(f"Login failed: {e}")
            return False
    
    def _update_tokens(self, token_data: dict):
        """Token bilgilerini güncelleme"""
        self.access_token = token_data["access_token"]
        self.refresh_token = token_data["refresh_token"]
        
        # Token expire süresini hesaplama (15 dakika - 1 dakika güvenlik marjı)
        self.token_expires_at = datetime.now() + timedelta(minutes=14)
    
    def _should_refresh(self) -> bool:
        """Token yenilenmeli mi kontrolü"""
        if not self.token_expires_at:
            return False
        
        # Token 2 dakika içinde expire olacaksa yenile
        return datetime.now() >= self.token_expires_at - timedelta(minutes=2)
    
    def _refresh_access_token(self) -> bool:
        """Access token'ı yenileme"""
        with self._refresh_lock:
            if not self.refresh_token:
                return False
            
            try:
                response = requests.post(
                    f"{self.base_url}/token/refresh",
                    json={"refresh_token": self.refresh_token}
                )
                response.raise_for_status()
                
                self._update_tokens(response.json())
                print("Token refreshed successfully")
                return True
                
            except requests.RequestException as e:
                print(f"Token refresh failed: {e}")
                return False
    
    def _start_auto_refresh(self):
        """Otomatik token yenileme thread'ini başlatma"""
        def refresh_loop():
            while self.refresh_token:
                if self._should_refresh():
                    self._refresh_access_token()
                time.sleep(60)  # Her dakika kontrol et
        
        refresh_thread = threading.Thread(target=refresh_loop, daemon=True)
        refresh_thread.start()
    
    def get_headers(self) -> dict:
        """Authorization header oluşturma"""
        # Gerekirse token'ı yenile
        if self._should_refresh():
            self._refresh_access_token()
        
        if not self.access_token:
            raise Exception("Not authenticated")
        
        return {"Authorization": f"Bearer {self.access_token}"}
    
    def make_request(self, method: str, endpoint: str, **kwargs) -> requests.Response:
        """Otomatik token yenileme ile HTTP request"""
        headers = kwargs.pop("headers", {})
        headers.update(self.get_headers())
        
        response = requests.request(
            method,
            f"{self.base_url}{endpoint}",
            headers=headers,
            **kwargs
        )
        
        # 401 hatası durumunda token yenileme ve yeniden deneme
        if response.status_code == 401:
            if self._refresh_access_token():
                headers.update(self.get_headers())
                response = requests.request(
                    method,
                    f"{self.base_url}{endpoint}",
                    headers=headers,
                    **kwargs
                )
        
        return response
    
    def logout(self):
        """Logout ve token iptali"""
        if self.refresh_token:
            try:
                requests.post(
                    f"{self.base_url}/token/revoke",
                    json={"refresh_token": self.refresh_token}
                )
            except requests.RequestException:
                pass
            
            self.access_token = None
            self.refresh_token = None
            self.token_expires_at = None


# Kullanım örneği
client = AutoRefreshAPIClient("http://localhost:8000")
client.login("testuser", "testpass")

# Otomatik token yenileme ile request
response = client.make_request("GET", "/protected-data")
print(response.json())
```

## OAuth2 ile Yetkilendirme

![OAuth2 Akış Rolleri](/assets/img/posts/api-oauth-flow-roles.png){: w="700" h="400" .shadow }
_OAuth2 authorization flow rolleri ve etkileşimleri_
_OAuth2 protokolünde rol ve etkileşimler_

OAuth2, üçüncü taraf uygulamaların kullanıcı kaynaklarına erişim yetkisi alması için tasarlanmış bir yetkilendirme framework'üdür.

### OAuth2 Authorization Code Flow

```python
from fastapi import FastAPI, Request, Depends, HTTPException
from fastapi.responses import RedirectResponse
from authlib.integrations.starlette_client import OAuth
from starlette.middleware.sessions import SessionMiddleware
import secrets

app = FastAPI()

# Session middleware ekleme
app.add_middleware(SessionMiddleware, secret_key=secrets.token_urlsafe(32))

# OAuth yapılandırması
oauth = OAuth()

# Google OAuth2 yapılandırması
oauth.register(
    name='google',
    client_id='YOUR_GOOGLE_CLIENT_ID',
    client_secret='YOUR_GOOGLE_CLIENT_SECRET',
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={
        'scope': 'openid email profile'
    }
)

# GitHub OAuth2 yapılandırması
oauth.register(
    name='github',
    client_id='YOUR_GITHUB_CLIENT_ID',
    client_secret='YOUR_GITHUB_CLIENT_SECRET',
    access_token_url='https://github.com/login/oauth/access_token',
    access_token_params=None,
    authorize_url='https://github.com/login/oauth/authorize',
    authorize_params=None,
    api_base_url='https://api.github.com/',
    client_kwargs={'scope': 'user:email'},
)


@app.get("/")
async def homepage(request: Request):
    """Ana sayfa"""
    user = request.session.get('user')
    if user:
        return {"message": f"Welcome {user['name']}!"}
    return {"message": "Not logged in"}


@app.get("/login/google")
async def login_google(request: Request):
    """Google OAuth2 login başlatma"""
    redirect_uri = request.url_for('auth_google')
    return await oauth.google.authorize_redirect(request, redirect_uri)


@app.get("/auth/google")
async def auth_google(request: Request):
    """Google OAuth2 callback"""
    try:
        # Token alma
        token = await oauth.google.authorize_access_token(request)
        
        # Kullanıcı bilgisi alma
        user_info = token.get('userinfo')
        
        if user_info:
            request.session['user'] = {
                'name': user_info.get('name'),
                'email': user_info.get('email'),
                'picture': user_info.get('picture')
            }
            
            return RedirectResponse(url='/')
        
        raise HTTPException(status_code=400, detail="Failed to get user info")
        
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))


@app.get("/login/github")
async def login_github(request: Request):
    """GitHub OAuth2 login başlatma"""
    redirect_uri = request.url_for('auth_github')
    return await oauth.github.authorize_redirect(request, redirect_uri)


@app.get("/auth/github")
async def auth_github(request: Request):
    """GitHub OAuth2 callback"""
    try:
        # Token alma
        token = await oauth.github.authorize_access_token(request)
        
        # Kullanıcı bilgisi alma
        resp = await oauth.github.get('user', token=token)
        user_info = resp.json()
        
        request.session['user'] = {
            'name': user_info.get('name') or user_info.get('login'),
            'email': user_info.get('email'),
            'avatar': user_info.get('avatar_url')
        }
        
        return RedirectResponse(url='/')
        
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))


@app.get("/logout")
async def logout(request: Request):
    """Logout"""
    request.session.clear()
    return RedirectResponse(url='/')


@app.get("/user")
async def get_user(request: Request):
    """Kullanıcı bilgisi"""
    user = request.session.get('user')
    if not user:
        raise HTTPException(status_code=401, detail="Not authenticated")
    return user
```

## API Key Authentication

Basit ve hızlı kimlik doğrulama için API key'ler kullanılabilir. Ancak, güvenlik açısından dikkatli kullanılmalıdır.

```python
from fastapi import FastAPI, Security, HTTPException, status
from fastapi.security import APIKeyHeader
from typing import Optional
import secrets
import hashlib
from datetime import datetime

app = FastAPI()

# API Key header tanımı
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

# API Key store (üretimde veritabanı kullanın)
api_keys_db = {
    "hashed_key_1": {
        "name": "Service A",
        "created_at": datetime.now(),
        "last_used": None,
        "rate_limit": 1000,
        "permissions": ["read", "write"]
    }
}


def hash_api_key(api_key: str) -> str:
    """API key hashleme"""
    return hashlib.sha256(api_key.encode()).hexdigest()


def generate_api_key() -> tuple[str, str]:
    """Yeni API key üretme"""
    # Güvenli random API key oluşturma
    api_key = secrets.token_urlsafe(32)
    
    # Hash'leme (database'de saklanacak)
    hashed_key = hash_api_key(api_key)
    
    return api_key, hashed_key


def verify_api_key(api_key: str) -> Optional[dict]:
    """API key doğrulama"""
    if not api_key:
        return None
    
    hashed_key = hash_api_key(api_key)
    
    if hashed_key in api_keys_db:
        # Son kullanım zamanını güncelleme
        api_keys_db[hashed_key]["last_used"] = datetime.now()
        return api_keys_db[hashed_key]
    
    return None


async def get_api_key(api_key: str = Security(api_key_header)):
    """API key dependency"""
    key_info = verify_api_key(api_key)
    
    if not key_info:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or missing API Key"
        )
    
    return key_info


@app.post("/api-keys")
async def create_api_key(name: str):
    """Yeni API key oluşturma"""
    # API key üretme
    api_key, hashed_key = generate_api_key()
    
    # Database'e kaydetme
    api_keys_db[hashed_key] = {
        "name": name,
        "created_at": datetime.now(),
        "last_used": None,
        "rate_limit": 1000,
        "permissions": ["read"]
    }
    
    return {
        "api_key": api_key,  # Sadece bir kez gösterilir
        "message": "Save this API key securely. It will not be shown again."
    }


@app.get("/protected")
async def protected_endpoint(key_info: dict = Depends(get_api_key)):
    """API key ile korunan endpoint"""
    return {
        "message": "Access granted",
        "service": key_info["name"],
        "permissions": key_info["permissions"]
    }


@app.delete("/api-keys/{key_name}")
async def revoke_api_key(key_name: str, key_info: dict = Depends(get_api_key)):
    """API key iptali"""
    # Admin permission kontrolü
    if "admin" not in key_info.get("permissions", []):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Insufficient permissions"
        )
    
    # Key'i bulup silme
    for hashed_key, info in list(api_keys_db.items()):
        if info["name"] == key_name:
            del api_keys_db[hashed_key]
            return {"message": f"API key '{key_name}' revoked"}
    
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="API key not found"
    )
```

## API Güvenliği En İyi Uygulamaları

![API Güvenlik En İyi Uygulamaları](/assets/img/posts/api-security-best-practices.jpg){: w="700" h="400" .shadow }
_API güvenlik en iyi uygulamaları özeti_
_API güvenliği için kritik uygulamalar_

### 1. HTTPS Kullanımı

```python
from fastapi import FastAPI, Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware

class HTTPSRedirectMiddleware(BaseHTTPMiddleware):
    """HTTP'den HTTPS'e yönlendirme middleware"""
    
    async def dispatch(self, request: Request, call_next):
        # Production ortamında HTTPS zorunluluğu
        if not request.url.scheme == "https":
            if request.app.state.env == "production":
                # HTTPS URL'ine yönlendirme
                url = request.url.replace(scheme="https")
                return RedirectResponse(url)
        
        response = await call_next(request)
        return response


app = FastAPI()
app.add_middleware(HTTPSRedirectMiddleware)
```

### 2. Rate Limiting (Hız Sınırlama)

```python
from fastapi import FastAPI, Request, HTTPException
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

# Rate limiter oluşturma
limiter = Limiter(key_func=get_remote_address)

app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)


@app.get("/api/data")
@limiter.limit("5/minute")  # Dakikada 5 request
async def get_data(request: Request):
    """Rate limit'li endpoint"""
    return {"data": "sensitive information"}


@app.get("/api/public")
@limiter.limit("100/hour")  # Saatte 100 request
async def get_public_data(request: Request):
    """Public endpoint - daha yüksek limit"""
    return {"data": "public information"}
```

### 3. Input Validation

```python
from pydantic import BaseModel, validator, EmailStr, constr
from typing import Optional
from fastapi import FastAPI, HTTPException

app = FastAPI()


class UserCreate(BaseModel):
    username: constr(min_length=3, max_length=50)  # Uzunluk kontrolü
    email: EmailStr  # Email validasyonu
    password: constr(min_length=8)  # Minimum şifre uzunluğu
    age: Optional[int] = None
    
    @validator('username')
    def username_alphanumeric(cls, v):
        """Username alfanumerik kontrolü"""
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v
    
    @validator('password')
    def password_strength(cls, v):
        """Şifre güçlülük kontrolü"""
        if not any(char.isdigit() for char in v):
            raise ValueError('Password must contain at least one digit')
        if not any(char.isupper() for char in v):
            raise ValueError('Password must contain at least one uppercase letter')
        return v
    
    @validator('age')
    def age_range(cls, v):
        """Yaş aralığı kontrolü"""
        if v is not None and (v < 0 or v > 150):
            raise ValueError('Age must be between 0 and 150')
        return v


@app.post("/users")
async def create_user(user: UserCreate):
    """Input validation ile kullanıcı oluşturma"""
    # Pydantic otomatik olarak validasyon yapar
    return {"message": "User created successfully", "username": user.username}
```

### 4. SQL Injection Koruması

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from fastapi import FastAPI, Depends

# SQLAlchemy setup
SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

app = FastAPI()


class User(Base):
    """User model"""
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)


def get_db():
    """Database session dependency"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@app.get("/users/{username}")
async def get_user(username: str, db: Session = Depends(get_db)):
    """
    Güvenli kullanıcı sorgusu - SQL injection'a karşı korumalı
    
    ❌ YANLIŞ: f"SELECT * FROM users WHERE username = '{username}'"
    DOĞRU: Parametreli sorgular kullanarak
    """
    # SQLAlchemy ORM otomatik olarak parametreli sorgu kullanır
    user = db.query(User).filter(User.username == username).first()
    
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    return {"id": user.id, "username": user.username, "email": user.email}
```

### 5. CORS Yapılandırması

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS middleware yapılandırması
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://yourdomain.com",  # Üretim domain'i
        "http://localhost:3000",   # Geliştirme ortamı
    ],
    allow_credentials=True,  # Cookie ve authorization header'ları
    allow_methods=["GET", "POST", "PUT", "DELETE"],  # İzin verilen HTTP methodları
    allow_headers=["*"],  # İzin verilen header'lar
    max_age=3600,  # Preflight cache süresi (saniye)
)


@app.get("/api/data")
async def get_data():
    """CORS korumalı endpoint"""
    return {"data": "cross-origin accessible"}
```

### 6. Audit Logging

```python
import logging
from datetime import datetime
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import json

# Audit logger yapılandırması
audit_logger = logging.getLogger("audit")
audit_logger.setLevel(logging.INFO)

# File handler
handler = logging.FileHandler("audit.log")
handler.setFormatter(logging.Formatter(
    '%(asctime)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
))
audit_logger.addHandler(handler)


class AuditMiddleware(BaseHTTPMiddleware):
    """API request audit logging middleware"""
    
    async def dispatch(self, request: Request, call_next):
        # Request bilgileri
        start_time = datetime.now()
        
        # Request body'yi okuma (eğer varsa)
        body = None
        if request.method in ["POST", "PUT", "PATCH"]:
            try:
                body = await request.body()
                body = body.decode()
            except:
                pass
        
        # Response'u alma
        response = await call_next(request)
        
        # Process time hesaplama
        process_time = (datetime.now() - start_time).total_seconds()
        
        # Audit log oluşturma
        audit_data = {
            "timestamp": start_time.isoformat(),
            "method": request.method,
            "path": request.url.path,
            "client_ip": request.client.host,
            "status_code": response.status_code,
            "process_time": process_time,
            "user": getattr(request.state, "user", None),  # Authenticated user
            "body": body[:200] if body else None  # İlk 200 karakter
        }
        
        # Hassas veri maskeleme
        if "password" in str(body).lower():
            audit_data["body"] = "[REDACTED]"
        
        # Log kaydetme
        audit_logger.info(json.dumps(audit_data))
        
        return response


app = FastAPI()
app.add_middleware(AuditMiddleware)
```

## Güvenlik Kontrol Listesi

API güvenliği için mutlaka uygulanması gereken kontrol listesi:

### Kimlik Doğrulama ve Yetkilendirme
- Güçlü kimlik doğrulama mekanizması kullanın (JWT, OAuth2)
- Şifreleri asla düz metin olarak saklamayın (bcrypt, argon2 kullanın)
- Token'ları güvenli şekilde saklayın (HTTP-only cookie'ler)
- Refresh token rotation uygulayın
- Role-based access control (RBAC) implementasyonu yapın

### İletişim Güvenliği
- Her zaman HTTPS kullanın
- TLS 1.2 veya üstü sürüm kullanın
- Certificate pinning kullanın (mobile app'lerde)
- HSTS header'ı ekleyin

### Input Validation
- Tüm kullanıcı girişlerini validate edin
- SQL injection'a karşı parametreli sorgular kullanın
- XSS saldırılarına karşı output encoding yapın
- File upload'larda tip ve boyut kontrolü yapın

### Rate Limiting ve DoS Koruması
- Endpoint bazlı rate limiting uygulayın
- IP bazlı rate limiting ekleyin
- Request size limitleri belirleyin
- Timeout değerlerini ayarlayın

### Logging ve Monitoring
- ✅ Tüm authentication attemptlerini loglayin
- ✅ Başarısız login denemelerini izleyin
- ✅ Hassas verileri log'lardan çıkarın
- ✅ Anomali tespit sistemi kurun

### API Key Güvenliği
- ✅ API key'leri asla kodda hardcode etmeyin
- ✅ Environment variable'ları kullanın
- ✅ API key'lere expire date ekleyin
- ✅ Key rotation policy oluşturun

## Sonuç

API kimlik doğrulama ve güvenlik, modern web uygulamalarının temel taşlarından biridir. JWT token'lar ile stateless authentication, OAuth2 ile üçüncü taraf entegrasyonlar ve refresh token'lar ile güvenli token yönetimi, production-ready API'ler için olmazsa olmazdır.

Bu yazıda ele aldığımız konular:

1. **JWT Authentication**: Stateless token tabanlı kimlik doğrulama
2. **Refresh Token Mechanism**: Güvenli token yenileme ve rotation
3. **OAuth2 Integration**: Google, GitHub gibi sağlayıcılarla entegrasyon
4. **API Key Management**: Basit servisler arası kimlik doğrulama
5. **Security Best Practices**: Rate limiting, input validation, HTTPS
6. **Audit Logging**: Güvenlik olaylarının izlenmesi

Unutmayın ki güvenlik bir süreçtir, tek seferlik bir implementasyon değildir. Sürekli güncel tutulması, test edilmesi ve iyileştirilmesi gereken bir konudur. Production ortamına geçmeden önce penetration testing yapın, security audit'leri düzenli olarak gerçekleştirin ve güvenlik best practice'lerini takip edin.

## Kaynaklar

- [FastAPI Security Documentation](https://fastapi.tiangolo.com/tutorial/security/)
- [JWT.io - JSON Web Tokens](https://jwt.io/)
- [OAuth 2.0 Specification](https://oauth.net/2/)
- [OWASP API Security Project](https://owasp.org/www-project-api-security/)
- [Python Jose Library](https://python-jose.readthedocs.io/)
