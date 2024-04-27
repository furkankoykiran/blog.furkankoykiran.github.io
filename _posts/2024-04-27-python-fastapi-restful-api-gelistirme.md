---
title: "Python FastAPI ile RESTful API Geliştirme"
date: "2024-04-27"
categories:
  - "backend"
  - "python"
tags:
  - "fastapi"
  - "rest-api"
  - "python"
  - "async"
  - "web-development"
  - "api-design"
  - "backend"
image:
  src: "/assets/img/posts/fastapi-microservices-architecture.png"
  alt: "FastAPI Microservices Architecture"
---

Modern web uygulamalarında performans ve geliştirici deneyimi kritik öneme sahip. FastAPI, Python ekosistemindeki en hızlı ve kullanımı en kolay web framework'lerinden biri olarak öne çıkıyor. Bu kapsamlı rehberde, FastAPI ile production-ready RESTful API'ler nasıl geliştirileceğini öğreneceksiniz.

## FastAPI Nedir?

FastAPI, Python 3.6+ için modern, hızlı (yüksek performanslı) bir web framework'üdür. Standart Python tip bildirimleri (type hints) üzerine inşa edilmiş olup, otomatik API dokümantasyonu, veri validasyonu ve yüksek performans sunar.

### Neden FastAPI?

- **Hızlı**: NodeJS ve Go ile yarışacak performans
- **Hızlı Geliştirme**: Kod yazma hızını %200-300 artırır
- **Otomatik Dokümantasyon**: Swagger UI ve ReDoc entegre
- **Tip Güvenliği**: Pydantic ile otomatik veri validasyonu
- **Async Destek**: Modern async/await syntax
- **Production Ready**: Production ortamı için hazır

## Kurulum ve Başlangıç

### Gerekli Paketler

```bash
# Temel kurulum
pip install fastapi
pip install "uvicorn[standard]"

# Geliştirme için ek paketler
pip install pydantic[email]
pip install python-multipart
pip install python-jose[cryptography]
pip install passlib[bcrypt]
pip install sqlalchemy
pip install alembic
```

### İlk API

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional, List

app = FastAPI(
    title="Blog API",
    description="Profesyonel blog yönetimi için API",
    version="1.0.0"
)

class Post(BaseModel):
    id: Optional[int] = None
    title: str
    content: str
    published: bool = True
    author_id: int

@app.get("/")
async def root():
    """Ana sayfa endpoint'i"""
    return {
        "message": "Blog API'ye hoş geldiniz",
        "version": "1.0.0",
        "docs": "/docs"
    }

@app.get("/posts", response_model=List[Post])
async def get_posts(skip: int = 0, limit: int = 10):
    """Tüm postları listele"""
    # Database query buraya gelecek
    return []

@app.get("/posts/{post_id}", response_model=Post)
async def get_post(post_id: int):
    """Belirli bir postu getir"""
    return {
        "id": post_id,
        "title": "FastAPI ile Geliştirme",
        "content": "FastAPI harika bir framework",
        "published": True,
        "author_id": 1
    }

@app.post("/posts", response_model=Post, status_code=201)
async def create_post(post: Post):
    """Yeni post oluştur"""
    post.id = 1  # Database'den gelecek
    return post
```

### Uygulamayı Çalıştırma

```bash
# Development server
uvicorn main:app --reload

# Production server
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

# HTTPS ile
uvicorn main:app --ssl-keyfile=./key.pem --ssl-certfile=./cert.pem
```

## Pydantic ile Veri Modelleme

Pydantic, FastAPI'nin kalbidir. Otomatik veri validasyonu ve serialization sağlar.

### Temel Modeller

```python
from pydantic import BaseModel, Field, EmailStr, HttpUrl, validator
from typing import Optional, List
from datetime import datetime
from enum import Enum

class UserRole(str, Enum):
    ADMIN = "admin"
    AUTHOR = "author"
    READER = "reader"

class UserBase(BaseModel):
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50)
    full_name: Optional[str] = Field(None, max_length=100)
    role: UserRole = UserRole.READER
    
    class Config:
        json_schema_extra = {
            "example": {
                "email": "user@example.com",
                "username": "johndoe",
                "full_name": "John Doe",
                "role": "reader"
            }
        }

class UserCreate(UserBase):
    password: str = Field(..., min_length=8, max_length=100)
    
    @validator('password')
    def password_strength(cls, v):
        if not any(char.isdigit() for char in v):
            raise ValueError('Şifre en az bir rakam içermelidir')
        if not any(char.isupper() for char in v):
            raise ValueError('Şifre en az bir büyük harf içermelidir')
        return v

