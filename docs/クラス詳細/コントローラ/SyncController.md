# SyncController クラス

## 概要

`SyncController` クラスは、アプリケーションのデータ同期状態（オンライン/オフライン、Firestoreとの同期状況など）を管理し、UIにフィードバックを提供するためのコントローラクラスです。`ConnectivityService` からネットワーク状態の変更を受け取り、それに応じて同期ステータスを更新します。また、Firestoreのデータ操作（特に書き込み操作）の成功/失敗に応じて同期状態を更新する役割も担います。

## クラス定義

```dart
// lib/features/data_persistence/application/sync_controller.dart

class SyncController {
  final Ref _ref;
  StreamSubscription<ConnectivityResult>? _connectivitySubscription;

  SyncController(this._ref) {
    // ネットワーク接続状態の監視を開始
    _connectivitySubscription = _ref.read(connectivityResultStreamProvider.stream)
        .listen(_handleConnectivityChange);
    
    // 初期接続状態を確認
    _ref.read(connectivityServiceProvider).checkConnectivity().then(_handleConnectivityChange);
  }

  // ネットワーク接続状態の変更を処理
  void _handleConnectivityChange(ConnectivityResult result) {
    final currentSyncState = _ref.read(syncStateProvider);
    if (result == ConnectivityResult.none) {
      // オフラインになった場合
      if (currentSyncState != SyncState.pendingSync && currentSyncState != SyncState.error) {
        _ref.read(syncStateProvider.notifier).state = SyncState.pendingSync;
        _ref.read(syncErrorMessageProvider.notifier).state = 'ネットワーク接続がありません。';
      }
    } else {
      // オンラインになった場合
      if (currentSyncState == SyncState.pendingSync || currentSyncState == SyncState.error) {
        // 以前オフラインまたはエラーだった場合は、同期中に遷移
        // (実際の同期処理はFirestoreのオフライン機能に依存するか、手動トリガー)
        _ref.read(syncStateProvider.notifier).state = SyncState.syncing;
        _ref.read(syncErrorMessageProvider.notifier).state = null;
        // ここで自動的に保留中の書き込みを再試行するロジックや、
        // ユーザーに再同期を促す通知を出すことも検討できる。
        // Firestoreのオフライン機能が有効なら、接続時に自動的に同期が試みられる。
        // その完了を検知する仕組みが必要な場合がある。
        // 簡単な例として、少し遅れてsyncedにする (実際の同期完了検知が望ましい)
        Future.delayed(const Duration(seconds: 3), () {
          if (_ref.read(syncStateProvider) == SyncState.syncing) { // まだsyncingなら
             _ref.read(syncStateProvider.notifier).state = SyncState.synced;
          }
        });
      }
    }
  }

  // データ書き込み操作の開始を通知
  void notifyDataWriteStarted() {
    if (_ref.read(isOnlineProvider)) {
      _ref.read(syncStateProvider.notifier).state = SyncState.syncing;
      _ref.read(syncErrorMessageProvider.notifier).state = null;
    }
    // オフライン時は pendingSync のまま (Firestoreがローカルに書き込む)
  }

  // データ書き込み操作の成功を通知
  void notifyDataWriteSuccess() {
    if (_ref.read(isOnlineProvider)) {
      _ref.read(syncStateProvider.notifier).state = SyncState.synced;
      _ref.read(syncErrorMessageProvider.notifier).state = null;
    }
    // オフラインで成功した場合も、pendingSync のまま。
    // オンライン復帰時にFirestoreが同期し、その結果を別途ハンドリングする必要がある。
  }

  // データ書き込み操作の失敗を通知
  void notifyDataWriteError(String errorMessage, {bool isNetworkError = false}) {
    _ref.read(syncStateProvider.notifier).state = SyncState.error;
    _ref.read(syncErrorMessageProvider.notifier).state = errorMessage;
    
    // ネットワークエラーが原因でなければ、pendingSyncにはしない
    if (isNetworkError && !_ref.read(isOnlineProvider)){
        _ref.read(syncStateProvider.notifier).state = SyncState.pendingSync;
    }
  }
  
  // 手動で同期を試みる (オプション)
  Future<void> forceSync() async {
    if (!_ref.read(isOnlineProvider)) {
      _ref.read(syncErrorMessageProvider.notifier).state = 'オフラインのため同期できません。';
      return;
    }
    _ref.read(syncStateProvider.notifier).state = SyncState.syncing;
    try {
      // Firestoreの保留中の書き込みを強制的に実行するAPIは直接的にはないが、
      // 例えば、最新データを再取得するなどのアクションを行うことで間接的に同期を促す。
      // または、特定のフラグを更新してFirestoreのリスナーをトリガーするなど。
      // ここでは、単純に状態を更新し、Firestoreの自動同期に期待する。
      await Future.delayed(const Duration(seconds: 2)); // ダミーの同期処理時間
      _ref.read(syncStateProvider.notifier).state = SyncState.synced;
      _ref.read(syncErrorMessageProvider.notifier).state = null;
    } catch (e) {
      notifyDataWriteError('手動同期中にエラーが発生しました: ${e.toString()}');
    }
  }

  // コントローラが破棄される際にストリーム購読をキャンセル
  void dispose() {
    _connectivitySubscription?.cancel();
  }
}
```

