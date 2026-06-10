---
title: "【まなびクエスト開発日記】Django + DRFでバックエンドAPIを構築した"
emoji: "🐍"
type: "tech"
topics: ["django", "python", "docker", "postgresql", "drf"]
published: true
---

# はじめに

小学生向け学習RPG「まなびクエスト」を開発しています。

前回はPhaser.jsでランダムエンカウントとバトル画面を実装しました。

https://zenn.dev/naokisatoojiyan/articles/manabi-quest-battle

今回はDjango + Django REST Framework（DRF）でバックエンドAPIを構築した内容をまとめます。

## 今回作るAPI

| エンドポイント | メソッド | 内容 |
|--------------|---------|------|
| `/api/register/` | POST | ユーザー登録 |
| `/api/login/` | POST | ログイン・トークン発行 |
| `/api/player/` | GET / PUT | ステータス取得・更新 |
| `/api/player/create/` | POST | プレイヤー作成 |

---

# 環境

- Python 3.14
- Django 5.2
- Django REST Framework 3.16.0
- PostgreSQL 16
- Docker / Docker Compose
- psycopg3（psycopg[binary] 3.3.4）

---

# STEP9: プロジェクト作成

## フォルダ構成

```
~/app/
├── manabi-quest/        ← フロントエンド（Phaser.js）
└── manabi-quest-api/    ← バックエンド（Django）← 今回作る
```

## ファイル作成

```bash
mkdir ~/app/manabi-quest-api
cd ~/app/manabi-quest-api
touch Dockerfile docker-compose.yml requirements.txt .env .env.example .gitignore
```

## requirements.txt

```
Django==5.2
djangorestframework==3.16.0
psycopg[binary]==3.3.4
django-cors-headers==4.7.0
python-dotenv==1.1.0
pytest-django==4.11.0
```

### psycopg3を選んだ理由

psycopg2はPython 3.14に非対応のため、psycopg3を採用しました。

```
psycopg2 → Python 3.13まで対応
psycopg3 → Python 3.14対応・非同期処理も得意
```

## .gitignore

```
.env
__pycache__/
*.pyc
*.pyo
.DS_Store
*.zip
db.sqlite3
```

## .env.example

```
DEBUG=True
SECRET_KEY=your-secret-key-here
DB_NAME=manabi_quest
DB_USER=postgres
DB_PASSWORD=your-db-password
DB_HOST=db
DB_PORT=5432
ALLOWED_HOSTS=localhost,127.0.0.1
CORS_ALLOWED_ORIGINS=http://127.0.0.1:5500,http://localhost:5500
```

## Dockerfile

```dockerfile
FROM python:3.14-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

## docker-compose.yml

```yaml
services:
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    environment:
      - DEBUG=${DEBUG}
      - SECRET_KEY=${SECRET_KEY}
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - ALLOWED_HOSTS=${ALLOWED_HOSTS}
      - CORS_ALLOWED_ORIGINS=${CORS_ALLOWED_ORIGINS}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### venvが不要な理由

```
venv → Macでプロジェクトごとに環境を分ける
Docker → コンテナ自体が独立した環境

→ Dockerを使う場合はvenvは不要
```

## Djangoプロジェクト作成

```bash
docker compose run --rm web django-admin startproject config .
```

---

# STEP10: settings.py の設定

```python
from pathlib import Path
import os
from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.environ['SECRET_KEY']

DEBUG = os.environ.get('DEBUG', 'False') == 'True'

ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', 'localhost').split(',')

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework.authtoken',
    'corsheaders',
    'players.apps.PlayersConfig',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',  # CORSはSessionの直後
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ['DB_NAME'],
        'USER': os.environ['DB_USER'],
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': os.environ['DB_HOST'],
        'PORT': os.environ['DB_PORT'],
    }
}

LANGUAGE_CODE = 'ja'
TIME_ZONE = 'Asia/Tokyo'
USE_I18N = True
USE_TZ = True

STATIC_URL = 'static/'
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# カスタムユーザーモデル
AUTH_USER_MODEL = 'players.User'

# CORS設定（フロントエンドからのリクエストを許可）
CORS_ALLOWED_ORIGINS = os.environ.get('CORS_ALLOWED_ORIGINS', '').split(',')

# DRF設定
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

### CORSとは

```
フロントエンド（localhost:5500）からバックエンド（localhost:8000）に
リクエストを送るとブラウザがブロックする（ドメインが違うため）

