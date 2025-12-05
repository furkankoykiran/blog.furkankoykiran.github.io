---
title: "Poetry ile Python Proje Yönetimi"
description: "Poetry ile modern Python proje yönetimi. Bağımlılık yönetimi, virtual environment, paketleme ve PyPI'ye yayınlama."
date: "2024-12-28 09:00:00 +0300"
categories: [Development Tools, Python]
tags: [python, poetry, dependency-management, packaging, virtual-environment, pyproject-toml]
image:
  path: /assets/img/posts/poetry-dependency-management-workflow.png
  alt: "Poetry Dependency Management Workflow"
---

Python ekosisteminde bağımlılık yönetimi ve proje konfigürasyonu uzun yıllardır karmaşık bir konu olmuştur. Poetry, bu sorunu çözmek için geliştirilmiş modern bir araçtır ve Python projelerinde tek bir araçla dependency management, packaging ve virtual environment yönetimini sağlar.

## Poetry Neden Önemli?

![Python Environment Problem](/assets/img/posts/python-virtual-environment-xkcd.png)
*Python ortam yönetiminin karmaşıklığı (xkcd #1987)*

Geleneksel Python projelerinde karşılaşılan sorunlar:

- **setup.py, requirements.txt, setup.cfg, MANIFEST.in** gibi birden fazla konfigürasyon dosyası
- **pip** ile bağımlılık sürümlerinin tam olarak kilitlenememesi
- **Virtual environment** yönetiminin ayrı araçlarla yapılması
- **Dependency resolution** problemleri
- **Paket yayınlama** işleminin karmaşıklığı

### Poetry'nin Sunduğu Çözümler

```bash
# Geleneksel yaklaşım
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip freeze > requirements.txt

# Poetry yaklaşımı
poetry install
```

**Poetry'nin avantajları:**
- **Tek dosya**: Tüm konfigürasyon `pyproject.toml` dosyasında
- **Dependency resolution**: Otomatik ve güvenilir bağımlılık çözümleme
- **Lock file**: `poetry.lock` ile deterministik kurulumlar
- **Virtual environment**: Otomatik venv yönetimi
- **Publishing**: PyPI'a tek komutla paket yayınlama

## Poetry Kurulumu

### Resmi Installer (Önerilen)

```bash
# Linux, macOS, Windows (WSL)
curl -sSL https://install.python-poetry.org | python3 -

# Alternatif: wget ile
wget -qO- https://install.python-poetry.org | python3 -

# Windows (PowerShell)
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | py -
```

### PATH Konfigürasyonu

```bash
# Poetry binary'sini PATH'e ekleme
# Linux/macOS - ~/.bashrc veya ~/.zshrc
export PATH="$HOME/.local/bin:$PATH"

# Konfigürasyonu yenileme
source ~/.bashrc

# Poetry sürümünü kontrol etme
poetry --version
# Output: Poetry (version 1.7.1)
```

### Poetry Konfigürasyonu

```bash
# Virtual environment'ı proje dizininde oluşturma (önerilen)
poetry config virtualenvs.in-project true

# Mevcut konfigürasyonu görme
poetry config --list

# Output:
# cache-dir = "/home/user/.cache/pypoetry"
# virtualenvs.create = true
# virtualenvs.in-project = true
# virtualenvs.path = "{cache-dir}/virtualenvs"
```

## Yeni Proje Oluşturma

### Sıfırdan Proje Başlatma

```bash
# Yeni proje oluşturma
poetry new myproject

# Proje yapısı
myproject/
├── myproject/
│   └── __init__.py
├── tests/
│   └── __init__.py
├── pyproject.toml
└── README.md
```

### Mevcut Projeye Poetry Ekleme

```bash
# Mevcut dizinde Poetry başlatma
cd existing-project
poetry init

# İnteraktif olarak proje bilgilerini girme
# - Package name
# - Version
# - Description
# - Author
# - License
# - Dependencies
```

## pyproject.toml Dosyası

### Temel Yapı

```toml
[tool.poetry]
name = "myproject"
version = "0.1.0"
description = "A fantastic Python project"
authors = ["Your Name <you@example.com>"]
readme = "README.md"
license = "MIT"
homepage = "https://github.com/username/myproject"
repository = "https://github.com/username/myproject"
documentation = "https://myproject.readthedocs.io"
keywords = ["api", "web", "framework"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
]

# Python version constraint
[tool.poetry.dependencies]
python = "^3.9"
requests = "^2.31.0"
fastapi = "^0.104.1"
pydantic = "^2.5.0"

# Development dependencies
[tool.poetry.group.dev.dependencies]
pytest = "^7.4.3"
black = "^23.11.0"
flake8 = "^6.1.0"
mypy = "^1.7.1"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```
{: file="pyproject.toml" }

### Dependency Version Constraints

```toml
[tool.poetry.dependencies]
# Caret requirement (önerilen)
# ^1.2.3 := >=1.2.3 <2.0.0
requests = "^2.31.0"

# Tilde requirement
# ~1.2.3 := >=1.2.3 <1.3.0
flask = "~1.1.0"

# Wildcard requirement
# * := >=0.0.0
pytest = "*"

# Inequality requirement
django = ">=3.2,<5.0"

# Exact requirement
pillow = "==9.5.0"

# Multiple requirements
celery = ">=5.0,<6.0,!=5.1.0"

# Git dependencies
mypackage = { git = "https://github.com/user/repo.git", branch = "main" }

# Local path dependencies
mylocalpackage = { path = "../mylocalpackage", develop = true }

# Extras (optional dependencies)
sqlalchemy = { version = "^2.0", extras = ["asyncio"] }

# Python version specific
importlib-metadata = { version = "^6.0", python = "<3.10" }

# Platform specific
pywin32 = { version = "^306", platform = "win32" }
```
{: file="pyproject.toml" }

## Dependency Management

### Paket Ekleme ve Kaldırma

```bash
# Production dependency ekleme
poetry add requests
poetry add fastapi uvicorn

# Development dependency ekleme
poetry add --group dev pytest black flake8

# Belirli versiyon ekleme
poetry add "django==4.2.0"
poetry add "pandas>=1.5,<2.0"

# Git repository'den ekleme
poetry add git+https://github.com/user/repo.git

# Local package ekleme
poetry add ../my-local-package

# Optional dependency group oluşturma
poetry add --group docs sphinx sphinx-rtd-theme

# Paket kaldırma
poetry remove requests
poetry remove --group dev pytest
```

### Bağımlılıkları Güncelleme

```bash
# Tüm bağımlılıkları güncelleme
poetry update

# Belirli paketi güncelleme
poetry update requests

# Dry-run (sadece ne yapılacağını göster)
poetry update --dry-run

# Lock dosyasını güncelleme (kurulum yapmadan)
poetry lock

# Lock dosyasını kontrol etme
poetry check
```

### poetry.lock Dosyası

`poetry.lock` dosyası, tüm bağımlılıkların tam sürümlerini ve hash'lerini içerir:

```bash
# Lock dosyasından kurulum yapma (production'da)
poetry install --no-root

# Lock dosyasını yenileme
poetry lock --no-update

# Lock dosyasını silip yeniden oluşturma
rm poetry.lock
poetry lock
```

## Virtual Environment Yönetimi

![Poetry Workflow](/assets/img/posts/poetry-workflow-diagram.png)
*Poetry ile proje workflow'u*

### Virtual Environment Oluşturma ve Kullanma

```bash
# Virtual environment oluşturma ve dependencies kurma
poetry install

# Virtual environment'ı aktif etme
poetry shell

# Shell'den çıkış
exit

# Tek bir komut çalıştırma (venv'e girmeden)
poetry run python script.py
poetry run pytest
poetry run uvicorn app:app --reload
```

### Environment Bilgileri

```bash
# Virtual environment yolunu görme
poetry env info
poetry env info --path

# Output:
# Virtualenv
# Python:         3.11.6
# Implementation: CPython
# Path:           /home/user/myproject/.venv
# Executable:     /home/user/myproject/.venv/bin/python
# Valid:          True

# Kurulu paketleri listeleme
poetry show
poetry show --tree  # Dependency tree
poetry show --latest  # Güncel versiyonlarla karşılaştırma
poetry show --outdated  # Sadece eski paketleri göster

# Belirli bir paketin detaylarını görme
poetry show requests
```

### Multiple Environment Yönetimi

```bash
# Python versiyonu belirtme
poetry env use python3.9
poetry env use python3.11
poetry env use /usr/local/bin/python3.11

# Mevcut environment'ları listeleme
poetry env list
poetry env list --full-path

# Environment silme
poetry env remove python3.9
poetry env remove 3.11
poetry env remove --all
```

## Dependency Groups

### Group Tanımlama

```toml
[tool.poetry.group.dev.dependencies]
pytest = "^7.4.3"
black = "^23.11.0"
mypy = "^1.7.1"

[tool.poetry.group.docs.dependencies]
sphinx = "^7.2.6"
sphinx-rtd-theme = "^2.0.0"

[tool.poetry.group.test.dependencies]
pytest-cov = "^4.1.0"
pytest-asyncio = "^0.21.1"
faker = "^20.1.0"

# Optional group (varsayılan olarak kurulmaz)
[tool.poetry.group.ml]
optional = true

[tool.poetry.group.ml.dependencies]
tensorflow = "^2.15.0"
numpy = "^1.26.2"
```

### Group'ları Kurma

```bash
# Sadece production dependencies
poetry install --only main

# Specific groups
poetry install --with docs
poetry install --with docs,test

# Belirli group'ları hariç tutma
poetry install --without dev,docs

# Tüm optional groups
poetry install --all-extras

# Group ekleme/kaldırma
poetry add --group test pytest-mock
poetry remove --group docs sphinx
```

## Scripts ve Commands

### pyproject.toml'de Script Tanımlama

```toml
[tool.poetry.scripts]
# Basit command
myapp = "myproject.cli:main"

# Development scripts
dev = "myproject.server:run_dev"
test = "pytest"
lint = "flake8 myproject tests"
format = "black myproject tests"

[tool.poetry.dev-dependencies]
# Script dependencies
```

### Custom Scripts Oluşturma

```python
# myproject/cli.py
def main():
    """Ana CLI fonksiyonu"""
    print("Hello from myproject!")
    
if __name__ == "__main__":
    main()
```

```bash
# Script'i çalıştırma
poetry run myapp

# Birden fazla script
poetry run dev
poetry run test
poetry run lint
```

### Makefile ile Entegrasyon

```makefile
# Makefile
.PHONY: install test lint format clean

install:
	poetry install

test:
	poetry run pytest tests/ -v --cov=myproject

lint:
	poetry run flake8 myproject tests
	poetry run mypy myproject

format:
	poetry run black myproject tests
	poetry run isort myproject tests

clean:
	find . -type d -name "__pycache__" -exec rm -rf {} +
	find . -type f -name "*.pyc" -delete
	rm -rf .pytest_cache .mypy_cache .coverage htmlcov dist

dev:
	poetry run uvicorn myproject.main:app --reload --port 8000
```

## Paket Oluşturma ve Yayınlama

### Paket Build Etme

```bash
# Wheel ve source distribution oluşturma
poetry build

# Output:
# Building myproject (0.1.0)
#   - Building sdist
#   - Built myproject-0.1.0.tar.gz
#   - Building wheel
#   - Built myproject-0.1.0-py3-none-any.whl

# dist/ dizini
dist/
├── myproject-0.1.0.tar.gz
└── myproject-0.1.0-py3-none-any.whl
```

### PyPI'a Yayınlama

```bash
# PyPI credentials konfigürasyonu
poetry config pypi-token.pypi your-token-here

# Test PyPI'a yayınlama
poetry config repositories.testpypi https://test.pypi.org/legacy/
poetry publish -r testpypi --build

# Production PyPI'a yayınlama
poetry publish --build

# Username/password ile (alternatif)
poetry publish --username=your-username --password=your-password
```

### Paket Metadata

```toml
[tool.poetry]
name = "awesome-package"
version = "1.0.0"
description = "An awesome Python package"
authors = ["Your Name <you@example.com>"]
license = "MIT"
readme = "README.md"
homepage = "https://github.com/user/awesome-package"
repository = "https://github.com/user/awesome-package"
documentation = "https://awesome-package.readthedocs.io"
keywords = ["awesome", "package", "python"]

# PyPI classifiers
classifiers = [
    "Development Status :: 5 - Production/Stable",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Topic :: Software Development :: Libraries :: Python Modules",
]

# Include/exclude files
include = ["CHANGELOG.md", "LICENSE"]
exclude = ["tests/*", "docs/*"]

# Package data
packages = [
    { include = "mypackage" },
    { include = "mypackage/data", format = "sdist" }
]
```
{: file="pyproject.toml" }

## Gerçek Dünya Örneği: FastAPI Projesi

### Proje Yapısı

```bash
myapi/
├── myapi/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── user.py
│   ├── routes/
│   │   ├── __init__.py
│   │   └── users.py
│   ├── services/
│   │   ├── __init__.py
│   │   └── user_service.py
│   └── utils/
│       ├── __init__.py
│       └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_users.py
├── alembic/
│   ├── versions/
│   └── env.py
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── Makefile
├── pyproject.toml
├── poetry.lock
└── README.md
```

### pyproject.toml Konfigürasyonu

```toml
[tool.poetry]
name = "myapi"
version = "0.1.0"
description = "Modern FastAPI application"
authors = ["Developer <dev@example.com>"]
readme = "README.md"
license = "MIT"

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.1"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
pydantic = "^2.5.0"
pydantic-settings = "^2.1.0"
sqlalchemy = "^2.0.23"
alembic = "^1.13.0"
psycopg2-binary = "^2.9.9"
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
passlib = {extras = ["bcrypt"], version = "^1.7.4"}
python-multipart = "^0.0.6"
redis = "^5.0.1"
celery = "^5.3.4"
httpx = "^0.25.2"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.3"
pytest-asyncio = "^0.21.1"
pytest-cov = "^4.1.0"
black = "^23.11.0"
isort = "^5.12.0"
flake8 = "^6.1.0"
mypy = "^1.7.1"
pre-commit = "^3.5.0"
faker = "^20.1.0"

[tool.poetry.group.docs.dependencies]
mkdocs = "^1.5.3"
mkdocs-material = "^9.5.1"

[tool.poetry.scripts]
dev = "myapi.main:run_dev"
migrate = "alembic:upgrade"

[tool.black]
line-length = 100
target-version = ['py311']
include = '\.pyi?$'

[tool.isort]
profile = "black"
line_length = 100

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
asyncio_mode = "auto"

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```
{: file="pyproject.toml" }

### Ana Uygulama Dosyası

```python
# myapi/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from myapi.config import settings
from myapi.routes import users

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events"""
    # Startup
    print("Starting up...")
    yield
    # Shutdown
    print("Shutting down...")

app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    description="Modern FastAPI application with Poetry",
    lifespan=lifespan
)

# Include routers
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])

@app.get("/")
async def root():
    return {
        "message": "Welcome to MyAPI",
        "version": settings.VERSION,
        "docs": "/docs"
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy"}

def run_dev():
    """Development server runner"""
    import uvicorn
    uvicorn.run(
        "myapi.main:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
        log_level="info"
    )

if __name__ == "__main__":
    run_dev()
```
{: file="myapi/main.py" }

### Konfigürasyon Yönetimi

```python
# myapi/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    """Application settings"""
    PROJECT_NAME: str = "MyAPI"
    VERSION: str = "0.1.0"
    
    # Database
    DATABASE_URL: str = "postgresql://user:pass@localhost/myapi"
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"
    
    # Security
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    
    # CORS
    BACKEND_CORS_ORIGINS: list[str] = ["http://localhost:3000"]
    
    class Config:
        env_file = ".env"
        case_sensitive = True

@lru_cache()
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```
{: file="myapi/config.py" }

### Testing Setup

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from myapi.main import app
from myapi.models.base import Base

SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

@pytest.fixture(scope="session")
def db_engine():
    """Create test database engine"""
    engine = create_engine(
        SQLALCHEMY_DATABASE_URL,
        connect_args={"check_same_thread": False}
    )
    Base.metadata.create_all(bind=engine)
    yield engine
    Base.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def db_session(db_engine):
    """Create test database session"""
    TestingSessionLocal = sessionmaker(
        autocommit=False,
        autoflush=False,
        bind=db_engine
    )
    session = TestingSessionLocal()
    yield session
    session.close()

@pytest.fixture
def client(db_session):
    """Create test client"""
    def override_get_db():
        try:
            yield db_session
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as test_client:
        yield test_client

# tests/test_users.py
import pytest
from fastapi import status

def test_read_users(client):
    """Test users list endpoint"""
    response = client.get("/api/v1/users/")
    assert response.status_code == status.HTTP_200_OK
    assert isinstance(response.json(), list)

def test_create_user(client):
    """Test user creation"""
    user_data = {
        "email": "test@example.com",
        "username": "testuser",
        "password": "TestPass123!"
    }
    response = client.post("/api/v1/users/", json=user_data)
    assert response.status_code == status.HTTP_201_CREATED
    data = response.json()
    assert data["email"] == user_data["email"]
    assert "id" in data
```
{: file="tests/test_users.py" }

## Docker ile Entegrasyon

### Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Poetry kurulumu
RUN pip install poetry==1.7.1

# Konfigürasyon
ENV POETRY_NO_INTERACTION=1 \
    POETRY_VIRTUALENVS_IN_PROJECT=1 \
    POETRY_VIRTUALENVS_CREATE=1 \
    POETRY_CACHE_DIR=/tmp/poetry_cache

# Dependencies kurulumu
COPY pyproject.toml poetry.lock ./
RUN poetry install --without dev --no-root && rm -rf $POETRY_CACHE_DIR

# Uygulama kodunu kopyalama
COPY myapi ./myapi

# Root paketi kurma
RUN poetry install --without dev

EXPOSE 8000

# Uygulamayı çalıştırma
CMD ["poetry", "run", "uvicorn", "myapi.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
{: file="Dockerfile" }

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapi
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    volumes:
      - ./myapi:/app/myapi
    command: poetry run uvicorn myapi.main:app --host 0.0.0.0 --port 8000 --reload

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapi
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```
{: file="docker-compose.yml" }

## Poetry vs Diğer Araçlar

![Python Packaging Tools Comparison](/assets/img/posts/python-packaging-tools-comparison.png)
*Python paketleme araçları karşılaştırması*

### Poetry vs pip + venv

```bash
# pip + venv
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip freeze > requirements.txt

# Poetry
poetry install
```

**Poetry avantajları:**
- Dependency resolution (pip'te yok)
- Lock file ile deterministik builds
- Virtual environment otomatik yönetimi
- Tek komutla paket yayınlama

### Poetry vs Pipenv

| Özellik | Poetry | Pipenv |
|---------|--------|--------|
| Dependency Resolution | ✅ Hızlı ve güvenilir | ✅ Yavaş |
| Lock File | ✅ poetry.lock | ✅ Pipfile.lock |
| Build/Publish | ✅ Built-in | ❌ Yok |
| Performance | ✅ Hızlı | ⚠️ Yavaş |
| pyproject.toml | ✅ Native | ❌ Pipfile |
| Aktif Geliştirme | ✅ Çok aktif | ⚠️ Az aktif |

### Poetry vs pip-tools

```bash
# pip-tools
pip-compile requirements.in
pip-sync requirements.txt

# Poetry
poetry lock
poetry install
```

**Poetry avantajları:**
- Virtual environment yönetimi
- Daha iyi dependency resolution
- Paket yayınlama desteği
- Script management

## Best Practices

### 1. poetry.lock Dosyasını Commit Edin

```bash
# .gitignore
__pycache__/
*.py[cod]
.venv/
dist/
*.egg-info/

# poetry.lock'u COMMIT EDİN (ignore ETMEYİN!)
```

### 2. Dependency Groups Kullanın

```toml
# Production dependencies
[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.1"

# Development dependencies
[tool.poetry.group.dev.dependencies]
pytest = "^7.4.3"
black = "^23.11.0"

# Optional dependencies
[tool.poetry.group.ml]
optional = true

[tool.poetry.group.ml.dependencies]
tensorflow = "^2.15.0"
```

### 3. Version Constraints Dikkatli Kullanın

```toml
# ✅ İyi - Caret (^) kullanımı
fastapi = "^0.104.1"  # >=0.104.1 <0.105.0

# ⚠️ Dikkatli - Wildcard
requests = "*"  # Her zaman latest (riskli)

# ✅ İyi - Inequality constraints
django = ">=4.2,<5.0"

# ❌ Kötü - Exact version (gereksiz kısıtlama)
numpy = "==1.26.2"
```

### 4. Pre-commit Hooks Kullanın

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 23.11.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort

  - repo: https://github.com/pycqa/flake8
    rev: 6.1.0
    hooks:
      - id: flake8
```
{: file=".pre-commit-config.yaml" }

```bash
# Pre-commit kurulumu
poetry add --group dev pre-commit
poetry run pre-commit install

# Manuel çalıştırma
poetry run pre-commit run --all-files
```

### 5. CI/CD Pipeline'ında Poetry

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install Poetry
        uses: snok/install-poetry@v1
        with:
          version: 1.7.1
          virtualenvs-create: true
          virtualenvs-in-project: true
      
      - name: Load cached venv
        id: cached-poetry-dependencies
        uses: actions/cache@v3
        with:
          path: .venv
          key: venv-${{ runner.os }}-${{ steps.setup-python.outputs.python-version }}-${{ hashFiles('**/poetry.lock') }}
      
      - name: Install dependencies
        if: steps.cached-poetry-dependencies.outputs.cache-hit != 'true'
        run: poetry install --no-interaction --no-root
      
      - name: Install project
        run: poetry install --no-interaction
      
      - name: Run tests
        run: |
          poetry run pytest tests/ -v --cov=myproject --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```
{: file=".github/workflows/test.yml" }

## Sorun Giderme

### Lock File Çakışmaları

```bash
# Lock file çakışması varsa
git checkout --theirs poetry.lock
poetry lock --no-update

# Veya
rm poetry.lock
poetry install
```

### Dependency Resolution Hataları

```bash
# Verbose mode ile çalıştırma
poetry add package-name -vvv

# Cache temizleme
poetry cache clear pypi --all

# Lock dosyasını yeniden oluşturma
rm poetry.lock
poetry lock
```

### Virtual Environment Sorunları

```bash
# Environment'ı yeniden oluşturma
poetry env remove python3.11
poetry install

# Environment bilgilerini kontrol etme
poetry env info
poetry env list

# Manuel environment silme
rm -rf .venv
poetry install
```

## Sonuç

Poetry, modern Python projelerinde dependency management ve packaging için vazgeçilmez bir araçtır. Bu yazıda öğrendiklerimiz:

- **Kurulum ve Konfigürasyon**: Poetry'nin sistem kurulumu ve temel ayarları
- **pyproject.toml**: Tek dosyada tüm proje konfigürasyonu
- **Dependency Management**: Paket ekleme, güncelleme ve version constraints
- **Virtual Environments**: Otomatik venv yönetimi ve multiple environments
- **Packaging**: Paket build etme ve PyPI'a yayınlama
- **Real-World**: FastAPI projesi ile gerçek dünya örneği
- **Best Practices**: Pre-commit hooks, CI/CD entegrasyonu

### Önemli Noktalar

1. **poetry.lock** dosyasını mutlaka commit edin
2. **Dependency groups** kullanarak farklı ortamlar için bağımlılıkları ayırın
3. **Caret constraints** (^) kullanarak esnek version yönetimi yapın
4. **Pre-commit hooks** ile kod kalitesini otomatikleştirin
5. **CI/CD pipeline**'da Poetry cache'ini kullanın

### Kaynaklar

- [Poetry Documentation](https://python-poetry.org/docs/)
- [pyproject.toml Specification](https://peps.python.org/pep-0518/)
- [Python Packaging Guide](https://packaging.python.org/)

Bir sonraki yazımızda **MongoDB ile NoSQL Veri Yönetimi** konusunu işleyeceğiz!
