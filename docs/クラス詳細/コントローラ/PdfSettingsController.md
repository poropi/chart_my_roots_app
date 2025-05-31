# PdfSettingsController クラス (`StateNotifier`)

## 概要

`PdfSettingsController`（実際には `PdfSettingsNotifier` という名前の `StateNotifier`）は、PDF出力時の設定（タイトル、用紙サイズ、向き、表示オプションなど）の状態 (`PdfSettings`) を管理します。ユーザーがUIを通じて行った設定変更を保持し、PDF生成処理に渡すための設定情報を提供します。Riverpodプロバイダを通じてUIウィジェット（例: `PdfSettingsDialog`）から利用されます。

## クラス定義 (`PdfSettings` と `PdfSettingsNotifier`)

```dart
// lib/features/pdf_export/application/pdf_settings_controller.dart

// PdfSettingsモデルは既にモデルフォルダのPdfSettings.mdで定義済みのため、ここでは省略
// import 'package:chart_my_roots_app/features/pdf_export/models/pdf_settings.dart';
// import 'package:chart_my_roots_app/models/family_tree_data.dart'; // For initial title
// import 'package:pdf/pdf.dart'; // For PdfPageFormat

// PdfSettingsを管理するStateNotifier
class PdfSettingsNotifier extends StateNotifier<PdfSettings> {
  PdfSettingsNotifier(PdfSettings initialState) : super(initialState);

  // フォームを初期化 (または家系図データに基づいてリセット)
  void initializeOrDefault(FamilyTreeData? currentFamilyTree) {
    String initialTitle = '家系図';
    if (currentFamilyTree != null && currentFamilyTree.name.isNotEmpty) {
      initialTitle = '家系図 - ${currentFamilyTree.name}';
    }
    state = PdfSettings(title: initialTitle); // 他のフィールドはデフォルト値を使用
  }

  // 各設定フィールドの更新メソッド
  void updateTitle(String title) {
    state = state.copyWith(title: title);
  }

  void updatePageFormat(PdfPageFormat pageFormat) {
    state = state.copyWith(pageFormat: pageFormat);
  }

  void updateOrientation(PdfOrientation orientation) {
    state = state.copyWith(orientation: orientation);
  }

  void updateShowDate(bool showDate) {
    state = state.copyWith(showDate: showDate);
  }

  void updateShowLegend(bool showLegend) {
    state = state.copyWith(showLegend: showLegend);
  }

  void updateFitToSinglePage(bool fitToSinglePage) {
    state = state.copyWith(fitToSinglePage: fitToSinglePage);
  }
  
  // 状態を完全にリセット (オプション)
  void resetToDefaults() {
    state = const PdfSettings(title: '家系図'); // デフォルトタイトルでリセット
  }
}
```

## 関連するプロバイダ

```dart
// lib/features/pdf_export/application/pdf_export_providers.dart (または適切な場所)

// PdfSettingsNotifierのプロバイダ
final pdfSettingsProvider = StateNotifierProvider.autoDispose<PdfSettingsNotifier, PdfSettings>((ref) {
  // アプリ起動時またはダイアログ表示時に、選択中の家系図名で初期化することを推奨
  final selectedTree = ref.watch(selectedFamilyTreeStreamProvider).valueOrNull; 
  String initialTitle = '家系図';
  if (selectedTree != null && selectedTree.name.isNotEmpty) {
    initialTitle = '家系図 - ${selectedTree.name}';
  }
  return PdfSettingsNotifier(PdfSettings(title: initialTitle));
});

// PDF生成処理の状態を管理するプロバイダ (PdfExportController.mdで定義済み)
// final pdfGenerationStateProvider = StateNotifierProvider<PdfGenerationNotifier, PdfGenerationState>(...);

// PDF出力サービスプロバイダ (PdfExportController.mdで定義済み)
// final pdfExportServiceProvider = Provider<PdfExportService>(...);
```

## `PdfSettings` モデルプロパティ