class UserInDB(UserBase):
    id: int
    hashed_password: str
    created_at: datetime
    updated_at: Optional[datetime] = None
    is_active: bool = True
    
    class Config:
        from_attributes = True

class UserResponse(UserBase):
    id: int
    created_at: datetime
    is_active: bool

class PostBase(BaseModel):
    title: str = Field(..., min_length=5, max_length=200)
    content: str = Field(..., min_length=10)
    published: bool = True
    tags: List[str] = []
    cover_image: Optional[HttpUrl] = None

class PostCreate(PostBase):
    pass

class PostUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=5, max_length=200)
    content: Optional[str] = Field(None, min_length=10)
    published: Optional[bool] = None
    tags: Optional[List[str]] = None

class PostInDB(PostBase):
    id: int
    author_id: int
    created_at: datetime
    updated_at: Optional[datetime] = None
    views: int = 0
    
    class Config:
        from_attributes = True

class PostResponse(PostInDB):
    author: UserResponse
```

## Database Entegrasyonu

### SQLAlchemy ile ORM

```python
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship
from datetime import datetime

SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/blogdb"
# SQLite için: "sqlite:///./blog.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    pool_pre_ping=True,
    pool_size=10,
    max_overflow=20
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Database modelleri
class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    username = Column(String, unique=True, index=True, nullable=False)
    full_name = Column(String, nullable=True)
    hashed_password = Column(String, nullable=False)
    role = Column(String, default="reader")
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, nullable=True)
    
    # İlişkiler
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    comments = relationship("Comment", back_populates="user", cascade="all, delete-orphan")

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False, index=True)
    content = Column(Text, nullable=False)
    published = Column(Boolean, default=True)
    views = Column(Integer, default=0)
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, nullable=True, onupdate=datetime.utcnow)
    
    # İlişkiler
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")
    tags = relationship("Tag", secondary="post_tags", back_populates="posts")

class Comment(Base):
    __tablename__ = "comments"
    
    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text, nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id"), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    user = relationship("User", back_populates="comments")
    post = relationship("Post", back_populates="comments")

class Tag(Base):
    __tablename__ = "tags"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), unique=True, nullable=False)
    
    posts = relationship("Post", secondary="post_tags", back_populates="tags")

# Database bağımlılığı
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### CRUD İşlemleri

```python
from sqlalchemy.orm import Session
from typing import List, Optional
from fastapi import HTTPException, status

class PostRepository:
    """Post CRUD işlemleri"""
    
    @staticmethod
    def get_all(db: Session, skip: int = 0, limit: int = 100) -> List[Post]:
        """Tüm postları getir"""
        return db.query(Post).filter(Post.published == True).offset(skip).limit(limit).all()
    
    @staticmethod
    def get_by_id(db: Session, post_id: int) -> Optional[Post]:
        """ID'ye göre post getir"""
        return db.query(Post).filter(Post.id == post_id).first()
    
    @staticmethod
    def create(db: Session, post: PostCreate, author_id: int) -> Post:
        """Yeni post oluştur"""
        db_post = Post(
            **post.dict(),
            author_id=author_id
        )
        db.add(db_post)
        db.commit()
        db.refresh(db_post)
        return db_post
    
    @staticmethod
    def update(db: Session, post_id: int, post_update: PostUpdate, user_id: int) -> Post:
        """Post güncelle"""
        db_post = db.query(Post).filter(Post.id == post_id).first()
        
        if not db_post:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Post bulunamadı"
            )
        
        if db_post.author_id != user_id:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Bu işlem için yetkiniz yok"
            )
        
        # Sadece verilen alanları güncelle
        update_data = post_update.dict(exclude_unset=True)
        for key, value in update_data.items():
            setattr(db_post, key, value)
        
        db_post.updated_at = datetime.utcnow()
        db.commit()
        db.refresh(db_post)
        return db_post
    
    @staticmethod
    def delete(db: Session, post_id: int, user_id: int) -> bool:
        """Post sil"""
        db_post = db.query(Post).filter(Post.id == post_id).first()
        
        if not db_post:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Post bulunamadı"
            )
        
        if db_post.author_id != user_id:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Bu işlem için yetkiniz yok"
            )
        
        db.delete(db_post)
        db.commit()
        return True
    
    @staticmethod
    def search(db: Session, query: str, skip: int = 0, limit: int = 100) -> List[Post]:
        """Postlarda arama yap"""
        return db.query(Post).filter(
            (Post.title.ilike(f"%{query}%")) | (Post.content.ilike(f"%{query}%"))
        ).filter(Post.published == True).offset(skip).limit(limit).all()
```