→ django-cors-headers で許可するドメインを設定して解決
```

---

# STEP11: モデルの設計

## なぜカスタムユーザーモデルが必要か

Djangoのデフォルトは `username` でログインしますが、今回は `email` でログインしたいためカスタムユーザーモデルを作成します。

:::message
**現場の鉄則：カスタムユーザーモデルはプロジェクト最初に作る**
途中から変更するのは非常に難しいため、最初に設計するのが重要です。
:::

## players/models.py

```python
from django.db import models
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin


class UserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('メールアドレスは必須です')
        email = self.normalize_email(email)  # 大文字→小文字に統一
        user = self.model(email=email, **extra_fields)
        user.set_password(password)  # パスワードをハッシュ化
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        return self.create_user(email, password, **extra_fields)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)  # ログインID（一意）
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    date_joined = models.DateTimeField(auto_now_add=True)

    objects = UserManager()

    USERNAME_FIELD = 'email'  # ログインにemailを使う
    REQUIRED_FIELDS = []

    def __str__(self):
        return self.email


class PlayerStatus(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    character_name = models.CharField(max_length=20)  # キャラクター名（重複OK）
    level = models.IntegerField(default=1)
    hp = models.IntegerField(default=10)
    max_hp = models.IntegerField(default=10)
    exp = models.IntegerField(default=0)
    attack = models.IntegerField(default=5)
    defense = models.IntegerField(default=5)
    agility = models.IntegerField(default=5)
    luck = models.IntegerField(default=5)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f'{self.character_name} - Lv.{self.level}'
```

### テーブルの関係

```
Userテーブル                  PlayerStatusテーブル
─────────────                 ────────────────────
id: 1                         id: 1
email: naoki@example.com  ←── user_id: 1（OneToOneField）
password: ***（ハッシュ化）    character_name: 勇者ナオキ
                              level: 5
                              attack: 8
```

### キャラクター名をUserテーブルに入れない理由

```
username（ログインID）→ 一意である必要がある
character_name      → 重複OK・後から自由に変更できる

→ PlayerStatusに持たせる方が設計として正しい
```

---

# STEP12: Serializer・View・URL

## players/serializers.py

```python
from rest_framework import serializers
from django.contrib.auth import get_user_model
from .models import PlayerStatus

User = get_user_model()


class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)  # レスポンスに含めない

    class Meta:
        model = User
        fields = ['email', 'password']

    def create(self, validated_data):
        return User.objects.create_user(
            email=validated_data['email'],
            password=validated_data['password']
        )


class PlayerStatusSerializer(serializers.ModelSerializer):
    class Meta:
        model = PlayerStatus
        fields = [
            'character_name', 'level', 'hp', 'max_hp', 'exp',
            'attack', 'defense', 'agility', 'luck',
            'created_at', 'updated_at'
        ]
        read_only_fields = ['created_at', 'updated_at']
```

## players/views.py

```python
from rest_framework import status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated, AllowAny
from rest_framework.response import Response
from rest_framework.authtoken.models import Token
from django.contrib.auth import get_user_model, authenticate
from .models import PlayerStatus
from .serializers import RegisterSerializer, PlayerStatusSerializer

User = get_user_model()


