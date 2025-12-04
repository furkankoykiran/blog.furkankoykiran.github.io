---
title: "MySQL ve SQLAlchemy ile Veritabanı Yönetimi"
description: "SQLAlchemy ORM ile MySQL veritabanı yönetimi. Model tanımlama, ilişkiler, migration, query optimization ve best practices."
date: "2024-12-22 09:00:00 +0300"
categories: [Database, Python]
tags: [python, mysql, sqlalchemy, orm, veritabanı, alembic, database-management]
image:
  path: /assets/img/posts/sqlalchemy-orm-class-diagram.png
  alt: "SQLAlchemy ORM Architecture"
---

Modern Python uygulamalarında veritabanı yönetimi, uygulamanın başarısı için kritik öneme sahiptir. SQLAlchemy, Python ekosistemindeki en güçlü ve esnek ORM (Object-Relational Mapping) kütüphanesidir. Bu yazıda, MySQL ile SQLAlchemy kullanarak profesyonel veritabanı yönetimi tekniklerini detaylı şekilde inceleyeceğiz.

## SQLAlchemy Nedir?

SQLAlchemy, Python için geliştirilmiş, SQL veritabanları ile çalışmak için iki farklı yaklaşım sunan bir toolkit'tir:

- **Core**: SQL ifadelerini Python objelerine dönüştüren düşük seviye API
- **ORM**: Veritabanı tablolarını Python sınıflarına map eden yüksek seviye API

### SQLAlchemy'nin Avantajları

```python
# Geleneksel SQL yaklaşımı
cursor.execute("""
    SELECT users.name, posts.title 
    FROM users 
    JOIN posts ON users.id = posts.user_id 
    WHERE users.age > 18
""")

# SQLAlchemy ORM yaklaşımı
session.query(User.name, Post.title)\
    .join(Post)\
    .filter(User.age > 18)\
    .all()
```

**Avantajlar:**
- **Type Safety**: Python tiplerini veritabanı tiplerine otomatik dönüştürme
- **Database Agnostic**: Kod değiştirmeden farklı veritabanları kullanabilme
- **Migration Support**: Alembic ile veritabanı şema değişikliklerini yönetme
- **Connection Pooling**: Otomatik bağlantı havuzu yönetimi
- **Lazy Loading**: İhtiyaç anında veri yükleme

## MySQL ve SQLAlchemy Kurulumu

### Gerekli Paketlerin Yüklenmesi

```bash
# SQLAlchemy ve MySQL driver
pip install sqlalchemy pymysql

# Alembic (migrations için)
pip install alembic

# Async desteği için (opsiyonel)
pip install asyncio aiomysql
```

### MySQL Bağlantı Kurulumu

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# MySQL bağlantı URL'i oluşturma
# Format: mysql+pymysql://user:password@host:port/database
DATABASE_URL = "mysql+pymysql://root:password@localhost:3306/myapp"

# Engine oluşturma
engine = create_engine(
    DATABASE_URL,
    echo=True,  # SQL loglarını göster
    pool_size=10,  # Bağlantı havuzu boyutu
    max_overflow=20,  # Maksimum taşma bağlantı sayısı
    pool_pre_ping=True,  # Bağlantı kontrolü
    pool_recycle=3600,  # 1 saatte bir bağlantıları yenile
)

# Session factory oluşturma
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)
```

## Model Tanımlama ve İlişkiler

### Temel Model Yapısı

```python
from sqlalchemy import Column, Integer, String, DateTime, Text, Boolean
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    """Kullanıcı modeli"""
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(100), unique=True, nullable=False, index=True)
    password_hash = Column(String(255), nullable=False)
    full_name = Column(String(100))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    def __repr__(self):
        return f"<User(username='{self.username}', email='{self.email}')>"
    
    def to_dict(self):
        """Model'i dictionary'ye dönüştürme"""
        return {
            "id": self.id,
            "username": self.username,
            "email": self.email,
            "full_name": self.full_name,
            "is_active": self.is_active,
            "created_at": self.created_at.isoformat() if self.created_at else None,
        }