## 関連するプロバイダ

```dart
// lib/features/data_persistence/application/sync_providers.dart (または適切な場所)

// 同期状態の列挙型
enum SyncState {
  synced,      // 同期済み (オンラインで最新)
  syncing,     // 同期中 (オンラインで処理中)
  pendingSync, // 同期待ち (オフラインで未同期の変更がある可能性)
  error,       // 同期エラー発生
}

// 現在の同期状態を保持するプロバイダ
final syncStateProvider = StateProvider<SyncState>((ref) {
  // 初期状態は、ネットワーク状態に基づいて決定
  final isOnline = ref.watch(isOnlineInitialValueProvider); // 初期接続状態を使用
  return isOnline ? SyncState.synced : SyncState.pendingSync;
});

// 同期エラーメッセージを保持するプロバイダ
final syncErrorMessageProvider = StateProvider<String?>((ref) => null);

// アプリ起動時の初期オンライン状態 (FutureProviderの結果を同期的に利用する例)
final isOnlineInitialValueProvider = Provider<bool>((ref) {
  // initialConnectivityStatusProvider は FutureProvider<ConnectivityResult>
  // ここでは簡略化のため、常にtrueを返すか、実際の値で初期化するロジックが必要
  // 例: return ref.watch(initialConnectivityStatusProvider).valueOrNull != ConnectivityResult.none;
  // ただし、FutureProviderの値を直接watchするとビルドエラーになるため工夫が必要。
  // ここでは、ConnectivityServiceのisOnline()を起動時に一度だけ呼ぶことを想定。
  // もしくは、起動時に checkConnectivity() の結果で syncStateProvider を初期化する。
  return true; // 仮の初期値
});

// SyncControllerのプロバイダ
final syncControllerProvider = Provider.autoDispose<SyncController>((ref) {
  final controller = SyncController(ref);
  ref.onDispose(() => controller.dispose()); // autoDispose時に購読をキャンセル
  return controller;
});
```

## メソッド詳細

### _handleConnectivityChange (プライベートヘルパー)

```dart
void _handleConnectivityChange(ConnectivityResult result)
```

`ConnectivityService`からのネットワーク接続状態の変更イベントを処理します。オフラインになった場合は`SyncState.pendingSync`に、オンラインに復帰した場合は`SyncState.syncing`（その後、成功すれば`SyncState.synced`）に状態を更新します。

### notifyDataWriteStarted

```dart
void notifyDataWriteStarted()
```

Firestoreへのデータ書き込み操作が開始されたことをコントローラに通知します。オンラインであれば`SyncState.syncing`に状態を設定します。

### notifyDataWriteSuccess

```dart
void notifyDataWriteSuccess()
```

Firestoreへのデータ書き込み操作が成功したことをコントローラに通知します。オンラインであれば`SyncState.synced`に状態を設定します。

### notifyDataWriteError

