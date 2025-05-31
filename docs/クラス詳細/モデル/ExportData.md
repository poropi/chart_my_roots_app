# ExportData クラス

## 概要

`ExportData` クラスは家系図データをエクスポート・インポートする際のルート構造を表すモデルクラスです。メタデータ（バージョン、エクスポート日時など）と家系図データを含みます。JSONファイルとしてエクスポートしたり、インポートしたりする際に使用されます。

## クラス定義

```dart
@freezed
class ExportData with _$ExportData {
  const factory ExportData({
    required ExportMetadata metadata,
    required FamilyTreeData familyTree,
  }) = _ExportData;

  factory ExportData.fromJson(Map<String, dynamic> json) => _$ExportDataFromJson(json);
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `metadata` | `ExportMetadata` | エクスポートのメタデータ | なし（必須） |
| `familyTree` | `FamilyTreeData` | エクスポートする家系図データ | なし（必須） |

## メソッド

### ExportData.fromJson

```dart
factory ExportData.fromJson(Map<String, dynamic> json)
```

JSONオブジェクトから`ExportData`インスタンスを生成します。`freezed`パッケージによって自動生成されます。

**パラメータ:**
- `json`: `Map<String, dynamic>` - 変換元のJSONオブジェクト

**戻り値:**
- `ExportData` - 生成された`ExportData`インスタンス

### toJson

```dart
Map<String, dynamic> toJson()
```

`ExportData`インスタンスをJSON形式に変換します。`freezed`パッケージによって自動生成されます。

**戻り値:**
- `Map<String, dynamic>` - JSON形式のマップ

## 関連するクラス

### ExportMetadata クラス

`ExportMetadata`クラスはエクスポートデータのメタデータを表します。

```dart
@freezed
class ExportMetadata with _$ExportMetadata {
  const factory ExportMetadata({
    required String version,
    required DateTime exportDate,
    required String appName,
  }) = _ExportMetadata;

  factory ExportMetadata.fromJson(Map<String, dynamic> json) => _$ExportMetadataFromJson(json);
}
```

#### プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `version` | `String` | エクスポートフォーマットのバージョン | なし（必須） |
| `exportDate` | `DateTime` | エクスポート日時 | なし（必須） |
| `appName` | `String` | アプリケーション名 | なし（必須） |

## 使用例

```dart
// 新しいエクスポートデータの作成
final exportData = ExportData(
  metadata: ExportMetadata(
    version: '1.0',
    exportDate: DateTime.now(),
    appName: 'ChartMyRoots',
  ),
  familyTree: familyTreeData,
);

// JSON形式への変換
final jsonString = jsonEncode(exportData.toJson());

// ファイルへの書き込み（Web環境の例）
final blob = html.Blob([jsonString], 'application/json');
final url = html.Url.createObjectUrlFromBlob(blob);
final anchor = html.AnchorElement(href: url)
  ..setAttribute('download', 'family_tree_export.json')
  ..click();
html.Url.revokeObjectUrl(url);

// JSONからの復元
final importedData = ExportData.fromJson(jsonDecode(jsonString));
```

## 実装例：インポート機能

```dart
Future<String> importFromJson(String jsonString, ImportMode mode) async {
  try {
    // JSONをパース
    final Map<String, dynamic> jsonData = jsonDecode(jsonString);
    
    // バリデーション
    _validateJsonStructure(jsonData);
    
    // ExportDataオブジェクトに変換
    final exportData = ExportData.fromJson(jsonData);
    
    // インポート処理を実行
    return await _repository.importFamilyTree(exportData, mode);
  } catch (e) {
    rethrow;
  }
}

// JSONデータの構造を検証
void _validateJsonStructure(Map<String, dynamic> json) {
  if (!json.containsKey('metadata')) {
    throw FormatException('メタデータが見つかりません');
  }
  
  if (!json.containsKey('familyTree')) {
    throw FormatException('家系図データが見つかりません');
  }
  
  final familyTree = json['familyTree'] as Map<String, dynamic>;
  
  if (!familyTree.containsKey('persons')) {
    throw FormatException('人物データが見つかりません');
  }
  
  if (!familyTree.containsKey('relationships')) {
    throw FormatException('関係性データが見つかりません');
  }
}
```

## 注意事項

- エクスポート・インポート機能は、家系図データのバックアップやデバイス間の移行に使用されます。
- バージョン情報は、将来的なフォーマット変更があった場合の互換性確保に重要です。
- JSONファイルのエクスポート時には、日付やタイムスタンプが適切に変換されるよう注意してください。
- インポート時には、JSONデータの構造が正しいかのバリデーションを必ず行ってください。
- このクラスは`freezed`パッケージを使用した不変（イミュータブル）クラスです。

## 関連するクラス

- [FamilyTreeData](FamilyTreeData.md) - 家系図全体のデータを管理するクラス
- [Person](Person.md) - 家系図内の人物を表すクラス
- [Relationship](Relationship.md) - 人物間の関係性を表すクラス
