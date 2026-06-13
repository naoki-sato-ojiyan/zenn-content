---
title: "【まなびクエスト開発日記】本番管理画面の修正とセキュリティ強化（レート制限・メール認証・トークン有効期限・AWS SES）"
emoji: "🔐"
type: "tech"
topics: ["django", "aws", "ses", "python", "個人開発"]
published: true
---

# はじめに

小学生向けの学習RPG「まなびクエスト」を個人開発しています。今回はいよいよ本番公開に向けたセキュリティ実装と、AWS SESによるメール認証の設定をしました。

実装した内容はこちらです：

- **レート制限**（ブルートフォース攻撃対策）
- **メール認証**（登録時にメールでアカウント確認）
- **トークン有効期限**（30日で期限切れ）
- **AWS SES設定**（本番環境のメール送信）

---

# レート制限の実装

## なぜレート制限が必要か

APIに制限がないと、悪意のある攻撃者が大量のリクエストを送ってパスワードを総当たりで試す「ブルートフォース攻撃」を受けます。

今回は以下の制限を設けました：

| エンドポイント | 制限 |
|--------------|------|
| `/api/register/` | 同一IPから1時間3回まで |
| `/api/login/` | 同一IPから15分10回まで |

## 実装方針：PostgreSQLで自前実装

`django-ratelimit` などのライブラリも検討しましたが、ローカルと本番で環境差異をなくしたかったため、PostgreSQLを使った自前実装にしました。

**RateLimitLogモデルを追加**

```python
class RateLimitLog(models.Model):
    ip = models.GenericIPAddressField()
    endpoint = models.CharField(max_length=100)
    requested_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # IPとエンドポイントと日時で絞り込むクエリを高速化
            models.Index(fields=['ip', 'endpoint', 'requested_at']),
        ]
```

**チェック関数をviews.pyに追加**

```python
def check_rate_limit(ip, endpoint, max_requests, period_minutes):
    """指定期間内のリクエスト数が上限を超えていたらTrueを返す"""
    since = timezone.now() - timedelta(minutes=period_minutes)
    count = RateLimitLog.objects.filter(
        ip=ip,
        endpoint=endpoint,
        requested_at__gte=since
    ).count()
    if count >= max_requests:
        return True
    RateLimitLog.objects.create(ip=ip, endpoint=endpoint)
    return False

def get_client_ip(request):
    """CloudFront経由でも正しいIPを取得する"""
    forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
    if forwarded_for:
        return forwarded_for.split(',')[0].strip()
    return request.META.get('REMOTE_ADDR')
```

`HTTP_X_FORWARDED_FOR` を使っているのは、CloudFront経由のリクエストでは実際のIPがこのヘッダーに入るためです。

**registerとloginに適用**

```python
@api_view(['POST'])
@permission_classes([AllowAny])
def register(request):
    ip = get_client_ip(request)
    if check_rate_limit(ip, 'register', max_requests=3, period_minutes=60):
        return Response(
            {'error': 'リクエストが多すぎます。しばらく待ってから再試行してください。'},
            status=status.HTTP_429_TOO_MANY_REQUESTS
        )
    # ...以下省略
```

## 動作確認

```bash
# 4回連続で登録APIを叩いてみる
for i in 1 2 3 4; do
  echo "=== $i回目 ==="
  curl -s -X POST http://localhost:8000/api/register/ \
    -H "Content-Type: application/json" \
    -d '{"email":"test'$i'@example.com","password":"testpass123"}'
  echo ""
done
```

```
=== 1回目 ===
{"token":"...","email":"test1@example.com"}
=== 2回目 ===
{"token":"...","email":"test2@example.com"}
=== 3回目 ===
{"token":"...","email":"test3@example.com"}
=== 4回目 ===
{"error":"リクエストが多すぎます。しばらく待ってから再試行してください。"}
```

3回目まで成功し、4回目で429エラーが返ってきました。

---

# メール認証の実装

## なぜメール認証が必要か

メール認証なしだと他人のメールアドレスで登録できてしまいます。週次レポートを送るサービスなので、本人のメールアドレスを確認することが重要です。

## 実装の流れ

```
ユーザーが登録
　↓
is_active=Falseでユーザーを作成
　↓
認証メールを送信（ローカル：MailHog・本番：AWS SES）
　↓
メール内のリンクをクリック
　↓
is_active=True になってログイン可能
```

