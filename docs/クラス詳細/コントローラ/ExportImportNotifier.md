# ExportImportNotifier クラス (`StateNotifier`)

## 概要

`ExportImportNotifier` クラスは、家系図データのエクスポートおよびインポート処理中の状態 (`ExportImportState`) を管理する `StateNotifier` です。処理の開始（ローディング）、成功、失敗（エラーメッセージを含む）といった状態をUIに通知し、ユーザーに適切なフィードバックを提供します。Riverpodプロバイダを通じてUIウィジェット（例: `ExportImportDialog`）から利用されます。

## クラス定義 (`ExportImportState` と `ExportImportNotifier`)

```dart
// lib/features/data_persistence/application/export_import_controller.dart (または providers.dart)

// エクスポート・インポート処理の状態を表すイミュータブルなクラス
@freezed
class ExportImportState with _$ExportImportState {
  const factory ExportImportState({
    @Default(false) bool isLoading,       // 処理中かどうか
    @Default(false) bool isSuccess,       // 処理が成功したかどうか
    String? successMessage,             // 成功メッセージ
    String? errorMessage,               // エラーメッセージ
    // double? progress,                // (オプション) 詳細な進捗状況 (0.0 ~ 1.0)
  }) = _ExportImportState;
}

// ExportImportStateを管理するStateNotifier
class ExportImportNotifier extends StateNotifier<ExportImportState> {
  ExportImportNotifier() : super(const ExportImportState());

  // 処理開始 (ローディング状態に設定)
  void startLoading() {
    state = const ExportImportState(isLoading: true);
  }

  // 処理成功
  void setSuccess(String message) {
    state = ExportImportState(
      isLoading: false,
      isSuccess: true,
      successMessage: message,
    );
  }

  // 処理失敗
  void setError(String message) {
    state = ExportImportState(
      isLoading: false,
      isSuccess: false,
      errorMessage: message,
    );
  }
  
  // (オプション) 進捗更新
  // void updateProgress(double progressValue) {
  //   state = state.copyWith(isLoading: true, progress: progressValue.clamp(0.0, 1.0));
  // }

  // 状態をリセット (ダイアログを閉じる際など)
  void reset() {
    state = const ExportImportState();
  }
}
```

## 関連するプロバイダ

```dart
// lib/features/data_persistence/application/export_import_providers.dart (または適切な場所)

// ExportImportNotifierのプロバイダ
final exportImportStateProvider = 
  StateNotifierProvider.autoDispose<ExportImportNotifier, ExportImportState>((ref) {
    return ExportImportNotifier();
});

// インポートモード選択プロバイダ (ExportImportDialogで選択されたモードを保持)
final importModeProvider = StateProvider.autoDispose<ImportMode>((ref) {
  return ImportMode.create; // デフォルトは新規作成モード
});
```

## `ExportImportState` プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `isLoading` | `bool` | エクスポートまたはインポート処理が実行中かどうか。 | `false` |
| `isSuccess` | `bool` | 直前の処理が成功したかどうか。 | `false` |
| `successMessage` | `String?` | 処理成功時に表示するメッセージ。 | `null` |
| `errorMessage` | `String?` | 処理失敗時に表示するエラーメッセージ。 | `null` |
| `progress` | `double?` | (オプション) 詳細な進捗状況（0.0から1.0）。大規模なデータのインポート時などに使用。 | `null` |

## `ExportImportNotifier` メソッド

### startLoading

```dart
void startLoading()
```

エクスポートまたはインポート処理を開始する際に呼び出され、状態をローディング中に設定します。

### setSuccess

```dart
void setSuccess(String message)
```

処理が正常に完了した際に呼び出され、状態を成功に設定し、成功メッセージを保持します。

**パラメータ:**
- `message`: `String` - 表示する成功メッセージ

### setError

```dart
void setError(String message)
```

処理中にエラーが発生した際に呼び出され、状態をエラーに設定し、エラーメッセージを保持します。

**パラメータ:**
- `message`: `String` - 表示するエラーメッセージ

### updateProgress (オプション)