```

### One-to-Many İlişkisi

![Database Relationships](/assets/img/posts/database-relationships-one-to-many.png)
*Bir kullanıcının birden fazla postu olabilir*

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class Post(Base):
    """Blog post modeli"""
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    slug = Column(String(200), unique=True, nullable=False, index=True)
    content = Column(Text, nullable=False)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    is_published = Column(Boolean, default=False)
    view_count = Column(Integer, default=0)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # İlişki tanımlama (back_populates kullanarak çift yönlü)
    user = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")
    tags = relationship("Tag", secondary="post_tags", back_populates="posts")
    
    def __repr__(self):
        return f"<Post(title='{self.title}', user_id={self.user_id})>"

# User modeline eklenmesi gereken ilişki
User.posts = relationship("Post", back_populates="user", cascade="all, delete-orphan")
```

### Many-to-Many İlişkisi

```python
from sqlalchemy import Table

# Association table (ara tablo)
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", Integer, ForeignKey("posts.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id", ondelete="CASCADE"), primary_key=True),
    Column("created_at", DateTime, default=datetime.utcnow)
)

class Tag(Base):
    """Tag modeli"""
    __tablename__ = "tags"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), unique=True, nullable=False, index=True)
    slug = Column(String(50), unique=True, nullable=False)
    description = Column(Text)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Many-to-Many ilişkisi
    posts = relationship("Post", secondary=post_tags, back_populates="tags")
    
    def __repr__(self):
        return f"<Tag(name='{self.name}')>"

class Comment(Base):
    """Yorum modeli"""
    __tablename__ = "comments"
    
    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text, nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id", ondelete="CASCADE"), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    parent_id = Column(Integer, ForeignKey("comments.id", ondelete="CASCADE"), nullable=True)
    is_approved = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # İlişkiler
    post = relationship("Post", back_populates="comments")
    user = relationship("User")
    
    # Self-referential ilişki (nested comments)
    parent = relationship("Comment", remote_side=[id], backref="replies")
    
    def __repr__(self):
        return f"<Comment(post_id={self.post_id}, user_id={self.user_id})>"
```

## CRUD İşlemleri

### Create (Oluşturma)

```python
def create_user(db, username: str, email: str, password: str):
    """Yeni kullanıcı oluşturma"""
    # Password hashing (gerçek uygulamada bcrypt kullanın)
    from hashlib import sha256
    password_hash = sha256(password.encode()).hexdigest()
    
    # Yeni user objesi oluşturma
    new_user = User(
        username=username,
        email=email,
        password_hash=password_hash,
        full_name=username.title()
    )
    
    try:
        db.add(new_user)
        db.commit()
        db.refresh(new_user)  # ID'yi almak için
        return new_user
    except Exception as e:
        db.rollback()
        raise e

def create_post_with_tags(db, user_id: int, title: str, content: str, tag_names: list):
    """Tag'lerle birlikte post oluşturma"""
    # Post oluşturma
    new_post = Post(
        title=title,
        slug=title.lower().replace(" ", "-"),
        content=content,
        user_id=user_id,
        is_published=True
    )
    
    # Tag'leri bulma veya oluşturma
    for tag_name in tag_names:
        tag = db.query(Tag).filter(Tag.name == tag_name).first()
        if not tag:
            tag = Tag(
                name=tag_name,
                slug=tag_name.lower().replace(" ", "-")
            )
            db.add(tag)
        new_post.tags.append(tag)
    
    try:
        db.add(new_post)
        db.commit()
        db.refresh(new_post)
        return new_post
    except Exception as e:
        db.rollback()
        raise e

# Kullanım
db = SessionLocal()
try:
    user = create_user(db, "john_doe", "john@example.com", "secret123")
    post = create_post_with_tags(
        db, 
        user.id, 
        "SQLAlchemy Tutorial",
        "This is a comprehensive tutorial...",
        ["python", "database", "orm"]
    )
    print(f"Post created: {post.title} with {len(post.tags)} tags")
finally:
    db.close()
```

### Read (Okuma)