## EmailVerificationTokenモデルを追加

```python
class EmailVerificationToken(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    token = models.UUIDField(default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

UUIDを使うことで推測困難なトークンを生成できます。

## registerビューを修正

```python
@api_view(['POST'])
@permission_classes([AllowAny])
def register(request):
    # ...レート制限チェック...
    serializer = RegisterSerializer(data=request.data)
    if serializer.is_valid():
        # 変更：登録時はis_active=Falseにしてメール認証待ちにする
        user = serializer.save(is_active=False)
        verification = EmailVerificationToken.objects.create(user=user)
        verify_url = f"{settings.API_BASE_URL}/api/verify-email/?token={verification.token}"
        send_mail(
            subject='【まなびクエスト】メールアドレスの確認',
            message=f'以下のリンクをクリックしてメールアドレスを確認してください。\n\n{verify_url}',
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[user.email],
        )
        return Response(
            {'message': '確認メールを送信しました。メールをご確認ください。'},
            status=status.HTTP_201_CREATED
        )
```

## verify_emailビューを追加

```python
@api_view(['GET'])
@permission_classes([AllowAny])
def verify_email(request):
    token = request.query_params.get('token')
    if not token:
        return Response({'error': 'トークンがありません'},
                        status=status.HTTP_400_BAD_REQUEST)
    try:
        verification = EmailVerificationToken.objects.get(token=token)
    except EmailVerificationToken.DoesNotExist:
        return Response({'error': '無効なトークンです'},
                        status=status.HTTP_400_BAD_REQUEST)
    user = verification.user
    user.is_active = True
    user.save()
    verification.delete()
    Token.objects.get_or_create(user=user)
    return Response({'message': 'メールアドレスの確認が完了しました。ログインしてください。'})
```

## MailHogでローカルテスト

ローカル環境では実際にメールを送らずMailHogで受信します。

**docker-compose.ymlにMailHogを追加**

```yaml
  mailhog:
    image: mailhog/mailhog
    ports:
      - "8025:8025"
      - "1025:1025"
```

**settings.pyのメール設定**

```python
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = os.environ.get('EMAIL_HOST', 'mailhog')
EMAIL_PORT = int(os.environ.get('EMAIL_PORT', 1025))
EMAIL_USE_TLS = os.environ.get('EMAIL_USE_TLS', 'False') == 'True'
EMAIL_HOST_USER = os.environ.get('EMAIL_HOST_USER', '')
EMAIL_HOST_PASSWORD = os.environ.get('EMAIL_HOST_PASSWORD', '')
DEFAULT_FROM_EMAIL = 'noreply@manabi-quest.com'
```

ローカルと本番で環境変数だけ切り替えることで、コードの差異をなくしています。

http://localhost:8025 でMailHogのUIを確認できます。

## メール未認証ユーザーへの対応

`authenticate()` は `is_active=False` のユーザーを認証失敗として返すため、パスワード間違いと区別できません。loginビューに専用メッセージを追加しました。

```python
    user = authenticate(request, username=email, password=password)
    if user:
        # ...ログイン成功処理...
    # 変更：メール未認証の場合は専用メッセージを返す
    try:
        unverified = User.objects.get(email=email)
        if not unverified.is_active:
            return Response(
                {'error': 'メール認証が完了していません。届いたメールのリンクをクリックしてください。'},
                status=status.HTTP_401_UNAUTHORIZED
            )
    except User.DoesNotExist:
        pass
    return Response({'error': 'メールアドレスまたはパスワードが違います'},
                    status=status.HTTP_401_UNAUTHORIZED)
```

---

# トークン有効期限の実装

## なぜトークン有効期限が必要か

DRFのデフォルトトークンは永久に有効です。ログインしたまま放置されたアカウントのセキュリティリスクを減らすため、30日で期限切れにします。

## カスタム認証クラスを作成

`players/authentication.py` を新規作成します。

```python
from datetime import timedelta
from django.utils import timezone
from rest_framework.authentication import TokenAuthentication
from rest_framework.exceptions import AuthenticationFailed


class ExpiringTokenAuthentication(TokenAuthentication):
    """30日で期限切れになるトークン認証"""

    def authenticate_credentials(self, key):
        user, token = super().authenticate_credentials(key)
        if timezone.now() > token.created + timedelta(days=30):
            token.delete()
            raise AuthenticationFailed('トークンの有効期限が切れています。再ログインしてください。')
        return user, token
