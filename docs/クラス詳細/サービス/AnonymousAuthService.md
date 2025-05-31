# AnonymousAuthService クラス

## 概要

`AnonymousAuthService` クラスは、Firebase Authentication を利用した匿名認証機能を提供するサービスクラスです。ユーザーが明示的なアカウント登録なしにアプリの機能を利用できるようにするために使用されます。匿名ユーザーのサインイン、サインアウト、および現在の認証状態の監視機能を提供します。

## クラス定義

```dart
// lib/services/auth_service.dart

class AnonymousAuthService {
  final FirebaseAuth _auth;
  
  AnonymousAuthService(this._auth);
  
  // 匿名サインイン
  Future<User?> signInAnonymously() async {
    try {
      final userCredential = await _auth.signInAnonymously();
      return userCredential.user;
    } on FirebaseAuthException catch (e) {
      _logError('匿名サインインに失敗しました', e);
      // エラーコードに基づいて具体的なエラーメッセージを返すことも可能
      //例: if (e.code == 'operation-not-allowed') { ... }
      rethrow; // UI層で処理するために再スロー
    } catch (e) {
      _logError('予期せぬエラーが発生しました（匿名サインイン）', e);
      rethrow;
    }
  }
  
  // サインアウト
  Future<void> signOut() async {
    try {
      await _auth.signOut();
    } on FirebaseAuthException catch (e) {
      _logError('サインアウトに失敗しました', e);
      rethrow;
    } catch (e) {
      _logError('予期せぬエラーが発生しました（サインアウト）', e);
      rethrow;
    }
  }
  
  // 現在のユーザーを取得
  User? getCurrentUser() {
    return _auth.currentUser;
  }
  
  // 認証状態の変更を監視するストリーム
  Stream<User?> authStateChanges() {
    return _auth.authStateChanges();
  }
  
  // ユーザーIDを取得 (匿名ユーザーID)
  String? getCurrentUserId() {
    return _auth.currentUser?.uid;
  }
  
  // エラーログ記録（実際の実装ではロギングライブラリを使用）
  void _logError(String message, dynamic error) {
    print('ERROR (AnonymousAuthService): $message - $error');
  }
}
```

## プロバイダ

```dart
// lib/services/auth_service_provider.dart (または app_providers.dart など)

// Firebase Auth インスタンスプロバイダ
final firebaseAuthProvider = Provider<FirebaseAuth>((ref) {
  return FirebaseAuth.instance;
});

// 匿名認証サービスプロバイダ
final anonymousAuthServiceProvider = Provider<AnonymousAuthService>((ref) {
  final auth = ref.watch(firebaseAuthProvider);
  return AnonymousAuthService(auth);
});

// 認証状態のStreamプロバイダ
final authStateProvider = StreamProvider<User?>((ref) {
  final authService = ref.watch(anonymousAuthServiceProvider);
  return authService.authStateChanges();
});

// 現在のユーザーIDプロバイダ
final currentUserIdProvider = Provider<String?>((ref) {
  final authService = ref.watch(anonymousAuthServiceProvider);
  return authService.getCurrentUserId();
});
```

## メソッド詳細

### signInAnonymously

```dart
Future<User?> signInAnonymously() async
```

Firebase Authentication を使用して匿名でサインインします。成功すると、匿名ユーザーの`User`オブジェクトを返します。失敗した場合は例外をスローします。

**戻り値:**
- `Future<User?>` - 匿名サインインに成功したユーザーの`User`オブジェクト、またはエラー時は`null`（ただし、通常は例外がスローされる）

**例外:**
- `FirebaseAuthException` - Firebase認証関連のエラー（例：ネットワークエラー、操作が許可されていないなど）
- その他の例外 - 予期せぬエラー

### signOut

```dart
Future<void> signOut() async
```

現在サインインしている匿名ユーザーをサインアウトします。

**戻り値:**
- `Future<void>` - サインアウト処理の完了を示すFuture

**例外:**
- `FirebaseAuthException` - Firebase認証関連のエラー
- その他の例外 - 予期せぬエラー

### getCurrentUser

```dart
User? getCurrentUser()
```

現在サインインしているユーザーの`User`オブジェクトを返します。サインインしていない場合は`null`を返します。

**戻り値:**
- `User?` - 現在のユーザーオブジェクト、または`null`

### authStateChanges

```dart
Stream<User?> authStateChanges()
```

ユーザーの認証状態の変更（サインイン、サインアウト）を監視するストリームを返します。

**戻り値:**
- `Stream<User?>` - ユーザーオブジェクトのストリーム。状態変更時に新しいユーザーオブジェクト（または`null`）が流れます。

### getCurrentUserId

```dart
String? getCurrentUserId()
```

現在サインインしている匿名ユーザーのUIDを返します。サインインしていない場合は`null`を返します。このUIDは、Firestoreのセキュリティルールでユーザーごとのデータアクセスを制御するために使用されます。

**戻り値:**
- `String?` - 現在のユーザーID、または`null`

## 使用例

```dart
// サービスの取得
final authService = ref.read(anonymousAuthServiceProvider);

// アプリ起動時に匿名サインインを実行
Future<void> signInOnAppStart() async {
  try {
    final user = await authService.signInAnonymously();
    if (user != null) {
      print('匿名サインイン成功: User ID = ${user.uid}');
      // ユーザーIDを状態管理プロバイダに設定など
      ref.read(currentUserIdProvider.notifier).state = user.uid;
    } else {
      print('匿名サインインに失敗しました。');
    }
  } catch (e) {
    print('匿名サインインエラー: $e');
    // エラー処理（例：リトライUIの表示）
  }
}

// 認証状態の監視
ref.listen<AsyncValue<User?>>(authStateProvider, (previous, next) {
  final user = next.value;
  if (user != null) {
    print('ユーザーがサインインしました: ${user.uid}');
  } else {
    print('ユーザーがサインアウトしました。');
    // 必要であれば再度匿名サインインを試みるなどの処理
  }
});

// サインアウト処理
Future<void> performSignOut() async {
  try {
    await authService.signOut();
    print('サインアウトしました。');
  } catch (e) {
    print('サインアウトエラー: $e');
  }
}
```

## 注意事項

- 匿名認証は、ユーザーが明示的なアカウント作成なしにアプリを利用開始できるようにするための便利な方法です。
- 匿名ユーザーのデータは、そのユーザーがアプリをアンインストールしたり、アプリデータをクリアしたりすると失われる可能性があります（Firebaseの仕様によります）。重要なデータを永続化する場合は、匿名アカウントを永続的なアカウント（例：メール/パスワード、Googleサインインなど）にリンクする機能の導入を検討してください。
- `FirebaseAuthException`の`code`プロパティを参照することで、より具体的なエラーハンドリングが可能です（例：`operation-not-allowed`はFirebaseコンソールで匿名認証が有効になっていない場合に発生します）。
- `authStateChanges`ストリームをリッスンすることで、アプリ全体で認証状態の変更にリアルタイムに対応できます。
- 取得した匿名ユーザーのUID（`user.uid`）は、Firestoreのセキュリティルールで `/users/{userId}/...` のようなパスへのアクセス制御に使用します。

## 関連するクラス・インターフェース

- `FirebaseAuth` (firebase_authパッケージ) - Firebase Authenticationのコア機能を提供するクラス
- `User` (firebase_authパッケージ) - 認証されたユーザー情報を表すクラス
- `FirestoreService` (または各リポジトリ) - 取得したユーザーIDを利用してFirestoreのパスを構築し、データアクセスを行います。