## Router'lar ile API Organizasyonu

```python
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List

# Posts router
posts_router = APIRouter(
    prefix="/posts",
    tags=["posts"],
    responses={404: {"description": "Bulunamadı"}}
)

@posts_router.get("/", response_model=List[PostResponse])
async def list_posts(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    search: Optional[str] = None,
    db: Session = Depends(get_db)
):
    """Tüm postları listele"""
    if search:
        posts = PostRepository.search(db, search, skip, limit)
    else:
        posts = PostRepository.get_all(db, skip, limit)
    return posts

@posts_router.get("/{post_id}", response_model=PostResponse)
async def get_post(
    post_id: int,
    db: Session = Depends(get_db)
):
    """Belirli bir postu getir"""
    post = PostRepository.get_by_id(db, post_id)
    if not post:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Post ID {post_id} bulunamadı"
        )
    
    # View sayısını artır
    post.views += 1
    db.commit()
    
    return post

@posts_router.post("/", response_model=PostResponse, status_code=status.HTTP_201_CREATED)
async def create_post(
    post: PostCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Yeni post oluştur"""
    return PostRepository.create(db, post, current_user.id)

@posts_router.put("/{post_id}", response_model=PostResponse)
async def update_post(
    post_id: int,
    post_update: PostUpdate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Post güncelle"""
    return PostRepository.update(db, post_id, post_update, current_user.id)

@posts_router.delete("/{post_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_post(
    post_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Post sil"""
    PostRepository.delete(db, post_id, current_user.id)
    return None

# Users router
users_router = APIRouter(
    prefix="/users",
    tags=["users"]
)

@users_router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def register_user(
    user: UserCreate,
    db: Session = Depends(get_db)
):
    """Yeni kullanıcı kaydı"""
    # Email kontrolü
    db_user = db.query(User).filter(User.email == user.email).first()
    if db_user:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Bu email zaten kullanılıyor"
        )
    
    # Username kontrolü
    db_user = db.query(User).filter(User.username == user.username).first()
    if db_user:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Bu kullanıcı adı zaten kullanılıyor"
        )
    
    # Şifreyi hashle
    hashed_password = get_password_hash(user.password)
    
    # Kullanıcı oluştur
    db_user = User(
        email=user.email,
        username=user.username,
        full_name=user.full_name,
        hashed_password=hashed_password,
        role=user.role
    )
    
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    
    return db_user

# Ana uygulamaya router'ları ekle
app.include_router(posts_router)
app.include_router(users_router)
```

## Authentication ve Authorization

### JWT Token ile Authentication

```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

# Konfigürasyon
SECRET_KEY = "your-secret-key-keep-it-secret"  # Ortam değişkeninden al
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: Optional[str] = None

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Şifre doğrulama"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """Şifre hashleme"""
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """JWT token oluştur"""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return encoded_jwt

def authenticate_user(db: Session, username: str, password: str) -> Optional[User]:
    """Kullanıcı doğrulama"""
    user = db.query(User).filter(User.username == username).first()
    if not user:
        return None
    if not verify_password(password, user.hashed_password):
        return None
    return user

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    """Mevcut kullanıcıyı getir"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Kimlik doğrulaması yapılamadı",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except JWTError:
        raise credentials_exception
    
    user = db.query(User).filter(User.username == token_data.username).first()
    if user is None:
        raise credentials_exception
    
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Kullanıcı aktif değil"
        )
    
    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """Aktif kullanıcıyı getir"""
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

def require_role(required_role: str):
    """Role tabanlı yetkilendirme decorator'ı"""
    async def role_checker(current_user: User = Depends(get_current_active_user)):
        if current_user.role != required_role and current_user.role != "admin":
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Bu işlem için yetkiniz yok"
            )
        return current_user
    return role_checker

# Login endpoint
@app.post("/token", response_model=Token)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """Kullanıcı girişi ve token alma"""
    user = authenticate_user(db, form_data.username, form_data.password)
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Kullanıcı adı veya şifre hatalı",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=access_token_expires
    )
    
    return {"access_token": access_token, "token_type": "bearer"}

# Korumalı endpoint örneği
@app.get("/users/me", response_model=UserResponse)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    """Mevcut kullanıcı bilgilerini getir"""
    return current_user
```