```python
def get_user_by_id(db, user_id: int):
    """ID ile kullanıcı bulma"""
    return db.query(User).filter(User.id == user_id).first()

def get_user_by_username(db, username: str):
    """Username ile kullanıcı bulma"""
    return db.query(User).filter(User.username == username).first()

def get_active_users(db, skip: int = 0, limit: int = 100):
    """Aktif kullanıcıları listeleme (pagination)"""
    return db.query(User)\
        .filter(User.is_active == True)\
        .offset(skip)\
        .limit(limit)\
        .all()

def get_published_posts_with_tags(db, skip: int = 0, limit: int = 20):
    """Tag'leriyle birlikte yayınlanmış postları getirme"""
    return db.query(Post)\
        .filter(Post.is_published == True)\
        .options(
            joinedload(Post.user),  # Eager loading
            joinedload(Post.tags)
        )\
        .order_by(Post.created_at.desc())\
        .offset(skip)\
        .limit(limit)\
        .all()

def search_posts(db, keyword: str):
    """Post'larda arama yapma"""
    search_term = f"%{keyword}%"
    return db.query(Post)\
        .filter(
            (Post.title.like(search_term)) | 
            (Post.content.like(search_term))
        )\
        .filter(Post.is_published == True)\
        .all()

# Gelişmiş sorgular
from sqlalchemy import func

def get_user_post_counts(db):
    """Kullanıcıların post sayılarını getirme"""
    return db.query(
        User.username,
        func.count(Post.id).label("post_count")
    )\
        .join(Post)\
        .group_by(User.username)\
        .order_by(func.count(Post.id).desc())\
        .all()

def get_popular_tags(db, limit: int = 10):
    """En popüler tag'leri getirme"""
    return db.query(
        Tag.name,
        func.count(post_tags.c.post_id).label("usage_count")
    )\
        .join(post_tags)\
        .group_by(Tag.name)\
        .order_by(func.count(post_tags.c.post_id).desc())\
        .limit(limit)\
        .all()
```

### Update (Güncelleme)

```python
def update_user(db, user_id: int, **kwargs):
    """Kullanıcı bilgilerini güncelleme"""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        return None
    
    # Sadece verilen alanları güncelleme
    for key, value in kwargs.items():
        if hasattr(user, key):
            setattr(user, key, value)
    
    try:
        db.commit()
        db.refresh(user)
        return user
    except Exception as e:
        db.rollback()
        raise e

def increment_post_view_count(db, post_id: int):
    """Post görüntülenme sayısını artırma"""
    db.query(Post)\
        .filter(Post.id == post_id)\
        .update({"view_count": Post.view_count + 1})
    db.commit()

def publish_post(db, post_id: int):
    """Post'u yayınlama"""
    post = db.query(Post).filter(Post.id == post_id).first()
    if post:
        post.is_published = True
        db.commit()
        db.refresh(post)
        return post
    return None

# Kullanım
db = SessionLocal()
try:
    # Kullanıcı güncelleme
    updated_user = update_user(db, 1, full_name="John Doe Updated", is_active=True)
    
    # View count artırma
    increment_post_view_count(db, 1)
finally:
    db.close()
```

### Delete (Silme)

```python
def delete_user(db, user_id: int):
    """Kullanıcıyı silme (cascade ile postları da silinir)"""
    user = db.query(User).filter(User.id == user_id).first()
    if user:
        db.delete(user)
        db.commit()
        return True
    return False

def delete_post(db, post_id: int):
    """Post silme"""
    post = db.query(Post).filter(Post.id == post_id).first()
    if post:
        db.delete(post)
        db.commit()
        return True
    return False

def soft_delete_user(db, user_id: int):
    """Soft delete (kullanıcıyı deaktif etme)"""
    return update_user(db, user_id, is_active=False)

def bulk_delete_old_posts(db, days: int = 365):
    """Belirli günden eski postları toplu silme"""
    from datetime import timedelta
    cutoff_date = datetime.utcnow() - timedelta(days=days)
    
    deleted_count = db.query(Post)\
        .filter(Post.created_at < cutoff_date)\
        .filter(Post.is_published == False)\
        .delete()
    
    db.commit()
    return deleted_count
```

