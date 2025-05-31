# PdfExportController クラス

## 概要

`PdfExportController` クラスはPDF出力に関する操作を管理するコントローラクラスです。Riverpodプロバイダを通じてアクセスされ、PDF生成処理の開始、進捗状況の管理、エラーハンドリングなどの機能を提供します。PDF生成サービスと連携して実際のPDFデータを生成し、ダウンロードや共有機能も提供します。

## クラス定義

```dart
class PdfExportController {
  final Ref _ref;
  final PdfExportService _service;
  
  PdfExportController(this._ref, this._service);
  
  // PDF生成処理の開始
  Future<void> generatePdf() async {
    try {
      // 生成中状態に設定
      _ref.read(pdfGenerationStateProvider.notifier).startGenerating();
      
      // 選択中の家系図データを取得
      final familyTreeData = _ref.read(selectedFamilyTreeStreamProvider).valueOrNull;
      if (familyTreeData == null) {
        throw Exception('家系図データが見つかりません');
      }
      
      // PDF設定を取得
      final pdfSettings = _ref.read(pdfSettingsProvider);
      
      // 大規模家系図の場合は警告
      if (_shouldWarnForLargeTree(familyTreeData, pdfSettings)) {
        final warningMessage = _getLargeTreeWarningMessage(familyTreeData, pdfSettings);
        _ref.read(pdfGenerationStateProvider.notifier).setWarning(warningMessage);
      }
      
      // PDFを生成
      final pdfData = await _service.generatePdf(
        familyTreeData: familyTreeData,
        settings: pdfSettings,
      );
      
      // 生成完了状態に更新
      _ref.read(pdfGenerationStateProvider.notifier).setPdfData(pdfData);
    } catch (e, stackTrace) {
      // エラー状態に更新
      final errorMessage = _formatErrorMessage(e);
      _ref.read(pdfGenerationStateProvider.notifier).setError(errorMessage);
      
      // エラーログ記録
      _logError('Failed to generate PDF', e, stackTrace);
    }
  }
  
  // PDFをダウンロード（または共有）
  Future<void> downloadPdf(String fileName) async {
    try {
      // 生成済みのPDFデータを取得
      final pdfData = _ref.read(pdfGenerationStateProvider).pdfData;
      if (pdfData == null) {
        throw Exception('PDFデータが見つかりません');
      }
      
      // PDFをダウンロード（または共有）
      await _service.downloadPdf(pdfData, fileName);
      
      // 成功状態に更新
      _ref.read(pdfGenerationStateProvider.notifier).setSuccess(
        'PDFを保存しました',
      );
    } catch (e, stackTrace) {
      // エラー状態に更新
      final errorMessage = _formatErrorMessage(e);
      _ref.read(pdfGenerationStateProvider.notifier).setError(errorMessage);
      
      // エラーログ記録
      _logError('Failed to download PDF', e, stackTrace);
    }
  }
  
  // プレビュー用にPDFを生成
  Future<void> generatePdfForPreview() async {
    await generatePdf();
  }
  
  // 大規模家系図の警告判定
  bool _shouldWarnForLargeTree(FamilyTreeData familyTree, PdfSettings settings) {
    final personCount = familyTree.persons.length;
    return (settings.fitToSinglePage && personCount > 100) || personCount > 150;
  }
  
  // 大規模家系図の警告メッセージ
  String _getLargeTreeWarningMessage(FamilyTreeData familyTree, PdfSettings settings) {
    final personCount = familyTree.persons.length;
    
    if (settings.fitToSinglePage && personCount > 100) {
      return '大規模な家系図（${personCount}人）を1ページに収めると、各ノードが非常に小さくなり読みにくくなる可能性があります。複数ページへの分割をお勧めします。';
    }
    
    if (personCount > 150) {
      return '大規模な家系図（${personCount}人）はPDFの生成に時間がかかる場合があります。しばらくお待ちください。';
    }
    
    return '';
  }
  
  // エラーメッセージのフォーマット
  String _formatErrorMessage(dynamic error) {
    if (error is OutOfMemoryError || 
        (error is Exception && error.toString().contains('memory'))) {
      return 'メモリ不足のため、PDFを生成できませんでした。「1ページに収める」をオフにするか、より小さなサイズで出力してください。';
    }
    
    if (error is Exception && error.toString().contains('font')) {
      return 'フォントの読み込みに失敗しました。';
    }
    
    return 'PDF生成中にエラーが発生しました: ${error.toString()}';
  }
  
  // エラーログ記録（実際の実装ではロギングライブラリを使用）
  void _logError(String message, dynamic error, StackTrace? stackTrace) {
    print('ERROR: $message - $error');
    if (stackTrace != null) {
      print(stackTrace);
    }
  }
}
```

## 関連するプロバイダ

```dart
// PDF出力サービスプロバイダ
final pdfExportServiceProvider = Provider<PdfExportService>((ref) {
  return PdfExportService();
});

// PDF生成状態プロバイダ
final pdfGenerationStateProvider = StateNotifierProvider<PdfGenerationNotifier, PdfGenerationState>((ref) {
  return PdfGenerationNotifier();
});

// PDF設定プロバイダ
final pdfSettingsProvider = StateNotifierProvider<PdfSettingsNotifier, PdfSettings>((ref) {
  // 選択中の家系図から初期タイトルを設定
  final selectedTree = ref.watch(selectedFamilyTreeStreamProvider).valueOrNull;
  
  String initialTitle = '家系図';
  if (selectedTree != null) {
    initialTitle = '家系図 - ${selectedTree.name}';
  }
  
  return PdfSettingsNotifier(
    PdfSettings(title: initialTitle),
  );
});

// PDF出力コントローラのプロバイダ
final pdfExportControllerProvider = Provider<PdfExportController>((ref) {
  final service = ref.watch(pdfExportServiceProvider);
  return PdfExportController(ref, service);
});
```

