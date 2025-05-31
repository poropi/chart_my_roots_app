# ConnectivityService クラス

## 概要

`ConnectivityService` クラスは、デバイスのネットワーク接続状態を監視し、アプリケーション全体でオンライン/オフライン状態をリアルタイムに把握できるようにするためのサービスクラスです。`connectivity_plus` パッケージを利用して、ネットワークの変更を検知し、その状態をStreamとして提供します。

## クラス定義

```dart
// lib/services/connectivity_service.dart

class ConnectivityService {
  final Connectivity _connectivity;

  ConnectivityService({Connectivity? connectivity}) 
      : _connectivity = connectivity ?? Connectivity();

  // ネットワーク接続状態の変更を監視するストリーム
  Stream<ConnectivityResult> get onConnectivityChanged {
    return _connectivity.onConnectivityChanged;
  }

  // 現在のネットワーク接続状態を取得
  Future<ConnectivityResult> checkConnectivity() async {
    return await _connectivity.checkConnectivity();
  }

  // 現在オンラインかどうかを判定
  Future<bool> isOnline() async {
    final result = await checkConnectivity();
    return result != ConnectivityResult.none;
  }
  
  // ネットワークタイプを文字列で取得 (デバッグ用など)
  String getConnectivityStatusString(ConnectivityResult result) {
    switch (result) {
      case ConnectivityResult.wifi:
        return 'WiFi接続';
      case ConnectivityResult.mobile:
        return 'モバイルデータ接続';
      case ConnectivityResult.ethernet:
        return 'イーサネット接続';
      case ConnectivityResult.vpn:
        return 'VPN接続';
      case ConnectivityResult.bluetooth:
        return 'Bluetooth接続';
      case ConnectivityResult.other:
        return 'その他接続';
      case ConnectivityResult.none:
        return 'オフライン';
      default:
        return '不明な接続状態';
    }
  }
}
```

## プロバイダ

```dart
// lib/services/connectivity_service_provider.dart (または app_providers.dart など)

// Connectivityインスタンスプロバイダ (テスト用にモック可能にするため)
final connectivityInstanceProvider = Provider<Connectivity>((ref) {
  return Connectivity();
});

// ConnectivityServiceプロバイダ
final connectivityServiceProvider = Provider<ConnectivityService>((ref) {
  final connectivity = ref.watch(connectivityInstanceProvider);
  return ConnectivityService(connectivity: connectivity);
});

// ネットワーク接続状態のStreamプロバイダ
final connectivityResultStreamProvider = StreamProvider<ConnectivityResult>((ref) {
  final service = ref.watch(connectivityServiceProvider);
  return service.onConnectivityChanged;
});

// 現在オンラインかどうかを判定するプロバイダ
final isOnlineProvider = Provider<bool>((ref) {
  // リアルタイムの接続状態を監視
  final connectivityResult = ref.watch(connectivityResultStreamProvider);
  
  return connectivityResult.when(
    data: (result) => result != ConnectivityResult.none,
    loading: () => true, // 初期状態はオンラインと仮定 (またはcheckConnectivityで確認)
    error: (_, __) => false, // エラー時はオフラインとみなす
  );
});

// アプリ起動時に一度だけ現在の接続状態を確認するFutureProvider (オプション)
final initialConnectivityStatusProvider = FutureProvider<ConnectivityResult>((ref) async {
  final service = ref.watch(connectivityServiceProvider);
  return await service.checkConnectivity();
});
```

## メソッド詳細

### onConnectivityChanged

```dart
Stream<ConnectivityResult> get onConnectivityChanged
```

ネットワーク接続状態の変更を監視するストリームを返します。接続タイプ（WiFi、モバイルデータ、オフラインなど）が変更されるたびに、新しい`ConnectivityResult`がストリームに流れます。

**戻り値:**
- `Stream<ConnectivityResult>` - ネットワーク接続結果のストリーム

### checkConnectivity

```dart
Future<ConnectivityResult> checkConnectivity() async
```

現在のネットワーク接続状態を一度だけ確認します。

**戻り値:**
- `Future<ConnectivityResult>` - 現在のネットワーク接続結果

### isOnline

```dart
Future<bool> isOnline() async
```

現在デバイスがオンライン（何らかのネットワークに接続されている）かどうかを判定します。

**戻り値:**
- `Future<bool>` - オンラインの場合は`true`、オフラインの場合は`false`

### getConnectivityStatusString

```dart
String getConnectivityStatusString(ConnectivityResult result)
```

`ConnectivityResult`を人間が読める形式の文字列（日本語）に変換します。主にデバッグやログ表示に使用します。

**パラメータ:**
- `result`: `ConnectivityResult` - 変換する接続結果

**戻り値:**
- `String` - 接続状態を表す文字列

## 使用例

```dart
// サービスの取得
final connectivityService = ref.read(connectivityServiceProvider);

// ネットワーク状態の変更を監視
ref.listen<AsyncValue<ConnectivityResult>>(connectivityResultStreamProvider, (previous, next) {
  final result = next.value;
  if (result != null) {
    final statusString = connectivityService.getConnectivityStatusString(result);
    print('ネットワーク状態変更: $statusString');
    
    if (result == ConnectivityResult.none) {
      // オフライン時の処理（例：UIに通知を表示）
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('オフラインになりました')),
      );
    } else {
      // オンライン復帰時の処理（例：データの再同期）
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('オンラインに復帰しました')),
      );
      // ref.read(dataSyncControllerProvider).triggerSync();
    }
  }
});

// 現在オンラインかどうかを確認して処理を分岐
Future<void> fetchDataFromServer() async {
  final bool online = await connectivityService.isOnline();
  if (online) {
    print('オンラインです。サーバーからデータを取得します。');
    // データ取得処理
  } else {
    print('オフラインです。ローカルデータを使用するか、接続を待ってください。');
    // オフライン時の代替処理
  }
}

// isOnlineProviderを使用してUIを更新
Consumer(
  builder: (context, ref, child) {
    final isDeviceOnline = ref.watch(isOnlineProvider);
    return Text(isDeviceOnline ? 'オンライン' : 'オフライン');
  },
)
```

## 注意事項

- このサービスは`connectivity_plus`パッケージに依存しています。`pubspec.yaml`にパッケージが追加されていることを確認してください。
- 各プラットフォーム（Android、iOS、Webなど）でネットワーク権限が正しく設定されている必要があります。
  - **Android**: `AndroidManifest.xml`に`<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />`を追加します。
  - **iOS**: 通常、追加の権限設定は不要です。
- `onConnectivityChanged`ストリームは、ネットワーク状態が実際に変更されたときにイベントを発行します。アプリ起動時の初期状態を取得するには`checkConnectivity()`を別途呼び出すか、`initialConnectivityStatusProvider`のようなFutureProviderを利用します。
- `isOnlineProvider`はリアルタイムの接続状態を反映しますが、初期値の扱いやエラー時の挙動に注意してください。アプリの要件に応じて調整が必要です。
- VPNやBluetoothテザリングなど、多様な接続タイプが存在するため、`ConnectivityResult.none`以外は基本的にオンラインとみなすのが一般的です。

## 関連するクラス・インターフェース

- `Connectivity` (connectivity_plusパッケージ) - ネットワーク接続状態の監視機能を提供するコアクラス
- `ConnectivityResult` (connectivity_plusパッケージ) - ネットワーク接続の種類を表す列挙型
- `SyncController` (アプリケーション内のクラス) - オンライン/オフライン状態に応じてデータ同期を制御するコントローラ（想定）