```

`TokenAuthentication` を継承して `authenticate_credentials` をオーバーライドしています。期限切れトークンはDBから削除して、次回ログイン時に新しいトークンが発行されます。

## settings.pyを更新

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'players.authentication.ExpiringTokenAuthentication',  # 変更
    ],
    ...
}
```

## ログイン時にトークンを新規発行

期限切れユーザーが再ログインしたとき、古いトークンが残っていると30日がリセットされません。ログインのたびに新しいトークンを発行します。

```python
    if user:
        # 変更：ログインのたびに新しいトークンを発行して30日延長
        Token.objects.filter(user=user).delete()
        token = Token.objects.create(user=user)
        return Response({
            'token': token.key,
            'email': user.email
        })
```

毎日プレイすれば実質期限切れにならない設計です。

---

# AWS SESの設定

## SESとは

Amazon Simple Email Serviceの略で、AWSのメール送信サービスです。本番環境でメール認証を動作させるために設定します。

## 設定の流れ

```
1. IAMユーザーにAmazonSESFullAccessを追加
2. SESでドメイン認証（manabi-quest.com）
3. Route53にDNSレコードを追加（DKIM・DMARC）
4. SMTP認証情報を作成
5. .env.prodにSES設定を追加
6. 本番アクセス申請（サンドボックス解除）
```

## ドメイン認証

SESの「SMTP設定」→「送信ドメインを追加」で `manabi-quest.com` を追加します。

「DNSレコードを取得」で表示されるレコードをRoute53に追加します：

| タイプ | 用途 |
|--------|------|
| CNAME × 3（DKIM） | メールの電子署名・改ざん防止 |
| TXT（DMARC） | なりすまし防止の設定 |

設定後しばらくするとAWSからDKIM設定成功のメールが届きます。

## SMTP認証情報の作成

SESの「SMTP設定」→「SMTP認証情報の作成」でSMTPユーザー名とパスワードを発行します。

:::message alert
SMTP認証情報は作成時しか表示されません。必ずCSVでダウンロードしてください。
:::

## .env.prodに設定を追加

```
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=email-smtp.ap-northeast-1.amazonaws.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=（SMTPユーザー名）
EMAIL_HOST_PASSWORD=（SMTPパスワード）
DEFAULT_FROM_EMAIL=noreply@manabi-quest.com
FRONTEND_URL=https://manabi-quest.com
API_BASE_URL=https://manabi-quest.com
```

## サンドボックス解除申請

SESはデフォルトでサンドボックスモードになっており、検証済みメールアドレスにしか送れません。本番で実際のユーザーにメールを送るには解除申請が必要です。

「アカウントダッシュボード」→「本番アクセスをリクエスト」から申請します。

| 項目 | 設定値 |
|------|--------|
| メールタイプ | トランザクション |
| ウェブサイトURL | https://manabi-quest.com |
| 希望言語 | 日本語 |

承認は通常24時間以内です。

---

# エラーログの設定

本番でエラーが発生したときにCloudWatchで確認できるよう、`settings.py` にLOGGING設定を追加しました。

```python
# エラーログをCloudWatchに出力する
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'ERROR',
    },
}
```

これがないとDjangoのエラーがCloudWatchに出力されず、本番のデバッグが困難になります。

---

# まとめ

今回実装したセキュリティ対策をまとめます：

| 機能 | 内容 |
|------|------|
| レート制限 | ブルートフォース攻撃対策・PostgreSQLで自前実装 |
| メール認証 | 登録時にメールアドレスを確認・MailHog/SESで送信 |
| トークン有効期限 | 30日で期限切れ・ログイン時に自動リセット |
| AWS SES | 本番環境のメール送信・DKIM/DMARC設定済み |
| エラーログ | CloudWatchにERRORログを出力 |

次回はパスワードリセット機能を実装する予定です。

---

# 参考リンク

- [Amazon SES ドキュメント](https://docs.aws.amazon.com/ses/)
- [Django TokenAuthentication](https://www.django-rest-framework.org/api-guide/authentication/#tokenauthentication)
- [MailHog GitHub](https://github.com/mailhog/MailHog)
