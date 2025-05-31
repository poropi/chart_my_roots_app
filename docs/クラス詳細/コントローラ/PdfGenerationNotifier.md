# PdfGenerationNotifier クラス (`StateNotifier`)

## 概要

`PdfGenerationNotifier` クラスは、PDF生成処理中の状態 (`PdfGenerationState`) を管理する `StateNotifier` です。処理の開始（ローディング）、成功（生成されたPDFデータを含む）、失敗（エラーメッセージを含む）、および警告メッセージといった状態をUIに通知し、ユーザーに適切なフィードバックを提供します。Riverpodプロバイダを通じてUIウィジェット（例: `PdfSettingsDialog`, `PdfPreviewDialog`）から利用されます。

## クラス定義 (`PdfGenerationState` と `PdfGenerationNotifier`)

```dart
// lib/features/pdf_export/application/pdf_export_controller.dart (または providers.dart)

// PDF生成処理の状態を表すイミュータブルなクラス
@freezed
class PdfGenerationState with _$PdfGenerationState {
  const factory PdfGenerationState({
    @Default(false) bool isGenerating,    // 生成処理中かどうか
    @Default(false) bool isComplete,      // 生成処理が完了したかどうか (成功/失敗問わず)
    @Default(false) bool isSuccess,       // 直前のダウンロード/共有が成功したか (オプション)
    String? warningMessage,             // 警告メッセージ (例: 大規模データ)
    String? errorMessage,               // エラーメッセージ
    String? successMessage,             // 成功メッセージ (ダウンロード/共有成功時)
    Uint8List? pdfData,                 // 生成されたPDFデータ
  }) = _PdfGenerationState;
}

// PdfGenerationStateを管理するStateNotifier
class PdfGenerationNotifier extends StateNotifier<PdfGenerationState> {
  PdfGenerationNotifier() : super(const PdfGenerationState());

  // PDF生成処理開始
  void startGenerating() {
    state = const PdfGenerationState(isGenerating: true);
  }

  // PDF生成完了 (成功時)
  void setPdfData(Uint8List data) {
    state = PdfGenerationState(
      isGenerating: false,
      isComplete: true,
      pdfData: data,
    );
  }

  // 警告メッセージを設定
  void setWarning(String message) {
    // isGenerating状態は維持しつつ警告メッセージを追加
    state = state.copyWith(warningMessage: message, errorMessage: null);
  }
  
  // エラー発生
  void setError(String message) {
    state = PdfGenerationState(
      isGenerating: false,
      isComplete: true, // エラーでも処理は一旦完了
      errorMessage: message,
    );
  }
  
  // ダウンロード/共有成功 (オプション)
  void setSuccess(String message) {
    state = state.copyWith(
      isSuccess: true, 
      successMessage: message,
      errorMessage: null, // エラーがあればクリア
    );
  }

  // 状態をリセット (ダイアログを閉じる際や新規生成前に呼び出す)
  void reset() {
    state = const PdfGenerationState();
  }
}
```

## 関連するプロバイダ

```dart
// lib/features/pdf_export/application/pdf_export_providers.dart (または適切な場所)

// PdfGenerationNotifierのプロバイダ
final pdfGenerationStateProvider = 
  StateNotifierProvider.autoDispose<PdfGenerationNotifier, PdfGenerationState>((ref) {
    return PdfGenerationNotifier();
});

// PdfSettingsNotifierのプロバイダ (PdfSettingsController.mdで定義済み)
// final pdfSettingsProvider = StateNotifierProvider<PdfSettingsNotifier, PdfSettings>(...);

// PDF出力サービスプロバイダ (PdfExportController.mdで定義済み)
// final pdfExportServiceProvider = Provider<PdfExportService>(...);

// PDF出力コントローラのプロバイダ (PdfExportController.mdで定義済み)
// final pdfExportControllerProvider = Provider<PdfExportController>(...);
```

## `PdfGenerationState` プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `isGenerating` | `bool` | PDF生成処理が実行中かどうか。 | `false` |
| `isComplete` | `bool` | PDF生成処理が完了したかどうか（成功・失敗を問わず）。 | `false` |
| `isSuccess` | `bool` | 直前のダウンロード/共有操作が成功したかどうか。 | `false` |
| `warningMessage` | `String?` | PDF生成に関する警告メッセージ（例：大規模データで時間がかかる可能性など）。 | `null` |
| `errorMessage` | `String?` | PDF生成または関連処理でエラーが発生した場合のエラーメッセージ。 | `null` |
| `successMessage` | `String?` | PDFのダウンロード/共有成功時に表示するメッセージ。 | `null` |
| `pdfData` | `Uint8List?` | 正常に生成されたPDFのバイナリデータ。 | `null` |

## `PdfGenerationNotifier` メソッド

### startGenerating

```dart
void startGenerating()
```