## Alembic ile Database Migrations

![Alembic Migration Workflow](/assets/img/posts/alembic-migration-workflow.png)
*Alembic veritabanı migration workflow'u*

### Alembic Kurulumu

```bash
# Alembic başlatma
alembic init alembic

# Klasör yapısı
# alembic/
#   ├── versions/
#   ├── env.py
#   ├── script.py.mako
#   └── README
# alembic.ini
```

### Alembic Konfigürasyonu

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
import os
import sys

# Projenin ana dizinini Python path'ine ekleme
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

# Model'lerinizi import edin
from models import Base  # Tüm model'lerinizin Base'i

# Alembic Config objesi
config = context.config

# Database URL'i environment variable'dan alma
config.set_main_option(
    'sqlalchemy.url',
    os.getenv('DATABASE_URL', 'mysql+pymysql://root:password@localhost:3306/myapp')
)

# Logging konfigürasyonu
fileConfig(config.config_file_name)

# Metadata
target_metadata = Base.metadata

def run_migrations_offline():
    """Offline migrations (SQL script olarak)"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    """Online migrations (veritabanına direkt uygulama)"""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Migration Oluşturma ve Uygulama

```bash
# İlk migration oluşturma (auto-generate)
alembic revision --autogenerate -m "Initial migration"

# Migration dosyası manuel oluşturma
alembic revision -m "Add view_count to posts"

# Migrations uygulama
alembic upgrade head

# Son migration'ı geri alma
alembic downgrade -1

# Belirli bir revision'a geri dönme
alembic downgrade <revision_id>

# Migration geçmişini görme
alembic history

# Mevcut revision'ı görme
alembic current
```

### Örnek Migration Dosyası

```python
# alembic/versions/001_add_view_count.py
"""Add view_count to posts

Revision ID: abc123def456
Revises: 
Create Date: 2024-12-22 09:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = 'abc123def456'
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    """Upgrade schema"""
    # view_count kolonu ekleme
    op.add_column('posts', 
        sa.Column('view_count', sa.Integer(), nullable=False, server_default='0')
    )
    
    # Index ekleme
    op.create_index('ix_posts_view_count', 'posts', ['view_count'])
    
    # Check constraint ekleme
    op.create_check_constraint(
        'check_view_count_positive',
        'posts',
        'view_count >= 0'
    )

def downgrade():
    """Downgrade schema"""
    op.drop_constraint('check_view_count_positive', 'posts')
    op.drop_index('ix_posts_view_count', 'posts')
    op.drop_column('posts', 'view_count')
```

## Connection Pooling ve Performans

![Connection Pool Architecture](/assets/img/posts/database-connection-pool-architecture.png)
*Database connection pool mimarisi*

### Connection Pool Konfigürasyonu

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# Gelişmiş pool ayarları
engine = create_engine(
    DATABASE_URL,
    
    # Pool tipi (QueuePool, NullPool, StaticPool, SingletonThreadPool)
    poolclass=QueuePool,
    
    # Pool boyutu (aktif bağlantı sayısı)
    pool_size=10,
    
    # Maksimum taşma (gerektiğinde ekstra bağlantılar)
    max_overflow=20,
    
    # Bağlantı timeout (saniye)
    pool_timeout=30,
    
    # Bağlantı yenileme süresi (saniye) - MySQL'in wait_timeout'undan küçük olmalı
    pool_recycle=3600,
    
    # Bağlantı kontrolü (her kullanımda ping)
    pool_pre_ping=True,
    
    # Echo SQL statements
    echo=False,
    
    # Connection arguments
    connect_args={
        "charset": "utf8mb4",
        "connect_timeout": 10,
    }
)

# Pool istatistiklerini görme
def get_pool_stats(engine):
    """Connection pool istatistikleri"""
    pool = engine.pool
    return {
        "size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "total": pool.size() + pool.overflow()
    }

# Kullanım
print(get_pool_stats(engine))
# Output: {'size': 10, 'checked_in': 8, 'checked_out': 2, 'overflow': 0, 'total': 10}
```

### Context Manager ile Güvenli Session Yönetimi

