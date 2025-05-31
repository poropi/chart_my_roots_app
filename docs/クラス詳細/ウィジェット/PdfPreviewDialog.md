# PdfPreviewDialog ウィジェット

## 概要

`PdfPreviewDialog` ウィジェットは、生成されたPDFドキュメントをユーザーが確認できるようにプレビュー表示するためのモーダルダイアログUIコンポーネントです。`printing`パッケージの`PdfPreview`ウィジェットを利用して、実際のPDFレンダリングを行い、ページめくりやズームなどの基本的なプレビュー機能を提供します。また、プレビュー画面から直接PDFをダウンロードまたは印刷するアクションも提供します。

## クラス定義

```dart
// lib/features/pdf_export/presentation/dialogs/pdf_preview_dialog.dart

class PdfPreviewDialog extends ConsumerWidget {
  const PdfPreviewDialog({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // PDF生成状態とサービスを取得
    final pdfGenerationState = ref.watch(pdfGenerationStateProvider);
    final pdfData = pdfGenerationState.pdfData;
    final exportController = ref.read(pdfExportControllerProvider);

    return Dialog.fullscreen(
      child: Scaffold(
        appBar: AppBar(
          title: const Text('PDFプレビュー'),
          leading: IconButton(
            icon: const Icon(Icons.close),
            onPressed: () => Navigator.of(context).pop(),
          ),
          actions: [
            // ダウンロードボタン
            IconButton(
              icon: const Icon(Icons.download_outlined),
              tooltip: 'PDFをダウンロード',
              onPressed: pdfData != null
                  ? () async {
                      final settings = ref.read(pdfSettingsProvider);
                      final timestamp = DateTime.now().millisecondsSinceEpoch;
                      final sanitizedTitle = settings.title.replaceAll(RegExp(r'[\\/:*?"<>|]'), '_').trim();
                      final fileName = sanitizedTitle.isNotEmpty ? '${sanitizedTitle}_$timestamp.pdf' : '家系図_$timestamp.pdf';
                      await exportController.downloadPdf(fileName);
                      if (context.mounted && ref.read(pdfGenerationStateProvider).isSuccess) {
                         PdfErrorHandler.showSuccessSnackBar(context, 'PDFを保存しました');
                      }
                    }
                  : null,
            ),
            // 印刷ボタン
            IconButton(
              icon: const Icon(Icons.print_outlined),
              tooltip: 'PDFを印刷',
              onPressed: pdfData != null
                  ? () async {
                      try {
                        await Printing.layoutPdf(
                          onLayout: (PdfPageFormat format) async => pdfData,
                          name: ref.read(pdfSettingsProvider).title, // 印刷ジョブ名
                        );
                      } catch (e) {
                        if (context.mounted) {
                          PdfErrorHandler.showErrorSnackBar(context, '印刷処理中にエラーが発生しました: $e');
                        }
                      }
                    }
                  : null,
            ),
          ],
        ),
        body: Column(
          children: [
            // 警告・エラーメッセージ表示エリア
            if (pdfGenerationState.warningMessage != null)
              _buildMessageBanner(pdfGenerationState.warningMessage!, Colors.orange, Icons.warning_amber_rounded),
            if (pdfGenerationState.errorMessage != null && !pdfGenerationState.isGenerating && pdfData == null)
              _buildMessageBanner(pdfGenerationState.errorMessage!, Theme.of(context).colorScheme.error, Icons.error_outline_rounded),
            
            // PDFプレビューエリア
            Expanded(
              child: pdfData != null
                  ? PdfPreview(
                      build: (format) => pdfData, // 表示するPDFデータ
                      allowPrinting: false, // AppBarに別途印刷ボタンを配置するためfalse
                      allowSharing: false,  // AppBarに別途ダウンロード/共有ボタンを配置するためfalse
                      canChangePageFormat: false, // ページフォーマットは固定
                      canChangeOrientation: false, // 向きは固定
                      maxPageWidth: 700, // プレビュー時の最大ページ幅
                      initialPageNumber: 0, // 最初のページから表示
                      pdfPreviewPageDecoration: BoxDecoration( // ページの背景や影
                        color: Colors.grey.shade200,
                        boxShadow: const [
                          BoxShadow(color: Colors.black26, blurRadius: 8, offset: Offset(0, 2)),
                        ],
                      ),
                      loadingWidget: const Center(child: CircularProgressIndicator()),
                      errorBuilder: (context, error) => Center(
                        child: Text('PDFプレビューの表示中にエラーが発生しました: $error',
                          style: TextStyle(color: Theme.of(context).colorScheme.error),
                        ),
                      ),
                    )
                  : Center(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          const Icon(Icons.picture_as_pdf_outlined, size: 64, color: Colors.grey),
                          const SizedBox(height: 16),
                          const Text('プレビューするPDFデータがありません。'),
                          if (pdfGenerationState.isGenerating)
                            const Padding(padding: EdgeInsets.only(top:16), child: CircularProgressIndicator()),
                        ],
                      ),
                    ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildMessageBanner(String message, Color backgroundColor, IconData icon) {
    return Material(
      color: backgroundColor.withOpacity(0.1),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 16.0, vertical: 8.0),
        child: Row(children: [
          Icon(icon, color: backgroundColor, size: 20),
          const SizedBox(width: 12),
          Expanded(child: Text(message, style: TextStyle(color: backgroundColor, fontSize: 13, fontWeight: FontWeight.w500))),
        ]),
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

ウィジェットのUIを構築します。フルスクリーンの`Dialog`内に`Scaffold`を配置し、`AppBar`にタイトル、閉じるボタン、ダウンロードボタン、印刷ボタンを配置します。`body`には`PdfPreview`ウィジェットを配置して、生成されたPDFデータを表示します。PDFデータがない場合やエラー時は適切なメッセージを表示します。

### _buildMessageBanner (プライベートヘルパー)

```dart
Widget _buildMessageBanner(String message, Color backgroundColor, IconData icon)
```

警告メッセージやエラーメッセージを画面上部にバナースタイルで表示するための共通ウィジェットを構築します。

## 使用例

`PdfPreviewDialog`は、`PdfSettingsDialog`で「プレビュー」ボタンが押され、PDFデータが正常に生成された後に表示されます。

```dart
// PdfSettingsDialog内でのプレビュー表示処理
Future<void> _onPreviewPressed(BuildContext context, WidgetRef ref, PdfExportController controller) async {
  await controller.generatePdfForPreview();
  final state = ref.read(pdfGenerationStateProvider); // 最新の状態を取得
  if (state.isComplete && state.pdfData != null && context.mounted) {
    Navigator.of(context).pop(); // 設定ダイアログを閉じる
    showDialog(
      context: context,
      builder: (_) => ResponsivePdfPreviewDialog( // レスポンシブ対応ラッパー
        child: PdfPreviewDialog(),
      ),
      // useSafeArea: false, // フルスクリーンダイアログの場合
    );
  }
}
```

## 注意事項

- このウィジェットは、`pdfGenerationStateProvider`を通じて生成されたPDFデータを取得します。
- PDFの実際のレンダリングとプレビュー機能は、`printing`パッケージの`PdfPreview`ウィジェットに依存しています。このパッケージが`pubspec.yaml`に追加されている必要があります。
- `PdfPreview`ウィジェットの各種プロパティ（`allowPrinting`, `allowSharing`など）は、`AppBar`にカスタムアクションボタンを配置しているため、ここでは`false`に設定しています。
- ダウンロード処理と印刷処理は、それぞれ`PdfExportController`のメソッドと`Printing.layoutPdf`を利用して行います。
- エラーハンドリングとして、PDFデータがない場合や`PdfPreview`ウィジェット自体でエラーが発生した場合の表示も考慮しています。
- `ResponsivePdfPreviewDialog`は、このダイアログをラップして画面サイズに応じた表示調整を行うウィジェットです（別途定義想定）。フルスクリーンダイアログとして表示することを想定しています。

## 関連するクラス・プロバイダ

- `pdfGenerationStateProvider` (Riverpodプロバイダ) - PDF生成処理の状態と生成されたPDFデータを管理
- [PdfExportController](../コントローラ/PdfExportController.md) - PDFのダウンロード処理などを担当するコントローラ
- [PdfSettings](../モデル/PdfSettings.md) - PDF設定情報を保持するモデル（ファイル名生成などに使用）
- `PdfPreview` (`printing`パッケージ) - PDFドキュメントをウィジェットとして表示する機能を提供
- `Printing` (`printing`パッケージ) - PDFの印刷機能を提供
- `PdfErrorHandler` (ユーティリティクラス) - エラーメッセージの表示を行う (別途定義想定)
- `ResponsivePdfPreviewDialog` (ウィジェット) - このダイアログのレスポンシブ対応ラッパー (別途定義想定)