PDF生成処理を開始する際に呼び出され、状態を「生成中」に設定します。既存のエラーメッセージやPDFデータはクリアされます。

### setPdfData

```dart
void setPdfData(Uint8List data)
```

PDF生成が正常に完了した際に呼び出され、状態を「完了」、生成されたPDFデータを保持するように設定します。

**パラメータ:**
- `data`: `Uint8List` - 生成されたPDFのバイナリデータ

### setWarning

```dart
void setWarning(String message)
```

PDF生成処理中に警告（エラーではないが注意が必要な情報）が発生した場合に呼び出されます。`isGenerating`状態は維持されます。

**パラメータ:**
- `message`: `String` - 表示する警告メッセージ

### setError

```dart
void setError(String message)
```

PDF生成処理中または関連処理（ダウンロードなど）でエラーが発生した際に呼び出され、状態を「完了」かつエラー状態に設定し、エラーメッセージを保持します。

**パラメータ:**
- `message`: `String` - 表示するエラーメッセージ

### setSuccess

```dart
void setSuccess(String message)
```

PDFのダウンロードや共有などの後続処理が成功した場合に呼び出されます。

**パラメータ:**
- `message`: `String` - 表示する成功メッセージ

### reset

```dart
void reset()
```

状態を初期状態（生成中でなく、メッセージやPDFデータもない状態）にリセットします。PDF設定ダイアログが閉じられる際や、新しいPDF生成処理を開始する前に呼び出されることを想定しています。

## 使用例

```dart
// PdfExportController内での使用例
class PdfExportController {
  // ... (他のコード)
  Future<void> generatePdf() async {
    final notifier = _ref.read(pdfGenerationStateProvider.notifier);
    notifier.startGenerating();
    try {
      // ... (家系図データと設定の取得)
      // ... (大規模データの場合の警告設定)
      // if (isLargeTree) notifier.setWarning('大規模な家系図のため時間がかかる場合があります');
      
      final pdfBytes = await _service.generatePdf(familyTreeData: treeData, settings: pdfSettings);
      notifier.setPdfData(pdfBytes);
    } catch (e) {
      notifier.setError('PDF生成エラー: ${e.toString()}');
    }
  }
  
  Future<void> downloadGeneratedPdf(String fileName) async {
    final state = _ref.read(pdfGenerationStateProvider);
    final notifier = _ref.read(pdfGenerationStateProvider.notifier);
    if (state.pdfData != null) {
      try {
        await _service.downloadOrSharePdf(state.pdfData!, fileName);
        notifier.setSuccess('「$fileName」を保存しました。');
      } catch (e) {
        notifier.setError('PDF保存エラー: ${e.toString()}');
      }
    } else {
      notifier.setError('保存するPDFデータがありません。');
    }
  }
}

// PdfSettingsDialogウィジェット内での状態表示例
Consumer(
  builder: (context, ref, child) {
    final state = ref.watch(pdfGenerationStateProvider);
    if (state.isGenerating) {
      return CircularProgressIndicator();
    } else if (state.errorMessage != null) {
      return Text('エラー: ${state.errorMessage}', style: TextStyle(color: Colors.red));
    } else if (state.warningMessage != null) {
      return Text('警告: ${state.warningMessage}', style: TextStyle(color: Colors.orange));
    } else if (state.isComplete && state.pdfData != null) {
      // プレビュー表示やダウンロードボタンを有効化
      return Text('PDF生成完了。プレビューまたはダウンロードしてください。');
    }
    return SizedBox.shrink();
  },
)
```

## 注意事項

- `PdfGenerationState`は不変オブジェクトとして`freezed`で定義されています。
- この`StateNotifier`は、PDF生成という非同期で時間のかかる可能性のある処理のUIフィードバックを管理するために使用されます。
- `isComplete`フラグは、処理が成功したか失敗したかに関わらず、一度処理が試みられたことを示します。
- `isSuccess`フラグは、主にダウンロード/共有のような後続処理の成功を示すためにオプションとして用意されています。
- `autoDispose`をプロバイダに付与することで、この状態が不要になった際に自動的に破棄されるようにしています（例：PDF設定ダイアログが閉じたとき）。
- 実際のPDF生成ロジックは`FamilyTreePdfGenerator`や`PdfExportService`にあり、このNotifierはそれらの処理の「状態」をUIに伝える役割に集中します。

## 関連するクラス・プロバイダ

- [PdfExportController](./PdfExportController.md) - このNotifierを利用してPDF生成処理を制御するコントローラ
- [PdfExportService](../サービス/PdfExportService.md) - 実際のPDF生成とファイル操作を行うサービスクラス
- [PdfSettingsDialog](../ウィジェット/PdfSettingsDialog.md) - このNotifierの状態に基づいてUIを更新するダイアログ
- [PdfPreviewDialog](../ウィジェット/PdfPreviewDialog.md) - 生成されたPDFデータを表示するダイアログ
