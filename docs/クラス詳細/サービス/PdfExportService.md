# PdfExportService クラス

## 概要

`PdfExportService` クラスは、PDFの生成、保存、共有といった一連のPDF出力処理を統括するサービスクラスです。`FamilyTreePdfGenerator` を利用してPDFデータを生成し、プラットフォーム（Web、モバイル、デスクトップ）に応じたファイル操作（ダウンロードまたは共有）を行います。

## クラス定義

```dart
// lib/features/pdf_export/services/pdf_export_service.dart

class PdfExportService {
  // 日本語フォントデータのロード (アセットから)
  Future<Uint8List?> _loadJapaneseFont() async {
    try {
      final fontData = await rootBundle.load('assets/fonts/NotoSansJP-Regular.ttf');
      return fontData.buffer.asUint8List();
    } catch (e) {
      print('日本語フォントの読み込みに失敗しました: $e');
      return null;
    }
  }

  // PDFを生成
  Future<Uint8List> generatePdf({
    required FamilyTreeData familyTreeData,
    required PdfSettings settings,
  }) async {
    try {
      final japaneseFontData = await _loadJapaneseFont();
      final generator = FamilyTreePdfGenerator(japaneseFontData: japaneseFontData);
      
      // バックグラウンドでのPDF生成を検討 (computeを使用)
      // return compute(_generatePdfIsolate, {
      //   'familyTreeData': familyTreeData,
      //   'settings': settings,
      //   'fontData': japaneseFontData,
      // });
      
      return await generator.generatePdf(
        familyTreeData: familyTreeData,
        settings: settings,
      );
    } catch (e) {
      _logError('PDF生成中にエラーが発生しました', e);
      rethrow;
    }
  }
  
  // PDF生成のIsolate用関数 (computeで使用する場合)
  // static Future<Uint8List> _generatePdfIsolate(Map<String, dynamic> params) async {
  //   final familyTreeData = params['familyTreeData'] as FamilyTreeData;
  //   final settings = params['settings'] as PdfSettings;
  //   final fontData = params['fontData'] as Uint8List?;
  //   final generator = FamilyTreePdfGenerator(japaneseFontData: fontData);
  //   return await generator.generatePdf(familyTreeData: familyTreeData, settings: settings);
  // }

  // PDFをダウンロード（Web）または共有（モバイル/デスクトップ）
  Future<void> downloadOrSharePdf(Uint8List pdfData, String fileName) async {
    try {
      if (kIsWeb) {
        // Webの場合はダウンロード
        final blob = html.Blob([pdfData], 'application/pdf');
        final url = html.Url.createObjectUrlFromBlob(blob);
        final anchor = html.AnchorElement(href: url)
          ..setAttribute('download', fileName)
          ..click();
        html.Url.revokeObjectUrl(url);
      } else {
        // モバイル/デスクトップの場合は一時ファイルに保存して共有
        final directory = await getTemporaryDirectory();
        final path = '${directory.path}/$fileName';
        final file = io.File(path);
        await file.writeAsBytes(pdfData);
        
        // 共有ダイアログを表示
        await Share.shareXFiles(
          [XFile(path)],
          text: '家系図PDF: $fileName',
          subject: fileName, // メールの件名などに使用
        );
      }
    } catch (e) {
      _logError('PDFのダウンロード/共有中にエラーが発生しました', e);
      rethrow;
    }
  }
  
  // PDFプレビュー用の画像を生成 (オプション機能)
  Future<Uint8List?> generatePdfPreviewImage({
    required FamilyTreeData familyTreeData,
    required PdfSettings settings,
    int pageNumber = 0, // プレビューするページ番号 (0始まり)
  }) async {
    try {
      final pdfData = await generatePdf(familyTreeData: familyTreeData, settings: settings);
      
      // pdf_renderパッケージなどを使用してPDFの特定ページを画像に変換
      // final doc = await PdfDocument.openData(pdfData);
      // final page = await doc.getPage(pageNumber + 1); // ページ番号は1始まり
      // final pageImage = await page.render(width: page.width, height: page.height);
      // await page.close();
      // return pageImage?.bytes;
      
      // ここではダミーデータを返す (pdf_renderの導入が必要なため)
      print('PDFプレビュー画像生成は現在ダミー実装です。');
      return null;
    } catch (e) {
      _logError('PDFプレビュー画像生成中にエラーが発生しました', e);
      return null;
    }
  }
  
  // エラーログ記録（実際の実装ではロギングライブラリを使用）
  void _logError(String message, dynamic error) {
    print('ERROR (PdfExportService): $message - $error');
  }
}
```

## プロバイダ

```dart
// lib/features/pdf_export/application/pdf_export_controller.dart (または専用のproviderファイル)

final pdfExportServiceProvider = Provider<PdfExportService>((ref) {
  return PdfExportService();
});
```

## メソッド詳細

### generatePdf

