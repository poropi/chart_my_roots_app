# PdfSettings クラス

## 概要

`PdfSettings` クラスはPDF出力に関する設定パラメータを管理するモデルクラスです。タイトル、用紙サイズ、向き、表示オプションなどを保持します。PDF生成プロセスはこの設定クラスを参照して出力形式を決定します。

## クラス定義

```dart
@freezed
class PdfSettings with _$PdfSettings {
  const factory PdfSettings({
    required String title,
    @Default(PdfPageFormat.a4) PdfPageFormat pageFormat,
    @Default(PdfOrientation.portrait) PdfOrientation orientation,
    @Default(true) bool showDate,
    @Default(true) bool showLegend,
    @Default(false) bool fitToSinglePage,
  }) = _PdfSettings;

  factory PdfSettings.fromJson(Map<String, dynamic> json) => _$PdfSettingsFromJson(json);
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `title` | `String` | PDFのタイトル | なし（必須） |
| `pageFormat` | `PdfPageFormat` | 用紙サイズ（A4、A3、レターなど） | `PdfPageFormat.a4` |
| `orientation` | `PdfOrientation` | 用紙の向き（縦/横） | `PdfOrientation.portrait` |
| `showDate` | `bool` | 日付を表示するかどうか | `true` |
| `showLegend` | `bool` | 凡例を表示するかどうか | `true` |
| `fitToSinglePage` | `bool` | 家系図全体を1ページに収めるかどうか | `false` |

## メソッド

### PdfSettings.fromJson

```dart
factory PdfSettings.fromJson(Map<String, dynamic> json)
```

JSONオブジェクトから`PdfSettings`インスタンスを生成します。`freezed`パッケージによって自動生成されます。

**パラメータ:**
- `json`: `Map<String, dynamic>` - 変換元のJSONオブジェクト

**戻り値:**
- `PdfSettings` - 生成された`PdfSettings`インスタンス

### toJson

```dart
Map<String, dynamic> toJson()
```

`PdfSettings`インスタンスをJSON形式に変換します。`freezed`パッケージによって自動生成されます。

**戻り値:**
- `Map<String, dynamic>` - JSON形式のマップ

## 関連する列挙型

### PdfOrientation 列挙型

```dart
enum PdfOrientation {
  portrait,  // 縦向き
  landscape,  // 横向き
}
```

## PdfPageFormatの拡張メソッド

`PdfPageFormat`クラスは`pdf`パッケージから提供されるクラスですが、JSONシリアライズのために以下の拡張メソッドを定義します。

```dart
extension PdfPageFormatJson on PdfPageFormat {
  static PdfPageFormat fromString(String format) {
    switch (format) {
      case 'a4':
        return PdfPageFormat.a4;
      case 'a3':
        return PdfPageFormat.a3;
      case 'letter':
        return PdfPageFormat.letter;
      default:
        return PdfPageFormat.a4;
    }
  }
  
  String toFormatString() {
    if (this == PdfPageFormat.a4) {
      return 'a4';
    } else if (this == PdfPageFormat.a3) {
      return 'a3';
    } else if (this == PdfPageFormat.letter) {
      return 'letter';
    } else {
      return 'custom';
    }
  }
}
```

## 使用例

```dart
// デフォルト設定でのインスタンス作成
final defaultSettings = PdfSettings(
  title: '家系図',
);

// カスタム設定でのインスタンス作成
final customSettings = PdfSettings(
  title: '山田家の家系図',
  pageFormat: PdfPageFormat.a3,
  orientation: PdfOrientation.landscape,
  showDate: true,
  showLegend: true,
  fitToSinglePage: true,
);

// 設定の更新（freezedのcopyWithを使用）
final updatedSettings = customSettings.copyWith(
  fitToSinglePage: false,
);

// JSON形式への変換
final jsonData = customSettings.toJson();

// JSONからの復元
final restoredSettings = PdfSettings.fromJson(jsonData);
```

## 注意事項

- `PdfPageFormat`は`pdf`パッケージから提供されるクラスで、A4、A3、レターサイズなどのページフォーマットを表します。
- `orientation`が`PdfOrientation.landscape`の場合、ページはA4横向きなどで出力されます。
- `fitToSinglePage`が`true`の場合、家系図は1ページに収まるようにスケールされます。これにより大きな家系図では各ノードが小さくなる場合があります。
- `showDate`と`showLegend`は、出力PDFに日付や凡例を含めるかどうかを制御します。
- このクラスは`freezed`パッケージを使用した不変（イミュータブル）クラスです。値を変更する場合は`copyWith`メソッドを使用します。

## PDF出力サービスとの連携

このクラスは`FamilyTreePdfGenerator`クラスと連携して使用されます。PDF生成サービスはこの設定に基づいてPDFの出力形式を決定します。

```dart
// PdfSettingsとの連携例
final pdfData = await pdfGenerator.generatePdf(
  familyTreeData: familyTree,
  settings: pdfSettings,
);
```