```python
from contextlib import contextmanager

@contextmanager
def get_db_session():
    """Context manager ile session yönetimi"""
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception as e:
        session.rollback()
        raise e
    finally:
        session.close()

# Kullanım
with get_db_session() as db:
    user = create_user(db, "jane_doe", "jane@example.com", "password123")
    print(f"User created: {user.username}")
```

### Dependency Injection (FastAPI ile)

```python
from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy.orm import Session

app = FastAPI()

def get_db():
    """Dependency injection için database session"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/users/")
def create_user_endpoint(
    username: str,
    email: str,
    password: str,
    db: Session = Depends(get_db)
):
    """User oluşturma endpoint'i"""
    try:
        user = create_user(db, username, email, password)
        return user.to_dict()
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.get("/users/{user_id}")
def get_user_endpoint(user_id: int, db: Session = Depends(get_db)):
    """User getirme endpoint'i"""
    user = get_user_by_id(db, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user.to_dict()
```

## Query Optimizasyonu

### Eager Loading vs Lazy Loading

```python
from sqlalchemy.orm import joinedload, subqueryload, selectinload

# N+1 Problem (BAD - Lazy Loading)
users = db.query(User).all()
for user in users:
    print(user.posts)  # Her user için ayrı sorgu!

# Eager Loading ile çözüm (GOOD)
users = db.query(User).options(joinedload(User.posts)).all()
for user in users:
    print(user.posts)  # Tek sorguda tüm data

# Farklı eager loading stratejileri
# 1. joinedload - LEFT OUTER JOIN ile
posts = db.query(Post)\
    .options(joinedload(Post.user))\
    .options(joinedload(Post.tags))\
    .all()

# 2. subqueryload - Ayrı subquery ile
posts = db.query(Post)\
    .options(subqueryload(Post.comments))\
    .all()

# 3. selectinload - IN clause ile (SQLAlchemy 1.2+)
posts = db.query(Post)\
    .options(selectinload(Post.tags))\
    .all()
```

### Indexleme ve Performans

```python
from sqlalchemy import Index

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    slug = Column(String(200), unique=True, nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"))
    is_published = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Composite index
    __table_args__ = (
        Index('ix_post_user_published', 'user_id', 'is_published'),
        Index('ix_post_created', 'created_at', postgresql_using='btree'),
    )

# Query planlama ve analiz
from sqlalchemy import text

def explain_query(db, query):
    """Query execution plan"""
    sql = str(query.statement.compile(
        dialect=db.bind.dialect,
        compile_kwargs={"literal_binds": True}
    ))
    result = db.execute(text(f"EXPLAIN {sql}"))
    for row in result:
        print(row)

# Kullanım
query = db.query(Post).filter(Post.is_published == True)
explain_query(db, query)
```

### Batch Operations

```python
def bulk_create_users(db, users_data: list):
    """Toplu kullanıcı oluşturma"""
    users = [User(**user_data) for user_data in users_data]
    db.bulk_save_objects(users)
    db.commit()

def bulk_update_posts(db, post_updates: list):
    """Toplu post güncelleme"""
    # post_updates = [{"id": 1, "view_count": 100}, {"id": 2, "view_count": 200}]
    db.bulk_update_mappings(Post, post_updates)
    db.commit()

# Daha hızlı insert (identity değerlerini döndürmez)
db.bulk_insert_mappings(
    User,
    [
        {"username": "user1", "email": "user1@example.com"},
        {"username": "user2", "email": "user2@example.com"},
    ]
)
db.commit()
```

## İleri Seviye Teknikler

### Custom Query Methods