(詳細は [PdfSettings.md](../モデル/PdfSettings.md) を参照)

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `title` | `String` | PDFのタイトル | - |
| `pageFormat` | `PdfPageFormat` | 用紙サイズ | `PdfPageFormat.a4` |
| `orientation` | `PdfOrientation` | 用紙の向き | `PdfOrientation.portrait` |
| `showDate` | `bool` | 日付を表示するか | `true` |
| `showLegend` | `bool` | 凡例を表示するか | `true` |
| `fitToSinglePage` | `bool` | 1ページに収めるか | `false` |

## `PdfSettingsNotifier` メソッド

### initializeOrDefault

```dart
void initializeOrDefault(FamilyTreeData? currentFamilyTree)
```

フォームを初期化します。引数に現在の家系図データ (`FamilyTreeData`) を渡すことで、家系図名に基づいた初期タイトルを設定できます。渡されない場合はデフォルトのタイトル「家系図」が使用されます。その他の設定は`PdfSettings`モデルのデフォルト値に従います。

**パラメータ:**
- `currentFamilyTree`: `FamilyTreeData?` - 現在選択されている家系図データ（オプショナル）

### updateTitle / updatePageFormat / updateOrientation / updateShowDate / updateShowLegend / updateFitToSinglePage

```dart
void updateTitle(String title)
// 他のフィールドも同様
```

各設定フィールドの値が変更されたときにUIから呼び出され、対応する`PdfSettings`の状態を更新します。

### resetToDefaults (オプション)

```dart
void resetToDefaults()
```

PDF設定をアプリケーション定義のデフォルト値（例：タイトル「家系図」、A4縦など）にリセットします。

## 使用例

```dart
// PdfSettingsDialogウィジェット内での使用

// ダイアログ表示時の初期化 (例: initStateやbuildの初回)
// final currentTree = ref.read(selectedFamilyTreeStreamProvider).valueOrNull;
// ref.read(pdfSettingsProvider.notifier).initializeOrDefault(currentTree);

// タイトル入力フィールド
TextFormField(
  initialValue: ref.watch(pdfSettingsProvider).title,
  onChanged: (value) => ref.read(pdfSettingsProvider.notifier).updateTitle(value),
  // ...
)

// 用紙サイズ選択ラジオボタン
RadioListTile<PdfPageFormat>(
  title: Text('A4'),
  value: PdfPageFormat.a4,
  groupValue: ref.watch(pdfSettingsProvider).pageFormat,
  onChanged: (value) => ref.read(pdfSettingsProvider.notifier).updatePageFormat(value!),
  // ...
)

// 「PDFを生成」ボタンの処理
ElevatedButton(
  onPressed: () {
    final settings = ref.read(pdfSettingsProvider);
    final familyTree = ref.read(selectedFamilyTreeStreamProvider).value;
    if (familyTree != null) {
      ref.read(pdfExportControllerProvider).generatePdf(); // generatePdfは内部でsettingsを読む
    }
  },
  child: Text('PDFを生成'),
)
```

## 注意事項

- `PdfSettings`モデルクラスは、`freezed`を使用して不変オブジェクトとして定義されていることを前提とします。
- `PdfSettingsNotifier`は、UIからの設定変更を受け取り、状態 (`PdfSettings`) を更新する責務を持ちます。
- バリデーションロジックは、このコントローラではなく、UI側（例: `PdfSettingsDialog`）または専用のバリデータクラスで行うことを想定しています（PDF設定には通常複雑なバリデーションは少ないため）。
- `autoDispose`をプロバイダに付与することで、設定ダイアログが閉じられるなどしてプロバイダが不要になった際に、状態が自動的に破棄されるようにしています。
- `initializeOrDefault`メソッドは、ダイアログが表示される際や、対象の家系図が変更された際に呼び出され、適切な初期タイトルを設定するのに役立ちます。

## 関連するクラス・プロバイダ

- [PdfSettings](../モデル/PdfSettings.md) - PDF出力設定を保持するモデルクラス
- [FamilyTreeData](../モデル/FamilyTreeData.md) - 初期タイトル生成のために参照される家系図データモデル
- `selectedFamilyTreeStreamProvider` (Riverpodプロバイダ) - 現在選択されている家系図のデータストリーム
- [PdfExportController](./PdfExportController.md) - 実際のPDF生成処理をトリガーするコントローラ
- [PdfSettingsDialog](../ウィジェット/PdfSettingsDialog.md) - このフォームコントローラを利用するUIウィジェット