@api_view(['POST'])
@permission_classes([AllowAny])
def register(request):
    serializer = RegisterSerializer(data=request.data)
    if serializer.is_valid():
        user = serializer.save()
        token, _ = Token.objects.get_or_create(user=user)
        return Response({
            'token': token.key,
            'email': user.email
        }, status=status.HTTP_201_CREATED)
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['POST'])
@permission_classes([AllowAny])
def login(request):
    email = request.data.get('email')
    password = request.data.get('password')
    user = authenticate(request, username=email, password=password)
    if user:
        token, _ = Token.objects.get_or_create(user=user)
        return Response({
            'token': token.key,
            'email': user.email
        })
    return Response(
        {'error': 'メールアドレスまたはパスワードが違います'},
        status=status.HTTP_401_UNAUTHORIZED
    )


@api_view(['GET', 'PUT'])
@permission_classes([IsAuthenticated])
def player_status(request):
    try:
        player = PlayerStatus.objects.get(user=request.user)
    except PlayerStatus.DoesNotExist:
        return Response(
            {'error': 'プレイヤーが存在しません'},
            status=status.HTTP_404_NOT_FOUND
        )

    if request.method == 'GET':
        serializer = PlayerStatusSerializer(player)
        return Response(serializer.data)

    if request.method == 'PUT':
        serializer = PlayerStatusSerializer(player, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['POST'])
@permission_classes([IsAuthenticated])
def create_player(request):
    if PlayerStatus.objects.filter(user=request.user).exists():
        return Response(
            {'error': 'プレイヤーは既に存在します'},
            status=status.HTTP_400_BAD_REQUEST
        )
    serializer = PlayerStatusSerializer(data=request.data)
    if serializer.is_valid():
        serializer.save(user=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

## players/urls.py

```python
from django.urls import path
from . import views

urlpatterns = [
    path('register/', views.register, name='register'),
    path('login/', views.login, name='login'),
    path('player/', views.player_status, name='player_status'),
    path('player/create/', views.create_player, name='create_player'),
]
```

## config/urls.py

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('players.urls')),
]
```

---

# STEP13: マイグレーション＆動作確認

## マイグレーション実行

```bash
docker compose run --rm web python manage.py makemigrations
docker compose run --rm web python manage.py migrate
```

## サーバー起動

```bash
docker compose up
```

## 動作確認

`http://localhost:8000/api/register/` にアクセスするとDRFのブラウザUIが表示されます。

Content欄に以下を入力してPOSTします：

```json
{
    "email": "naoki@test.com",
    "password": "TestPass123!"
}
```

レスポンス：

```json
HTTP 201 Created
{
    "token": "21455309175af3e0b794672b08fc8424f6eb9eee",
    "email": "naoki@test.com"
}
```

トークンが発行されればOKです！

---

# トークン認証の仕組み

```
① ユーザーがメール・パスワードでログイン
　↓
② サーバーがトークン（長い文字列）を発行
　↓
③ 以降のリクエストにトークンを添付
   Authorization: Token 21455309175af3e0b794672...
　↓
④ サーバー「このトークンはnaoki@test.comのものだ」→ 認証OK
```

セッションと違い、サーバー側に状態を持たないためAPIに向いています。

---

# セキュリティの注意点

## クライアントのデータを信頼しない

フロントエンド（JavaScript）の変数はブラウザから書き換えられます。

```javascript
// 開発者ツールで簡単に改ざんできる
GameData.player.attack = 999;
```

対策はDjango側でバリデーションすることです。

```python
# 例：attackの最大値を制限する
attack = models.IntegerField(
    default=5,
    validators=[MaxValueValidator(999)]
)
```

**「クライアントから来たデータは必ずサーバー側で検証する」** はWeb開発の鉄則です。

---

# ハマったポイント

## psycopg2がPython 3.14に非対応

```
ERROR: Failed to build 'psycopg2-binary'
```

Python 3.14はpsycopg2非対応のためpsycopg3を使う必要があります。

```
# requirements.txt
psycopg[binary]==3.3.4  # psycopg3
```

---

# まとめ

Django + DRFでプレイヤーステータスを管理するAPIを構築しました。

次回はAWS（S3・ECS・RDS）へのデプロイ手順を紹介します！

---

# ソースコード

https://github.com/naoki-sato-ojiyan/manabi-quest-api