## PDF生成状態クラス

```dart
// PDF生成状態
class PdfGenerationState {
  final bool isGenerating;
  final bool isComplete;
  final bool isSuccess;
  final String? warningMessage;
  final String? errorMessage;
  final String? successMessage;
  final Uint8List? pdfData;
  
  PdfGenerationState({
    this.isGenerating = false,
    this.isComplete = false,
    this.isSuccess = false,
    this.warningMessage,
    this.errorMessage,
    this.successMessage,
    this.pdfData,
  });
  
  PdfGenerationState copyWith({
    bool? isGenerating,
    bool? isComplete,
    bool? isSuccess,
    String? warningMessage,
    String? errorMessage,
    String? successMessage,
    Uint8List? pdfData,
  }) {
    return PdfGenerationState(
      isGenerating: isGenerating ?? this.isGenerating,
      isComplete: isComplete ?? this.isComplete,
      isSuccess: isSuccess ?? this.isSuccess,
      warningMessage: warningMessage,
      errorMessage: errorMessage,
      successMessage: successMessage,
      pdfData: pdfData ?? this.pdfData,
    );
  }
}

class PdfGenerationNotifier extends StateNotifier<PdfGenerationState> {
  PdfGenerationNotifier() : super(PdfGenerationState());
  
  // PDF生成開始
  void startGenerating() {
    state = PdfGenerationState(
      isGenerating: true,
      isComplete: false,
      pdfData: null,
    );
  }
  
  // 警告を設定
  void setWarning(String message) {
    state = state.copyWith(
      warningMessage: message,
    );
  }
  
  // エラー設定
  void setError(String message) {
    state = PdfGenerationState(
      isGenerating: false,
      isComplete: false,
      errorMessage: message,
    );
  }
  
  // PDF生成完了
  void setPdfData(Uint8List pdfData) {
    state = PdfGenerationState(
      isGenerating: false,
      isComplete: true,
      pdfData: pdfData,
    );
  }
  
  // 成功メッセージ設定
  void setSuccess(String message) {
    state = state.copyWith(
      isSuccess: true,
      successMessage: message,
    );
  }
  
  // 状態リセット
  void reset() {
    state = PdfGenerationState();
  }
}
```

## メソッド

### generatePdf

```dart
Future<void> generatePdf() async
```

選択中の家系図データと設定に基づいてPDFを生成します。生成状態を管理し、成功時はPDFデータを状態に保存します。

**戻り値:**
- `Future<void>` - 生成処理の完了を示すFuture

**例外:**
- 例外は内部で捕捉され、状態オブジェクトのエラーメッセージとして設定されます。

### downloadPdf

```dart
Future<void> downloadPdf(String fileName) async
```

生成済みのPDFデータをダウンロードまたは共有します。

**パラメータ:**
- `fileName`: `String` - 保存するPDFファイルの名前

**戻り値:**
- `Future<void>` - ダウンロード処理の完了を示すFuture

**例外:**
- 例外は内部で捕捉され、状態オブジェクトのエラーメッセージとして設定されます。

### generatePdfForPreview

```dart
Future<void> generatePdfForPreview() async
```

プレビュー表示用にPDFを生成します。内部的には`generatePdf`メソッドを呼び出します。

**戻り値:**
- `Future<void>` - 生成処理の完了を示すFuture

**例外:**
- 例外は内部で捕捉され、状態オブジェクトのエラーメッセージとして設定されます。

## 使用例

```dart
// コントローラの取得
final controller = ref.read(pdfExportControllerProvider);

// PDFの生成
await controller.generatePdf();

// 生成状態の確認
final generationState = ref.watch(pdfGenerationStateProvider);
if (generationState.isComplete && generationState.pdfData != null) {
  // PDFの生成に成功
  
  // ファイル名の生成
  final timestamp = DateTime.now().millisecondsSinceEpoch;
  final fileName = 'family_tree_${timestamp}.pdf';
  
  // PDFのダウンロード
  await controller.downloadPdf(fileName);
} else if (generationState.errorMessage != null) {
  // エラーが発生
  print('Error: ${generationState.errorMessage}');
}

// プレビュー用にPDFを生成
await controller.generatePdfForPreview();
```

## 注意事項

- このコントローラは、PDF生成に関する操作を管理し、生成状態を追跡します。
- 大規模な家系図の場合は警告メッセージを表示し、ユーザーに適切な選択を促します。
- エラーハンドリングを内部で行い、ユーザーフレンドリーなエラーメッセージを提供します。
- メモリ不足エラーなど、特定のエラーパターンに対しては具体的な解決策を提案します。
- 生成状態は`pdfGenerationStateProvider`を通じて追跡され、UIはこの状態に基づいて適切な表示を行います。
- 実際のPDF生成処理は`PdfExportService`に委譲されます。

## 関連するクラス・インターフェース

- [PdfSettings](../モデル/PdfSettings.md) - PDF出力設定を管理するモデルクラス
- [PdfExportService](../リポジトリ/PdfExportService.md) - PDF生成と出力を担当するサービスクラス
- [FamilyTreeData](../モデル/FamilyTreeData.md) - 家系図全体のデータを管理するモデルクラス
