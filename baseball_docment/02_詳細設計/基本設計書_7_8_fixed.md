# 第7章 セキュリティ設計

## 7.1. 認証フロー詳細

固定ルール（2.3. 通信方式）に基づき、以下の認証フローを実行する。

### ログイン (F-USR-002)

1. クライアントが `POST /api/v1/auth/login` に `email` と `password` を送信。
2. サーバーは `users` テーブルの `password` を Bcrypt で検証。
3. 成功時、Laravel Sanctum が **Personal Access Token (PAT)** を生成（有効期限24時間）。
4. サーバーはHTTP 200でトークンを返却。

### トークン検証 (全認証必須API)

1. クライアントは `Authorization: Bearer {token}` ヘッダーを付与してAPIリクエスト。
2. サーバー (Laravel) の `auth:sanctum` ミドルウェアがトークンを検証。
3. トークンが無効または期限切れの場合、HTTP 401 (E-401-01) を返却。
4. クライアントはHTTP 401受信時、`flutter_secure_storage` からトークンを削除し、ログイン画面 (SCR-USR-001) に強制遷移させる。

### リフレッシュ

- **リフレッシュトークンは使用しない**（固定ルール）。
- トークン（有効期限24時間）が切れた場合は、必ず再ログインを要求する。

### ログアウト (F-USR-003)

1. クライアントが `POST /api/v1/auth/logout` を（有効なトークンで）呼び出す。
2. サーバーは該当の PAT をデータベースから削除（無効化）する。
3. クライアントは `flutter_secure_storage` からトークンを削除し、ログイン画面 (SCR-USR-001) に遷移する。

---

## 7.2. 認可フロー詳細

### ユーザー種別

- **一般ユーザー** と **管理者** の2種類が存在する。

### 認可方式

Laravel のミドルウェアを用いて認可を制御する。

**一般ユーザー:**
- `auth:sanctum` ミドルウェアのみ。
- 自身の `user_id` に紐づくリソース（例: 参加登録、ログアウト）のみ操作可能。

**管理者:**
- `/api/v1/admin/*` のエンドポイント（F-ADM-001～010）には、`auth:sanctum` に加えて **AdminMiddleware** を適用する。
- AdminMiddleware は、認証されたユーザーが管理者権限を持つか（例: `users` テーブルの `role` カラム、または特定の email リスト）を判定し、権限がない場合は HTTP 403 (E-403-01) を返却する。

---

## 7.3. パスワード管理

- **ハッシュ化アルゴリズム**: Bcrypt （固定ルール）
- **Laravel実装**: `Hash::make($password)` を使用してハッシュ化し、`Hash::check($plainPassword, $hashedPassword)` で検証する。
- **コストパラメータ**: 10 (Laravel デフォルト)
- **ソルト**: Bcrypt に自動的に含まれるため、別途ソルトカラムは用意しない。
- **DB保存**: `users.password` カラム (VARCHAR(255)) にハッシュ値を保存する。

---

## 7.4. トークン保存方式 (Flutter)

- **採用パッケージ**: `flutter_secure_storage` （固定ルール）
- **保存場所**:
  - iOS: Keychain
  - Android: EncryptedSharedPreferences (AES暗号化)
- **目的**: トークンを平文でローカルストレージ（SharedPreferencesなど）に保存することを防ぎ、デバイス固有の安全な領域に格納する。

---

## 7.5. 通信暗号化

- **プロトコル**: TLS 1.2以上 を必須とする。
- **対象**: クライアント (Flutter) とサーバー (Nginx) 間の全API通信。
- **実装**: Xserver VPS上でSSL証明書（Let's Encryptなど）を設定し、HTTPS通信を強制する。（詳細は 16.3. Nginx設定ファイル を参照）

---

## 7.6. 入力値検証

### クライアント側 (Flutter)

- **責務**: ユーザービリティ (UX) の向上。
- **実装**: `TextFormField` の `validator` や Provider で、固定バリデーションルールに基づく形式チェック（必須、文字長、メール形式など）を行う。
- **目的**: 不正なリクエストを送信する前にユーザーに即時フィードバックを提供し、無駄なAPIコールを削減する。

### サーバー側 (Laravel)

- **責務**: データの完全性とセキュリティの担保（必須）。
- **実装**: Laravel FormRequest (`app/Http/Requests/`) を使用し、固定バリデーションルールに基づく全項目の厳密な検証（形式、長さ、一意制約、存在チェックなど）を行う。
- **目的**: クライアント側の検証はバイパス可能であるため、サーバー側で必ず全ての入力を信頼せず検証する（ゼロトラスト）。

