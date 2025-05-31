# SyncStatusIndicator ウィジェット

## 概要

`SyncStatusIndicator` ウィジェットは、Firestoreとのデータ同期状態を視覚的に表示する小さなインジケーターUIコンポーネントです。アプリバーなどに配置され、現在の同期ステータス（同期済み、同期中、オフライン、エラーなど）をアイコンとツールチップでユーザーに伝えます。

## クラス定義

```dart
// lib/features/data_persistence/presentation/widgets/sync_status_indicator.dart

class SyncStatusIndicator extends ConsumerWidget {
  const SyncStatusIndicator({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 同期状態とエラーメッセージを監視
    final syncState = ref.watch(syncStateProvider);
    final errorMessage = ref.watch(syncErrorMessageProvider);

    // 同期状態に応じたアイコン、色、ツールチップを設定
    IconData iconData;
    Color iconColor;
    String tooltipMessage;

    switch (syncState) {
      case SyncState.synced:
        iconData = Icons.cloud_done_outlined;
        iconColor = Colors.green.shade600;
        tooltipMessage = 'データは最新です';
        break;
      case SyncState.syncing:
        iconData = Icons.sync_rounded;
        iconColor = Colors.blue.shade600;
        tooltipMessage = 'データを同期中...';
        break;
      case SyncState.pendingSync:
        iconData = Icons.cloud_off_outlined;
        iconColor = Colors.orange.shade600;
        tooltipMessage = 'オフライン: 未同期の変更があります';
        break;
      case SyncState.error:
        iconData = Icons.error_outline_rounded;
        iconColor = Colors.red.shade600;
        tooltipMessage = '同期エラー: ${errorMessage ?? "詳細不明"}';
        break;
      default:
        iconData = Icons.help_outline_rounded; // 不明な状態
        iconColor = Colors.grey.shade600;
        tooltipMessage = '同期状態不明';
        break;
    }

    return Tooltip(
      message: tooltipMessage,
      child: InkWell(
        onTap: () {
          // エラー時は詳細を表示するなどのアクションを検討
          if (syncState == SyncState.error && errorMessage != null) {
            _showSyncErrorDialog(context, errorMessage);
          }
          // TODO: 同期状態の詳細画面や手動同期トリガーへの導線も検討
        },
        borderRadius: BorderRadius.circular(20), // タップ範囲を広げるため
        child: Padding(
          padding: const EdgeInsets.symmetric(horizontal: 8.0, vertical: 4.0),
          child: Row(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(
                iconData,
                color: iconColor,
                size: 20,
              ),
              // 同期中の場合はプログレスインジケーターを表示
              if (syncState == SyncState.syncing)
                Container(
                  width: 14,
                  height: 14,
                  margin: const EdgeInsets.only(left: 6),
                  child: CircularProgressIndicator(
                    strokeWidth: 2,
                    valueColor: AlwaysStoppedAnimation<Color>(iconColor),
                  ),
                ),
            ],
          ),
        ),
      ),
    );
  }

  // 同期エラーダイアログを表示するヘルパー
  void _showSyncErrorDialog(BuildContext context, String errorMessage) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Row(
          children: [
            Icon(Icons.error_outline_rounded, color: Colors.red.shade600),
            SizedBox(width: 8),
            Text('同期エラー'),
          ],
        ),
        content: Text(errorMessage),
        actions: [
          TextButton(
            onPressed: () {
              Navigator.of(context).pop();
            },
            child: Text('閉じる'),
          ),
          // TODO: 再試行ボタンなどを追加検討
        ],
      ),
    );
  }
}
```

## プロパティ

なし（`ConsumerWidget`のため、プロパティは持たず、Riverpodプロバイダから状態を取得します）

## 内部状態

なし（状態はRiverpodプロバイダによって管理されます）

## メソッド

### build

```dart
Widget build(BuildContext context, WidgetRef ref)
```

ウィジェットのUIを構築します。`syncStateProvider`と`syncErrorMessageProvider`を監視し、現在の同期状態に応じてアイコン、色、ツールチップを動的に変更します。

### _showSyncErrorDialog (プライベートヘルパー)

```dart
void _showSyncErrorDialog(BuildContext context, String errorMessage)
```

同期エラーが発生した場合に、エラー詳細を表示するためのダイアログを構築・表示します。

## 使用例

`SyncStatusIndicator`は、通常、アプリケーションのメイン画面の`AppBar`のアクション部分などに配置されます。

```dart
// MainScreenのAppBar内での使用例
AppBar(
  title: Text('ChartMyRoots'),
  actions: [
    // ... 他のアクションボタン ...
    Padding(
      padding: const EdgeInsets.only(right: 8.0),
      child: SyncStatusIndicator(), // ここで使用
    ),
  ],
)
```

## 注意事項

- このウィジェットは、`syncStateProvider`と`syncErrorMessageProvider`を通じて現在のデータ同期状態を取得します。これらのプロバイダは、Firestoreとの通信状態やオフライン状態を監視し、適切に更新される必要があります。
- アイコン、色、ツールチップメッセージは、`SyncState`列挙型の各状態に対応して定義されています。デザインや要件に応じてカスタマイズ可能です。
- 同期中の場合は、`CircularProgressIndicator`を表示してユーザーに処理中であることを伝えます。
- エラー発生時にインジケーターをタップすると、エラー詳細ダイアログが表示されます。このダイアログには、将来的に「再試行」ボタンなどを追加することも検討できます。
- `InkWell`と`Padding`を使用して、タップ可能な領域を確保し、視覚的なフィードバックを提供しています。

## 関連するクラス・プロバイダ

- `syncStateProvider` (Riverpodプロバイダ) - 現在のデータ同期状態（`SyncState`）を提供するプロバイダ
- `syncErrorMessageProvider` (Riverpodプロバイダ) - 同期エラー発生時のエラーメッセージを提供するプロバイダ
- `SyncState` (列挙型) - データ同期の状態（例: synced, syncing, pendingSync, error）を表す列挙型
- `SyncController` (アプリケーション内のコントローラ) - データ同期処理を制御し、上記のプロバイダを更新するコントローラ（想定）