## Dosya Upload ve İşleme

```python
from fastapi import File, UploadFile
from typing import List
import shutil
from pathlib import Path
import uuid

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)

@app.post("/upload/")
async def upload_file(file: UploadFile = File(...)):
    """Tek dosya yükleme"""
    # Güvenli dosya adı oluştur
    file_extension = Path(file.filename).suffix
    safe_filename = f"{uuid.uuid4()}{file_extension}"
    file_path = UPLOAD_DIR / safe_filename
    
    # Dosyayı kaydet
    with file_path.open("wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    
    return {
        "filename": safe_filename,
        "content_type": file.content_type,
        "size": file_path.stat().st_size
    }

@app.post("/upload/multiple/")
async def upload_multiple_files(files: List[UploadFile] = File(...)):
    """Çoklu dosya yükleme"""
    uploaded_files = []
    
    for file in files:
        file_extension = Path(file.filename).suffix
        safe_filename = f"{uuid.uuid4()}{file_extension}"
        file_path = UPLOAD_DIR / safe_filename
        
        with file_path.open("wb") as buffer:
            shutil.copyfileobj(file.file, buffer)
        
        uploaded_files.append({
            "filename": safe_filename,
            "original_name": file.filename,
            "size": file_path.stat().st_size
        })
    
    return {"uploaded_files": uploaded_files}

@app.post("/posts/{post_id}/image")
async def upload_post_image(
    post_id: int,
    file: UploadFile = File(...),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_active_user)
):
    """Post için resim yükleme"""
    # Post kontrolü
    post = PostRepository.get_by_id(db, post_id)
    if not post:
        raise HTTPException(status_code=404, detail="Post bulunamadı")
    
    if post.author_id != current_user.id:
        raise HTTPException(status_code=403, detail="Yetkiniz yok")
    
    # Resim formatı kontrolü
    allowed_types = ["image/jpeg", "image/png", "image/gif", "image/webp"]
    if file.content_type not in allowed_types:
        raise HTTPException(
            status_code=400,
            detail=f"Sadece şu formatlar desteklenir: {', '.join(allowed_types)}"
        )
    
    # Dosya boyutu kontrolü (5MB)
    file.file.seek(0, 2)
    file_size = file.file.tell()
    file.file.seek(0)
    
    if file_size > 5 * 1024 * 1024:
        raise HTTPException(
            status_code=400,
            detail="Dosya boyutu 5MB'dan büyük olamaz"
        )
    
    # Dosyayı kaydet
    file_extension = Path(file.filename).suffix
    safe_filename = f"post_{post_id}_{uuid.uuid4()}{file_extension}"
    file_path = UPLOAD_DIR / safe_filename
    
    with file_path.open("wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    
    return {
        "image_url": f"/uploads/{safe_filename}",
        "filename": safe_filename
    }
```

## Background Tasks

```python
from fastapi import BackgroundTasks
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

def send_email(email: str, subject: str, body: str):
    """Email gönderme (background task)"""
    # Email gönderme kodu
    print(f"Email gönderildi: {email}")

def generate_report(user_id: int, report_type: str):
    """Rapor oluşturma (background task)"""
    import time
    time.sleep(5)  # Simüle edilmiş işlem
    print(f"Rapor oluşturuldu: {report_type} for user {user_id}")

@app.post("/posts/{post_id}/publish")
async def publish_post(
    post_id: int,
    background_tasks: BackgroundTasks,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_active_user)
):
    """Post yayınla ve takipçilere bildirim gönder"""
    post = PostRepository.get_by_id(db, post_id)
    
    if not post:
        raise HTTPException(status_code=404, detail="Post bulunamadı")
    
    if post.author_id != current_user.id:
        raise HTTPException(status_code=403, detail="Yetkiniz yok")
    
    # Postu yayınla
    post.published = True
    db.commit()
    
    # Background task olarak email gönder
    background_tasks.add_task(
        send_email,
        current_user.email,
        "Post Yayınlandı",
        f"'{post.title}' başlıklı postunuz yayınlandı!"
    )
    
    return {"message": "Post yayınlandı", "post_id": post_id}
```

## Middleware ve CORS