```python
class UserRepository:
    """Repository pattern ile database işlemleri"""
    
    def __init__(self, db: Session):
        self.db = db
    
    def get_by_id(self, user_id: int):
        return self.db.query(User).filter(User.id == user_id).first()
    
    def get_by_username(self, username: str):
        return self.db.query(User).filter(User.username == username).first()
    
    def get_active_users(self, skip: int = 0, limit: int = 100):
        return self.db.query(User)\
            .filter(User.is_active == True)\
            .offset(skip)\
            .limit(limit)\
            .all()
    
    def search(self, keyword: str):
        search_term = f"%{keyword}%"
        return self.db.query(User)\
            .filter(
                (User.username.like(search_term)) |
                (User.email.like(search_term)) |
                (User.full_name.like(search_term))
            )\
            .all()
    
    def create(self, user_data: dict):
        user = User(**user_data)
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user
    
    def update(self, user_id: int, user_data: dict):
        user = self.get_by_id(user_id)
        if user:
            for key, value in user_data.items():
                setattr(user, key, value)
            self.db.commit()
            self.db.refresh(user)
        return user
    
    def delete(self, user_id: int):
        user = self.get_by_id(user_id)
        if user:
            self.db.delete(user)
            self.db.commit()
            return True
        return False

# Kullanım
db = SessionLocal()
user_repo = UserRepository(db)

user = user_repo.create({
    "username": "alice",
    "email": "alice@example.com",
    "password_hash": "hashed_password"
})

users = user_repo.search("alice")
```

### Transaction Management

```python
from sqlalchemy.exc import SQLAlchemyError

def transfer_post_ownership(db, post_id: int, new_owner_id: int):
    """Transaction ile post sahipliğini transfer etme"""
    try:
        # Transaction başlat
        post = db.query(Post).filter(Post.id == post_id).first()
        if not post:
            raise ValueError("Post not found")
        
        new_owner = db.query(User).filter(User.id == new_owner_id).first()
        if not new_owner:
            raise ValueError("New owner not found")
        
        # Değişiklikleri yap
        old_owner_id = post.user_id
        post.user_id = new_owner_id
        
        # Log kaydı ekle
        log_entry = AuditLog(
            action="transfer_ownership",
            post_id=post_id,
            old_value=str(old_owner_id),
            new_value=str(new_owner_id)
        )
        db.add(log_entry)
        
        # Tüm değişiklikleri commit et
        db.commit()
        return post
        
    except (SQLAlchemyError, ValueError) as e:
        # Hata durumunda rollback
        db.rollback()
        raise e

# Nested transactions
from sqlalchemy import event

@event.listens_for(Session, "after_transaction_end")
def receive_after_transaction_end(session, transaction):
    """Transaction sonrası hook"""
    print(f"Transaction ended: {transaction}")
```

### Hybrid Properties ve Expressions

```python
from sqlalchemy.ext.hybrid import hybrid_property, hybrid_method
from sqlalchemy import func

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    first_name = Column(String(50))
    last_name = Column(String(50))
    created_at = Column(DateTime, default=datetime.utcnow)
    
    @hybrid_property
    def full_name(self):
        """Instance seviyesinde çalışır"""
        return f"{self.first_name} {self.last_name}"
    
    @full_name.expression
    def full_name(cls):
        """SQL seviyesinde çalışır"""
        return func.concat(cls.first_name, ' ', cls.last_name)
    
    @hybrid_method
    def is_older_than(self, days: int):
        """Instance method"""
        from datetime import timedelta
        cutoff = datetime.utcnow() - timedelta(days=days)
        return self.created_at < cutoff
    
    @is_older_than.expression
    def is_older_than(cls, days: int):
        """SQL expression"""
        from datetime import timedelta
        cutoff = datetime.utcnow() - timedelta(days=days)
        return cls.created_at < cutoff

# Kullanım
# Instance seviyesinde
user = db.query(User).first()
print(user.full_name)  # "John Doe"

# Query seviyesinde
users = db.query(User).filter(User.full_name == "John Doe").all()
old_users = db.query(User).filter(User.is_older_than(365)).all()
```

## Best Practices ve Güvenlik

### SQL Injection Koruması

```python
# YANLIŞ - SQL Injection riski!
username = "admin' OR '1'='1"
query = f"SELECT * FROM users WHERE username = '{username}'"
db.execute(query)

# DOĞRU - Parameterized queries
username = "admin"
user = db.query(User).filter(User.username == username).first()

# Raw SQL gerekiyorsa text() kullanın
from sqlalchemy import text

username = "admin"
result = db.execute(
    text("SELECT * FROM users WHERE username = :username"),
    {"username": username}
)
```