```dart
void notifyDataWriteError(String errorMessage, {bool isNetworkError = false})
```

Firestoreへのデータ書き込み操作が失敗したことをコントローラに通知します。`SyncState.error`に状態を設定し、エラーメッセージを保持します。ネットワークエラーが原因でオフラインになった場合は`SyncState.pendingSync`に設定することも考慮します。

**パラメータ:**
- `errorMessage`: `String` - 表示するエラーメッセージ
- `isNetworkError`: `bool` - エラーがネットワーク関連であるかどうかを示すフラグ（オプション）

### forceSync (オプション)

```dart
Future<void> forceSync() async
```

ユーザーが手動でデータの再同期を試みるためのメソッド（オプション）。オンライン状態でのみ実行可能です。

### dispose

```dart
void dispose()
```

コントローラが破棄される際に、`ConnectivityService`のストリーム購読をキャンセルします。`autoDispose`プロバイダと組み合わせて使用されます。

## 使用例

```dart
// リポジトリや他のサービス内でデータ書き込みを行う前後で呼び出す

// 書き込み開始時
// ref.read(syncControllerProvider).notifyDataWriteStarted();

// 書き込み成功時
// ref.read(syncControllerProvider).notifyDataWriteSuccess();

// 書き込み失敗時
// ref.read(syncControllerProvider).notifyDataWriteError('データの保存に失敗しました。');


// UIウィジェット (例: SyncStatusIndicator) で同期状態を監視
final syncState = ref.watch(syncStateProvider);
final errorMessage = ref.watch(syncErrorMessageProvider);

switch (syncState) {
  case SyncState.synced:
    // 同期済みアイコン表示
    break;
  case SyncState.syncing:
    // 同期中アニメーション表示
    break;
  case SyncState.pendingSync:
    // オフラインアイコン表示
    break;
  case SyncState.error:
    // エラーアイコンとメッセージ表示
    // Tooltip(message: errorMessage ?? 'エラー')
    break;
}
```

## 注意事項

- このコントローラは、主にネットワーク状態の変更と、Firestoreへの書き込み操作の結果に基づいて`syncStateProvider`と`syncErrorMessageProvider`を更新します。
- Firestoreのオフライン機能が有効になっている場合、オフライン時の書き込みはローカルにキャッシュされ、オンライン復帰時に自動的に同期が試みられます。このコントローラは、その自動同期の「開始」と「完了」を直接的に検知するわけではありません。より高度な同期完了検知が必要な場合は、Firestoreの`waitForPendingWrites`や、特定のフラグを監視するなどの追加ロジックが必要になります。
- `_handleConnectivityChange`メソッド内のオンライン復帰時の処理は、単純に`SyncState.syncing`にした後、一定時間後に`SyncState.synced`にしています。これは簡略化された例であり、実際のアプリケーションではFirestoreの同期完了をより確実に検知する方法（例：書き込み結果の監視、サーバーからの確認応答など）を検討する必要があります。
- `forceSync`メソッドは、ユーザーに手動での再同期オプションを提供する場合の例ですが、Firestoreの自動同期機能があるため、必須ではありません。
- `isOnlineInitialValueProvider`の初期値設定は、アプリ起動時のUXに影響するため、`ConnectivityService.checkConnectivity()`の結果を非同期に待って設定するか、より洗練された方法で初期状態を決定する必要があります。

## 関連するクラス・プロバイダ

- [ConnectivityService](./ConnectivityService.md) - ネットワーク接続状態を提供するサービス
- `connectivityResultStreamProvider` (Riverpodプロバイダ) - ネットワーク接続状態のストリーム
- `isOnlineProvider` (Riverpodプロバイダ) - 現在オンラインかどうかを示すブール値
- `syncStateProvider` (Riverpodプロバイダ) - 現在のデータ同期状態 (`SyncState`) を保持
- `syncErrorMessageProvider` (Riverpodプロバイダ) - 同期エラーメッセージを保持
- `SyncState` (列挙型) - データ同期の状態を表す
- [SyncStatusIndicator](../ウィジェット/SyncStatusIndicator.md) - 同期状態を表示するUIウィジェット