```dart
// void updateProgress(double progressValue)
```

(オプション) インポート処理などが長時間に及ぶ場合に、進捗状況を更新します。

**パラメータ:**
- `progressValue`: `double` - 進捗状況 (0.0 - 1.0)

### reset

```dart
void reset()
```

状態を初期状態（ローディング中でなく、成功/エラーメッセージもない状態）にリセットします。ダイアログが閉じられる際などに呼び出されます。

## 使用例

```dart
// ExportImportDialogウィジェット内での使用

// エクスポート処理の呼び出し
ElevatedButton(
  onPressed: ref.watch(exportImportStateProvider).isLoading 
    ? null 
    : () async {
        final notifier = ref.read(exportImportStateProvider.notifier);
        notifier.startLoading();
        try {
          final service = ref.read(exportImportServiceProvider);
          final treeId = ref.read(selectedFamilyTreeIdProvider);
          if (treeId == null) throw Exception('家系図が選択されていません');
          
          final fileName = 'family_tree_${DateTime.now().millisecondsSinceEpoch}.json';
          await service.exportToJson(treeId, fileName);
          notifier.setSuccess('エクスポートが完了しました。');
        } catch (e) {
          notifier.setError('エクスポートエラー: ${e.toString()}');
        }
      },
  child: Text('エクスポート'),
)

// インポート処理の呼び出し
ElevatedButton(
  onPressed: ref.watch(exportImportStateProvider).isLoading 
    ? null 
    : () async {
        final notifier = ref.read(exportImportStateProvider.notifier);
        notifier.startLoading();
        try {
          // final jsonString = await pickJsonFile(); // ファイル選択処理
          final jsonString = "{...}"; // ダミー
          final mode = ref.read(importModeProvider);
          final service = ref.read(exportImportServiceProvider);
          
          final importedTreeId = await service.importFromJson(jsonString, mode);
          notifier.setSuccess('インポート完了。家系図ID: $importedTreeId');
          // 必要であれば、インポートされた家系図を選択状態にするなどの処理
          // ref.read(selectedFamilyTreeIdProvider.notifier).state = importedTreeId;
        } catch (e) {
          notifier.setError('インポートエラー: ${e.toString()}');
        }
      },
  child: Text('インポート実行'),
)

// UIでの状態表示
Consumer(
  builder: (context, ref, child) {
    final state = ref.watch(exportImportStateProvider);
    if (state.isLoading) {
      return CircularProgressIndicator();
    } else if (state.errorMessage != null) {
      return Text('エラー: ${state.errorMessage}', style: TextStyle(color: Colors.red));
    } else if (state.isSuccess && state.successMessage != null) {
      return Text('成功: ${state.successMessage}', style: TextStyle(color: Colors.green));
    }
    return SizedBox.shrink(); // 何も表示しない
  },
)
```

## 注意事項

- `ExportImportState`は不変オブジェクトとして`freezed`で定義されています。
- この`StateNotifier`は、エクスポート・インポート処理のUIフィードバック（ローディングインジケータ、成功/エラーメッセージの表示）を管理するために使用されます。
- 実際のファイル選択処理（インポート時）やファイル保存処理（エクスポート時）は、`ExportImportService`やプラットフォーム固有のコードで行われます。
- `autoDispose`をプロバイダに付与することで、この状態が不要になった際に自動的に破棄されるようにしています（例：ダイアログが閉じたとき）。
- オプションの`progress`プロパティと`updateProgress`メソッドは、非常に大きな家系図データのインポートなど、進捗表示が有益な場合に実装を検討します。

## 関連するクラス・プロバイダ

- [ExportImportService](../サービス/ExportImportService.md) - 実際のエクスポート・インポート処理を行うサービスクラス
- `ExportImportDialog` (ウィジェット) - この`StateNotifier`を利用してUIを構築するダイアログ (別途定義想定)
- `ImportMode` (列挙型) - インポートモード（新規作成/マージ/上書き）を定義
- `selectedFamilyTreeIdProvider` (Riverpodプロバイダ) - エクスポート対象の家系図IDを取得するために使用