```dart
Future<Uint8List> generatePdf({
  required FamilyTreeData familyTreeData,
  required PdfSettings settings,
})
```

指定された家系図データと設定に基づいてPDFドキュメントを生成します。内部で`FamilyTreePdfGenerator`を利用します。大規模な家系図の場合、パフォーマンス向上のために`compute`関数を使用してバックグラウンドIsolateで処理することも検討されます（コード内にコメントアウトで例示）。

**パラメータ:**
- `familyTreeData`: `FamilyTreeData` - PDF化する家系図データ
- `settings`: `PdfSettings` - PDFの出力設定

**戻り値:**
- `Future<Uint8List>` - 生成されたPDFデータのバイナリ

**例外:**
- フォントの読み込み失敗やPDF生成中のエラーが発生した場合、例外がスローされます。

### downloadOrSharePdf

```dart
Future<void> downloadOrSharePdf(Uint8List pdfData, String fileName) async
```

生成されたPDFデータを、プラットフォームに応じてダウンロードまたは共有します。
- **Web**: ブラウザの機能を利用して指定されたファイル名でPDFをダウンロードさせます。
- **モバイル/デスクトップ**: PDFデータを一時ファイルとして保存し、OSの共有機能（`share_plus`パッケージを利用）を呼び出してユーザーが他のアプリで共有できるようにします。

**パラメータ:**
- `pdfData`: `Uint8List` - ダウンロード/共有するPDFデータ
- `fileName`: `String` - 保存する際のファイル名

**戻り値:**
- `Future<void>` - 処理の完了を示すFuture

**例外:**
- ファイル操作や共有処理中にエラーが発生した場合、例外がスローされます。

### generatePdfPreviewImage (オプション)

```dart
Future<Uint8List?> generatePdfPreviewImage({
  required FamilyTreeData familyTreeData,
  required PdfSettings settings,
  int pageNumber = 0,
})
```

PDFの特定ページを画像としてレンダリングし、プレビュー用に返します。この機能の実装には`pdf_render`のような追加パッケージが必要になる場合があります。

**パラメータ:**
- `familyTreeData`: `FamilyTreeData` - PDF化する家系図データ
- `settings`: `PdfSettings` - PDFの出力設定
- `pageNumber`: `int` - プレビューするページ番号（0始まり）

**戻り値:**
- `Future<Uint8List?>` - 生成されたプレビュー画像のバイナリデータ、またはエラー時は`null`

## 使用例

```dart
// サービスの取得
final pdfService = ref.read(pdfExportServiceProvider);

// 家系図データと設定の準備
final familyTree = ref.read(selectedFamilyTreeStreamProvider).value;
final settings = ref.read(pdfSettingsProvider);

if (familyTree != null) {
  try {
    // 1. PDFを生成
    final Uint8List pdfBytes = await pdfService.generatePdf(
      familyTreeData: familyTree,
      settings: settings,
    );
    
    // 2. 生成されたPDFをダウンロードまたは共有
    final fileName = '${familyTree.name}_家系図.pdf';
    await pdfService.downloadOrSharePdf(pdfBytes, fileName);
    
    print('PDF処理が完了しました。');
    
  } catch (e) {
    print('PDFエクスポートエラー: $e');
    // ユーザーにエラーを通知
  }
}
```

## 注意事項

- このサービスクラスは、PDF出力に関する一連のフローを管理します。
- 日本語フォントのロードは、`generatePdf`メソッド内で最初に行われます。アセットパスが正しいことを確認してください。
- Webと非Webプラットフォームでのファイル操作の違いを`kIsWeb`で判定し、適切な処理を呼び出しています。
- モバイル/デスクトップでの共有機能には`share_plus`パッケージ、一時ファイル操作には`path_provider`パッケージが必要です。これらは`pubspec.yaml`に追加されている必要があります。
- `generatePdfPreviewImage`メソッドはオプション機能であり、実装には`pdf_render`などの追加パッケージの検討が必要です。現在のコードではダミー実装となっています。
- エラーハンドリングは、このサービスクラス内である程度行いログを記録しますが、UI層でのユーザーへのフィードバックは別途コントローラ層で行う必要があります。
- 大規模なPDF生成処理はメインスレッドをブロックする可能性があるため、`compute`関数を使用してバックグラウンドIsolateで実行することを強く推奨します（コード内にコメントアウトで例示）。

## 関連するクラス

- [FamilyTreePdfGenerator](./FamilyTreePdfGenerator.md) - 実際のPDFドキュメント構築ロジックを持つクラス
- [PdfSettings](../モデル/PdfSettings.md) - PDFの出力設定を保持するモデルクラス
- [FamilyTreeData](../モデル/FamilyTreeData.md) - PDF化する家系図データを保持するモデルクラス
- [PdfExportController](../コントローラ/PdfExportController.md) - このサービスを利用してUIとの連携を行うコントローラ