### Connection String Güvenliği

```python
import os
from urllib.parse import quote_plus

# Environment variables kullanma
DB_USER = os.getenv("DB_USER", "root")
DB_PASSWORD = quote_plus(os.getenv("DB_PASSWORD", ""))
DB_HOST = os.getenv("DB_HOST", "localhost")
DB_PORT = os.getenv("DB_PORT", "3306")
DB_NAME = os.getenv("DB_NAME", "myapp")

DATABASE_URL = f"mysql+pymysql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"

# .env dosyası kullanma (python-dotenv)
from dotenv import load_dotenv
load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")
```

### Session Lifecycle Management

```python
# FastAPI ile session lifecycle
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session

app = FastAPI()

# Startup event
@app.on_event("startup")
def startup_event():
    """Uygulama başlangıcında"""
    # Tabloları oluştur
    Base.metadata.create_all(bind=engine)
    print("Database tables created")

# Shutdown event
@app.on_event("shutdown")
def shutdown_event():
    """Uygulama kapanırken"""
    engine.dispose()
    print("Database connections closed")

# Request sonrası cleanup
@app.middleware("http")
async def db_session_middleware(request, call_next):
    """Her request için session yönetimi"""
    response = None
    try:
        response = await call_next(request)
    finally:
        # Session cleanup
        pass
    return response
```

## Hata Yönetimi ve Logging

```python
import logging
from sqlalchemy.exc import IntegrityError, OperationalError, SQLAlchemyError

# Logging konfigürasyonu
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_create_user(db, username: str, email: str, password: str):
    """Hata yönetimi ile kullanıcı oluşturma"""
    try:
        user = create_user(db, username, email, password)
        logger.info(f"User created successfully: {username}")
        return user
        
    except IntegrityError as e:
        # Unique constraint violation
        db.rollback()
        logger.error(f"User already exists: {username}")
        raise ValueError("Username or email already exists")
        
    except OperationalError as e:
        # Database connection error
        db.rollback()
        logger.error(f"Database connection error: {e}")
        raise ConnectionError("Database is unavailable")
        
    except SQLAlchemyError as e:
        # Diğer SQLAlchemy hataları
        db.rollback()
        logger.error(f"Database error: {e}")
        raise
        
    except Exception as e:
        # Beklenmeyen hatalar
        db.rollback()
        logger.exception(f"Unexpected error: {e}")
        raise

# Query logging
from sqlalchemy import event
from sqlalchemy.engine import Engine

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    """SQL query logging"""
    logger.debug(f"Query: {statement}")
    logger.debug(f"Parameters: {parameters}")
```

## Sonuç

SQLAlchemy ve MySQL kombinasyonu, Python uygulamalarında güçlü ve esnek bir veritabanı yönetim çözümü sunar. Bu yazıda öğrendiklerimiz:

- **ORM Temelleri**: Model tanımlama, ilişkiler ve CRUD işlemleri
- **Alembic Migrations**: Veritabanı şema değişikliklerini yönetme
- **Connection Pooling**: Performans optimizasyonu ve kaynak yönetimi
- **Query Optimization**: Eager loading, indexleme ve batch operations
- **İleri Seviye**: Repository pattern, transactions ve hybrid properties
- **Best Practices**: Güvenlik, hata yönetimi ve logging

### Önemli Noktalar

1. **Connection pooling** ayarlarını production ortamına göre optimize edin
2. **Migration** dosyalarını versiyon kontrolünde tutun
3. **Eager loading** kullanarak N+1 probleminden kaçının
4. **Index**'leri düzgün kullanarak query performansını artırın
5. **Transaction** yönetimini doğru yaparak veri tutarlılığını sağlayın

### Kaynaklar

- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [Alembic Documentation](https://alembic.sqlalchemy.org/)
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [FastAPI with SQLAlchemy](https://fastapi.tiangolo.com/tutorial/sql-databases/)

Bir sonraki yazımızda **Poetry ile Python Proje Yönetimi** konusunu işleyeceğiz!