```python
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.gzip import GZipMiddleware
import time
import logging

# CORS ayarları
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://yourdomain.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Güvenlik için trusted host
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "*.yourdomain.com"]
)

# Response compression
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Custom middleware - Request timing
@app.middleware("http")
async def add_process_time_header(request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response

# Custom middleware - Request logging
logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request, call_next):
    logger.info(f"Request: {request.method} {request.url}")
    response = await call_next(request)
    logger.info(f"Response status: {response.status_code}")
    return response
```

## Exception Handling

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from sqlalchemy.exc import SQLAlchemyError

class CustomException(Exception):
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code

@app.exception_handler(CustomException)
async def custom_exception_handler(request: Request, exc: CustomException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"message": exc.message}
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "message": "Veri doğrulama hatası",
            "errors": exc.errors()
        }
    )

@app.exception_handler(SQLAlchemyError)
async def sqlalchemy_exception_handler(request: Request, exc: SQLAlchemyError):
    logger.error(f"Database error: {str(exc)}")
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"message": "Veritabanı hatası oluştu"}
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {str(exc)}")
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"message": "Sunucu hatası oluştu"}
    )
```

## Testing

```python
from fastapi.testclient import TestClient
import pytest

client = TestClient(app)

def test_read_main():
    response = client.get("/")
    assert response.status_code == 200
    assert "message" in response.json()

def test_create_post():
    response = client.post(
        "/posts/",
        json={
            "title": "Test Post",
            "content": "Test content for the post",
            "published": True
        },
        headers={"Authorization": "Bearer fake-token"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Test Post"
    assert "id" in data

def test_get_post():
    response = client.get("/posts/1")
    assert response.status_code == 200
    data = response.json()
    assert "title" in data

@pytest.fixture
def test_user():
    response = client.post(
        "/users/register",
        json={
            "email": "test@example.com",
            "username": "testuser",
            "password": "Test123!",
            "role": "author"
        }
    )
    return response.json()

def test_login(test_user):
    response = client.post(
        "/token",
        data={
            "username": "testuser",
            "password": "Test123!"
        }
    )
    assert response.status_code == 200
    assert "access_token" in response.json()
```

## Production Deployment

### Docker ile Deploy

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code
COPY . .

# Run migrations
RUN alembic upgrade head

# Expose port
EXPOSE 8000

# Start application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/blogdb
      - SECRET_KEY=your-secret-key
    depends_on:
      - db
    restart: always
  
  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=blogdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always

volumes:
  postgres_data:
```

### Nginx ile Reverse Proxy

```nginx
# /etc/nginx/sites-available/fastapi
server {
    listen 80;
    server_name yourdomain.com;
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location /static {
        alias /var/www/static;
    }
    
    location /uploads {
        alias /var/www/uploads;
    }
}
```

## En İyi Pratikler

### 1. Proje Yapısı

```
myproject/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── posts.py
│   ├── services/
│   │   ├── __init__.py
│   │   └── auth.py
│   ├── repositories/
│   │   ├── __init__.py
│   │   └── post.py
│   └── utils/
│       ├── __init__.py
│       └── security.py
├── tests/
├── alembic/
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### 2. Konfigürasyon Yönetimi

```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    app_name: str = "Blog API"
    admin_email: str
    database_url: str
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    
    class Config:
        env_file = ".env"

@lru_cache()
def get_settings():
    return Settings()

settings = get_settings()
```

### 3. Dependency Injection

```python
from typing import Generator

def common_parameters(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    sort_by: str = Query("created_at"),
    order: str = Query("desc", regex="^(asc|desc)$")
):
    return {"skip": skip, "limit": limit, "sort_by": sort_by, "order": order}

@app.get("/posts/")
async def list_posts(
    commons: dict = Depends(common_parameters),
    db: Session = Depends(get_db)
):
    return PostRepository.get_all(db, **commons)
```

## Sonuç

FastAPI, modern Python web geliştirme için güçlü ve esnek bir framework'tür. Otomatik dokümantasyon, tip güvenliği, yüksek performans ve kolay kullanım gibi özellikleri sayesinde API geliştirme sürecini hızlandırır ve kod kalitesini artırır.

## Kaynaklar

- [FastAPI Resmi Dokümantasyonu](https://fastapi.tiangolo.com/)
- [Pydantic Dokümantasyonu](https://docs.pydantic.dev/)
- [SQLAlchemy Dokümantasyonu](https://docs.sqlalchemy.org/)
- [Uvicorn Dokümantasyonu](https://www.uvicorn.org/)

Başarılı projeler! 🚀
