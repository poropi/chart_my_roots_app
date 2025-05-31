# PdfSettingsDialog ウィジェット

## 概要

`PdfSettingsDialog` ウィジェットは、家系図をPDF形式で出力する際の各種設定（タイトル、用紙サイズ、向き、表示オプションなど）をユーザーが行うためのモーダルダイアログUIコンポーネントです。設定内容は`PdfSettings`モデルとして管理され、ユーザーは設定後にPDFのプレビュー表示や直接生成・ダウンロードを行うことができます。

## クラス定義

```dart
// lib/features/pdf_export/presentation/dialogs/pdf_settings_dialog.dart

class PdfSettingsDialog extends ConsumerWidget {
  const PdfSettingsDialog({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final pdfSettings = ref.watch(pdfSettingsProvider);
    final settingsNotifier = ref.read(pdfSettingsProvider.notifier);
    final generationState = ref.watch(pdfGenerationStateProvider);
    final generationNotifier = ref.read(pdfGenerationStateProvider.notifier);
    final exportController = ref.read(pdfExportControllerProvider);

    final isGenerating = generationState.isGenerating;

    return AlertDialog(
      title: const Text('PDF出力設定'),
      content: SingleChildScrollView(
        child: ConstrainedBox(
          constraints: const BoxConstraints(maxWidth: 550), // ダイアログの最大幅
          child: Column(
            mainAxisSize: MainAxisSize.min,
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              // タイトル
              TextFormField(
                initialValue: pdfSettings.title,
                decoration: const InputDecoration(labelText: 'タイトル', border: OutlineInputBorder()),
                onChanged: settingsNotifier.updateTitle,
              ),
              const SizedBox(height: 20),
              // 用紙サイズ
              _buildSectionTitle(context, '用紙サイズ'),
              ...PdfPageFormatOptions.all.map((formatOption) => RadioListTile<PdfPageFormat>(
                title: Text(formatOption.name),
                subtitle: Text(formatOption.dimensions),
                value: formatOption.format,
                groupValue: pdfSettings.pageFormat,
                onChanged: isGenerating ? null : (value) => value != null ? settingsNotifier.updatePageFormat(value) : null,
                dense: true,
              )),
              const SizedBox(height: 20),
              // 向き
              _buildSectionTitle(context, '向き'),
              Row(
                children: [
                  Expanded(
                    child: RadioListTile<PdfOrientation>(
                      title: const Text('縦向き'),
                      value: PdfOrientation.portrait,
                      groupValue: pdfSettings.orientation,
                      onChanged: isGenerating ? null : (value) => value != null ? settingsNotifier.updateOrientation(value) : null,
                      dense: true,
                    ),
                  ),
                  Expanded(
                    child: RadioListTile<PdfOrientation>(
                      title: const Text('横向き'),
                      value: PdfOrientation.landscape,
                      groupValue: pdfSettings.orientation,
                      onChanged: isGenerating ? null : (value) => value != null ? settingsNotifier.updateOrientation(value) : null,
                      dense: true,
                    ),
                  ),
                ],
              ),
              const SizedBox(height: 12),
              // 簡易プレビュー
              _buildSimplePagePreview(pdfSettings),
              const SizedBox(height: 20),
              // 表示オプション
              _buildSectionTitle(context, '表示オプション'),
              CheckboxListTile(
                title: const Text('出力日を表示'),
                value: pdfSettings.showDate,
                onChanged: isGenerating ? null : (value) => value != null ? settingsNotifier.updateShowDate(value) : null,
                dense: true,
                controlAffinity: ListTileControlAffinity.leading,
              ),
              CheckboxListTile(
                title: const Text('凡例を表示'),
                value: pdfSettings.showLegend,
                onChanged: isGenerating ? null : (value) => value != null ? settingsNotifier.updateShowLegend(value) : null,
                dense: true,
                controlAffinity: ListTileControlAffinity.leading,
              ),
              CheckboxListTile(
                title: const Text('1ページに収める'),
                subtitle: const Text('大きな家系図は縮小されます', style: TextStyle(fontSize: 12)),
                value: pdfSettings.fitToSinglePage,
                onChanged: isGenerating ? null : (value) => value != null ? settingsNotifier.updateFitToSinglePage(value) : null,
                dense: true,
                controlAffinity: ListTileControlAffinity.leading,
              ),
              // 警告・エラーメッセージ
              if (generationState.warningMessage != null)
                _buildMessageDisplay(generationState.warningMessage!, Colors.orange.shade700, Icons.warning_amber_rounded),
              if (generationState.errorMessage != null)
                _buildMessageDisplay(generationState.errorMessage!, Theme.of(context).colorScheme.error, Icons.error_outline_rounded),
              // 生成中インジケーター
              if (isGenerating)
                const Padding(
                  padding: EdgeInsets.symmetric(vertical: 16.0),
                  child: Center(child: Column(children: [CircularProgressIndicator(), SizedBox(height: 8), Text('PDF生成中...')]))
                ),
            ],
          ),
        ),
      ),
      actions: <Widget>[
        TextButton(
          onPressed: isGenerating ? null : () => Navigator.of(context).pop(),
          child: const Text('キャンセル'),
        ),
        TextButton(
          onPressed: isGenerating ? null : () => _onPreviewPressed(context, ref, exportController),
          child: const Text('プレビュー'),
        ),
        ElevatedButton(
          onPressed: isGenerating ? null : () => _onGeneratePressed(context, ref, exportController),
          child: isGenerating
              ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(strokeWidth: 2))
              : const Text('PDFを生成'),
        ),
      ],
    );
  }

  Widget _buildSectionTitle(BuildContext context, String title) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 8.0),
      child: Text(title, style: Theme.of(context).textTheme.titleSmall?.copyWith(fontWeight: FontWeight.bold, color: Colors.blueGrey.shade700)),
    );
  }
  
  Widget _buildSimplePagePreview(PdfSettings settings) {
    final double aspectRatio = settings.orientation == PdfOrientation.portrait 
        ? (settings.pageFormat.width / settings.pageFormat.height) 
        : (settings.pageFormat.height / settings.pageFormat.width);
    final bool isPortrait = settings.orientation == PdfOrientation.portrait;

    return Center(
      child: Container(
        width: isPortrait ? 100 : 140,
        height: isPortrait ? 140 : 100,
        decoration: BoxDecoration(
          border: Border.all(color: Colors.grey.shade400),
          color: Colors.white,
          boxShadow: [BoxShadow(color: Colors.black12, blurRadius: 2, offset: Offset(1,1))]
        ),
        child: AspectRatio(
          aspectRatio: aspectRatio,
          child: Stack(
            children: [
              Center(child: Icon(Icons.family_restroom_rounded, size: 30, color: Colors.grey.shade300)),
              if (settings.title.isNotEmpty) Positioned(top: 5, left: 5, right: 5, child: Container(height: 8, color: Colors.grey.shade200)),
              if (settings.showDate) Positioned(bottom: 5, left: 5, right: 5, child: Container(height: 6, color: Colors.grey.shade200)),
              if (settings.showLegend) Positioned(bottom: 15, left: 5, child: Container(width: 20, height: 10, color: Colors.grey.shade200)),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildMessageDisplay(String message, Color color, IconData icon) {
    return Padding(
      padding: const EdgeInsets.only(top: 12.0),
      child: Row(children: [
        Icon(icon, color: color, size: 18),
        const SizedBox(width: 8),
        Expanded(child: Text(message, style: TextStyle(color: color, fontSize: 12))),
      ]),
    );
  }

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
      );
    }
  }

  Future<void> _onGeneratePressed(BuildContext context, WidgetRef ref, PdfExportController controller) async {
    await controller.generatePdf();
    final state = ref.read(pdfGenerationStateProvider); // 最新の状態を取得
    if (state.isComplete && state.pdfData != null && context.mounted) {
      final settings = ref.read(pdfSettingsProvider);
      final timestamp = DateTime.now().millisecondsSinceEpoch;
      final sanitizedTitle = settings.title.replaceAll(RegExp(r'[\\/:*?"<>|]'), '_').trim();
      final fileName = sanitizedTitle.isNotEmpty ? '${sanitizedTitle}_$timestamp.pdf' : '家系図_$timestamp.pdf';
      
      await controller.downloadPdf(fileName);
      if (context.mounted && ref.read(pdfGenerationStateProvider).isSuccess) { // 再度成功状態を確認
         Navigator.of(context).pop();
         PdfErrorHandler.showSuccessSnackBar(context, 'PDFを保存しました');
      }
    }
  }
}

// 用紙サイズ選択肢のヘルパークラス
class PdfPageFormatOptions {
  final String name;
  final String dimensions;
  final PdfPageFormat format;

  const PdfPageFormatOptions(this.name, this.dimensions, this.format);

  static List<PdfPageFormatOptions> get all => [
    const PdfPageFormatOptions('A4', '21.0 x 29.7 cm', PdfPageFormat.a4),
    const PdfPageFormatOptions('A3', '29.7 x 42.0 cm', PdfPageFormat.a3),
    const PdfPageFormatOptions('Letter', '8.5 x 11 in', PdfPageFormat.letter),
  ];
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

ウィジェットのUIを構築します。`AlertDialog`内に、タイトル入力、用紙サイズ選択（ラジオボタン）、向き選択（ラジオボタン）、簡易プレビュー、表示オプション選択（チェックボックス）、操作ボタン（キャンセル、プレビュー、PDFを生成）を表示します。

### _buildSectionTitle (プライベートヘルパー)

```dart
Widget _buildSectionTitle(BuildContext context, String title)
```

各設定セクションのタイトルを表示するウィジェットを構築します。

### _buildSimplePagePreview (プライベートヘルパー)

```dart
Widget _buildSimplePagePreview(PdfSettings settings)
```

現在の設定に基づいて、PDFの簡易的なプレビュー（ページの向きや主要要素の配置イメージ）を表示するウィジェットを構築します。

### _buildMessageDisplay (プライベートヘルパー)

```dart
Widget _buildMessageDisplay(String message, Color color, IconData icon)
```

警告メッセージやエラーメッセージを表示するための共通ウィジェットを構築します。

### _onPreviewPressed (プライベートヘルパー)

```dart
Future<void> _onPreviewPressed(BuildContext context, WidgetRef ref, PdfExportController controller) async
```

「プレビュー」ボタンが押されたときの処理を行います。`PdfExportController`を通じてPDFを生成し、成功すればプレビューダイアログを表示します。

### _onGeneratePressed (プライベートヘルパー)

```dart
Future<void> _onGeneratePressed(BuildContext context, WidgetRef ref, PdfExportController controller) async
```

「PDFを生成」ボタンが押されたときの処理を行います。`PdfExportController`を通じてPDFを生成し、成功すればダウンロード/共有処理を実行します。

## PdfPageFormatOptions ヘルパークラス

用紙サイズの選択肢（名前、寸法、`PdfPageFormat`オブジェクト）を管理するためのヘルパークラスです。

## 使用例

`PdfSettingsDialog`は、メイン画面のPDF出力ボタンなどから呼び出されます。

```dart
// PDF設定ダイアログの表示
void _showPdfSettingsDialog(BuildContext context, WidgetRef ref) {
  // 必要であれば、ダイアログ表示前にpdfGenerationStateProviderをリセット
  ref.read(pdfGenerationStateProvider.notifier).reset();
  // 選択中の家系図に基づいてpdfSettingsProviderをリセット
  final currentTree = ref.read(selectedFamilyTreeStreamProvider).value;
  ref.read(pdfSettingsProvider.notifier).reset(currentTree);
  
  showDialog(
    context: context,
    builder: (_) => ResponsivePdfSettingsDialog( // レスポンシブ対応ラッパー
      child: PdfSettingsDialog(),
    ),
  );
}
```

## 注意事項

- このウィジェットは、`pdfSettingsProvider`を通じてPDF設定の状態を管理し、`pdfGenerationStateProvider`を通じてPDF生成処理の状態を監視します。
- 実際のPDF生成とダウンロード/共有は`PdfExportController`に委譲されます。
- 用紙サイズの選択肢は`PdfPageFormatOptions`ヘルパークラスで定義されています。
- 簡易プレビューは、ユーザーが設定変更を視覚的に確認しやすくするためのものです。
- 大規模な家系図の場合の警告メッセージや、生成エラー時のメッセージは`pdfGenerationStateProvider`から取得して表示します。
- 保存処理や生成処理中は、ボタンを無効化し、`CircularProgressIndicator`を表示してユーザーに処理中であることを伝えます。
- `ResponsivePdfSettingsDialog`は、このダイアログをラップして画面サイズに応じた表示調整を行うウィジェットです（別途定義想定）。

## 関連するクラス・プロバイダ

- [PdfSettings](../モデル/PdfSettings.md) - PDF出力設定を保持するモデルクラス
- `pdfSettingsProvider` (Riverpodプロバイダ) - `PdfSettings`の状態を管理
- `pdfGenerationStateProvider` (Riverpodプロバイダ) - PDF生成処理の状態を管理
- [PdfExportController](../コントローラ/PdfExportController.md) - PDF生成と出力を制御するコントローラ
- [PdfPreviewDialog](./PdfPreviewDialog.md) - 生成されたPDFをプレビュー表示するダイアログウィジェット
- `PdfPageFormat` (`pdf`パッケージ) - 用紙サイズを定義するクラス
- `PdfOrientation` (`pdf_export/models/pdf_settings.dart`) - 用紙の向きを定義する列挙型
- `PdfErrorHandler` (ユーティリティクラス) - エラーメッセージの表示を行う (別途定義想定)
- `ResponsivePdfSettingsDialog` (ウィジェット) - このダイアログのレスポンシブ対応ラッパー (別途定義想定)