---

## 7.7. OWASP Top 10対策 (具体的実装方法)

| 脆弱性 | 対策内容 | Laravel実装 (サーバー側) | Flutter実装 (クライアント側) |
|---|---|---|---|
| **A01: アクセス制御の不備** | 認証・認可の徹底 | - `auth:sanctum` ミドルウェアで全保護エンドポイントをガード<br>- AdminMiddleware で管理者権限をチェック<br>- リソース操作時に、`$request->user()->id` と対象リソースの `user_id` が一致するか確認する。 | - トークンを `flutter_secure_storage` に保存<br>- 全APIリクエストに `Authorization` ヘッダーを付与<br>- 401/403受信時にログイン画面へ遷移させる。 |
| **A02: 暗号化の失敗** | 輸送中・保存中のデータ保護 | - NginxでTLS 1.2以上を強制。HTTPからHTTPSへリダイレクト。<br>- パスワードはBcryptでハッシュ化（ソルト付き） | - API通信はHTTPSのみ許可。<br>- トークンは `flutter_secure_storage` に保存。 |
| **A03: インジェクション** | 入力値のサニタイズとクエリの分離 | - SQLi: Eloquent ORM および クエリビルダー を使用（パラメータバインディング）。生SQL (`DB::raw`, `selectRaw` 等）を原則禁止。<br>- XSS: Bladeテンプレート（管理者画面）では `{{ }}` 構文を使用し、自動的にエスケープする。 | - APIレスポンスをJSONとしてパースし、Widget（Textなど）に表示する（デフォルトでエスケープされる）。 |
| **A04: 安全でない設計** | セキュア設計原則の適用 | - 固定ルールに基づくレートリミットを実装（認証: 120回/分、ログイン: 5回/分）。<br>- 最小権限の原則に基づき、管理者機能とユーザー機能を明確に分離。 | （特になし） |
| **A05: セキュリティ設定の不備** | フレームワークとサーバーの堅牢化 | - `.env` ファイルで `APP_DEBUG=false` （本番環境）<br>- Nginx/PHPのバージョン情報をヘッダーから削除。<br>- Composerの依存関係を定期的に監査 (`composer audit`) | （特になし） |
| **A06: 脆弱で時代遅れのコンポーネント** | 依存関係の管理 | - `composer.json` と `package.json` のライブラリを最新の安定版に保つ。<br>- `composer audit` をCI/CDパイプラインに組み込む。 | - `pubspec.yaml` の依存関係を最新に保つ。<br>- `flutter pub outdated` で定期的に確認。 |
| **A07: 識別と認証の失敗** | セッション管理の強化 | - Sanctumトークンの有効期限を24時間に設定<br>- リフレッシュトークンを使用せず、期限切れ時は再ログインを強制。<br>- ログアウト時にサーバー側でトークンを無効化。 | - ログアウト時に `flutter_secure_storage` からトークンを確実に削除する。 |
| **A08: ソフトウェアとデータの完全性の不備** | デシリアライゼーション、CI/CDの保護 | - （PHPのデシリアライゼーション脆弱性）信頼できないソースからの `unserialize()` を禁止。<br>- CI/CDパイプライン（GitHub Actionsなど）のシークレット（.env）を暗号化して管理する。 | （特になし） |
| **A09: セキュリティログと監視の不備** | イベントの記録とアラート | - ログイン成功/失敗、パスワードリセット、強制退会などのセキュリティイベントをログ（JSON形式）に記録（詳細は12章）。<br>- 5xx系エラーや401/403の多発を監視（詳細は13章）。 | （特になし。クライアントログは通常対象外） |
| **A10: サーバーサイドリクエストフォージェリ (SSRF)** | 外部リソースアクセスの制限 | - （本システムでは該当機能なし）<br>- 将来的にURLから画像取得などを行う場合は、許可リストベースのドメイン検証を必須とする。 | （特になし） |

---

## 詳細設計への共通コンテキスト（第7章）

### 後続タスク（詳細設計）への指示:

**ミドルウェアの実装:**

- `auth:sanctum` を `routes/api.php` の認証必須ルートグループに適用してください。
- 「7.2. 認可フロー詳細」に基づき、管理者権限をチェックする `AdminMiddleware.php` を作成し、`/api/v1/admin/*` ルートグループに適用してください。

**クライアント側実装 (Flutter):**

- 「7.1. 認証フロー詳細」に基づき、HTTP 401エラーをグローバルにハンドリングする処理（APIクライアントやProvider層）を実装し、ログイン画面への強制遷移を行ってください。
- `flutter_secure_storage` を使用してトークンを読み書きする `TokenStorageService` クラスを設計・実装してください。

**レートリミットの実装:**

- 「7.7. OWASP A04」に基づき、Laravel の RouteServiceProvider またはミドルウェアで、固定ルール（認証: 120回/分、ログイン: 5回/分）のレートリミットを設定してください。

---

# 第8章 外部連携設計

## 8.1. Firebase Cloud Messaging (FCM) 連携詳細

### 設定ファイル配置方法

#### iOS (GoogleService-Info.plist)

1. Firebase Console から `GoogleService-Info.plist` をダウンロード。
2. Flutterプロジェクトの `ios/Runner/` ディレクトリに配置する。

```
ios/
  └── Runner/
      └── GoogleService-Info.plist
```

#### Android (google-services.json)

1. Firebase Console から `google-services.json` をダウンロード。
2. Flutterプロジェクトの `android/app/` ディレクトリに配置する。

```
android/
  └── app/
      └── google-services.json
```

---

### 初期化コード (main.dart)

`lib/main.dart` にFirebaseの初期化とバックグラウンドメッセージハンドラーを実装する。

```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// バックグラウンドメッセージハンドラー
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // Firebaseが初期化されていない場合は初期化する
  await Firebase.initializeApp();
  
  print('バックグラウンドメッセージ受信: ${message.messageId}');
  // TODO: 必要に応じてバックグラウンドでのデータ処理や通知表示（flutter_local_notifications）を行う
}

void main() async {
  // Flutter Engineの初期化
  WidgetsFlutterBinding.ensureInitialized();
  
  // Firebase初期化
  await Firebase.initializeApp();
  
  // バックグラウンドメッセージハンドラーの設定
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
  
  // (オプション) フォアグラウンド通知の表示設定
  await FirebaseMessaging.instance.setForegroundNotificationPresentationOptions(
    alert: true, // アラート
    badge: true, // バッジ
    sound: true, // サウンド
  );

  runApp(
    ProviderScope(
      child: MyApp(),
    ),
  );
}
```

---

### デバイストークン登録処理

ログイン完了後、FCMトークンを取得し、サーバーの `/api/v1/device-tokens` APIに送信する。

（`lib/core/services/fcm_service.dart` または `lib/features/auth/application/auth_provider.dart` に実装）

```dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
// (APIクライアントやリポジトリのimport)

class FcmService {
  final Ref ref;
  final FirebaseMessaging _fcm = FirebaseMessaging.instance;

  FcmService(this.ref);

  Future<void> initialize() async {
    // 1. 通知許可ダイアログの表示 (iOS / Android 13+)
    NotificationSettings settings = await _fcm.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    if (settings.authorizationStatus == AuthorizationStatus.authorized) {
      print('通知許可: 許可されました');
      
      // 2. FCMトークンの取得
      final String? token = await _fcm.getToken();
      if (token != null) {
        print('FCM Token: $token');
        
        // 3. サーバーへトークンを送信
        await _registerTokenToServer(token);
      }
      
      // 4. トークンが更新された場合のリスナー設定
      _fcm.onTokenRefresh.listen(_registerTokenToServer);

    } else {
      print('通知許可: 拒否されました');
    }
  }

  Future<void> _registerTokenToServer(String token) async {
    try {
      // (デバイス種別の判定)
      final String deviceType = (Theme.of(navigatorKey.currentContext!).platform == TargetPlatform.iOS) ? 'ios' : 'android';
      
      // APIリポジトリを呼び出し、POST /api/v1/device-tokens を実行
      // await ref.read(deviceTokenRepositoryProvider).registerToken(
      //   token: token,
      //   deviceType: deviceType,
      // );
      
      print('FCMトークンをサーバーに登録しました。');
      
    } catch (e) {
      print('FCMトークン登録失敗: $e');
      // 失敗してもエラーとはしない（通知が届かないだけ）
    }
  }
}
```

---

### プッシュ通知送信フロー (F-ADM-009)

1. 管理者 (Web) が `POST /api/v1/admin/notifications/push` を呼び出す。
2. Laravelコントローラーがリクエストを受領。
3. NotificationService が対象ユーザー（例: 全員、または特定の試合参加者）を決定する。
4. `device_tokens` テーブルから対象ユーザーのFCMトークンを取得する。
5. FCM Admin SDK (PHP) を使用し、対象トークンリストに対してメッセージ（タイトル、本文）を送信する。
6. FCM APIからのレスポンス（成功/失敗）に基づき、結果を管理画面に返す。
7. FCM APIがエラー（例: Unregistered）を返した場合、該当のトークンを `device_tokens` テーブルから削除（または無効化）する処理を非同期で行う。

---

## 8.2. メール配信 (SMTP)

### 連携方式

Laravel の標準 Mail 機能を使用し、Xserver の SMTP サーバー経由で送信する。

### 設定 (.env)

```env
MAIL_MAILER=smtp
MAIL_HOST=your-xserver-smtp-host.com
MAIL_PORT=587
MAIL_USERNAME=your-smtp-username
MAIL_PASSWORD=your-smtp-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="support@yourdomain.com"
MAIL_FROM_NAME="${APP_NAME}"
```

### メールテンプレート設計

Laravel の Mailable クラス (`app/Mail/`) と Blade テンプレート (`resources/views/emails/`) を使用して実装する（詳細は10.3章）。

### 配信タイミング

- **同期**: パスワードリセット要求 (F-USR-004)
- **非同期 (キュー)**: アカウント登録完了 (F-USR-001), 管理者メール配信 (F-ADM-010)

### 配信失敗時のリトライ

固定ルール（通知・メール設計）に基づき、非同期（キュー）で送信するメールはリトライを行う。

- **リトライ回数**: 3回
- **リトライ間隔**: 5分、15分、30分（指数バックオフ）
- 3回失敗した場合はログ（E-500-02）に記録し、手動対応とする。

---

## 8.3. Google Maps連携

### 連携方式

ディープリンク (URLスキーム)

### 目的

試合詳細画面 (SCR-USR-005) の住所をタップした際に、デバイスにインストールされているGoogle Mapsアプリを起動し、該当の住所を表示させる。

### Flutter実装

`url_launcher` パッケージを使用する。

住所文字列（例: 東京都文京区後楽1-3-61）をURLエンコードする。

以下のURLスキームを生成して `launchUrl()` を呼び出す。

```dart
import 'package:url_launcher/url_launcher.dart';

Future<void> launchGoogleMaps(String address) async {
  // 住所をURLエンコード
  final String query = Uri.encodeComponent(address);

  // Google Maps URL (Webフォールバックも考慮)
  final Uri url = Uri.parse('https://www.google.com/maps/search/?api=1&query=$query');

  if (await canLaunchUrl(url)) {
    await launchUrl(url, mode: LaunchMode.externalApplication);
  } else {
    // エラー処理（例: トースト表示）
    print('地図アプリを起動できませんでした');
  }
}
```

---

## 詳細設計への共通コンテキスト（第8章）

### 後続タスク（詳細設計）への指示:

**FCM実装 (Flutter):**

- 「8.1. 初期化コード」と「デバイストークン登録処理」に基づき、`main.dart` と `FcmService`（または `AuthProvider`）を実装してください。
- ログイン成功時のフローに、`FcmService.initialize()` の呼び出しを追加してください。
- フォアグラウンド（アプリ起動中）でプッシュ通知を受信した際のハンドリング処理（例: `FirebaseMessaging.onMessage.listen`）を実装してください。

**FCM実装 (Laravel):**

- `POST /api/v1/device-tokens` API（コントローラー、サービス、リポジトリ）を実装し、受け取ったトークンとデバイス種別を `device_tokens` テーブルに保存（upsert）する処理を実装してください。
- F-ADM-009（プッシュ通知配信）機能の `NotificationService` を設計し、FCM Admin SDK (PHP) を使って `device_tokens` テーブルのトークンリストへ通知を送信するロジックを実装してください。

**SMTP実装 (Laravel):**

- 「8.2. 配信タイミング」に基づき、Mailableクラス（例: `AccountRegistered.php`, `PasswordReset.php`）を作成してください。
- 同期（`Mail::sendNow()`）と非同期（`Mail::queue()`）の送信処理を、該当するサービスクラス（AuthServiceなど）に実装してください。

**Google Maps実装 (Flutter):**

- SCR-USR-005 (試合詳細画面) の住所表示部分（Text や ListTile）を `GestureDetector` や `InkWell` でラップし、`onTap` イベントで「8.3. Google Maps連携」の `launchGoogleMaps` 関数を呼び出すよう実装してください。

---

**第7章と第8章は以上となります。**
