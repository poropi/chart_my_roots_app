# ExportImportService クラス

## 概要

`ExportImportService` クラスは、家系図データのJSON形式でのエクスポートおよびインポート処理を担当するサービスクラスです。`FamilyTreeStorageRepository` と連携してデータの永続化層とやり取りし、ファイル操作やデータ検証などの機能を提供します。

## クラス定義

```dart
// lib/services/export_import_service.dart

class ExportImportService {
  final FamilyTreeStorageRepository _repository;
  
  ExportImportService(this._repository);
  
  // JSONファイルとしてエクスポート (Web/モバイル対応)
  Future<void> exportToJson(String treeId, String fileName) async {
    try {
      // 家系図データを取得
      final exportData = await _repository.exportFamilyTree(treeId);
      
      // JSONに変換
      final jsonString = jsonEncode(exportData.toJson());
      final bytes = utf8.encode(jsonString);
      
      if (kIsWeb) {
        // Webの場合はブラウザのダウンロード機能を使用
        final blob = html.Blob([bytes], 'application/json');
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
        await file.writeAsBytes(bytes);
        
        // 共有ダイアログを表示
        await Share.shareXFiles(
          [XFile(path)],
          text: '家系図データ: $fileName',
          subject: fileName,
        );
      }
    } catch (e) {
      // エラーログ記録
      _logError('Failed to export to JSON', e);
      rethrow;
    }
  }
  
  // JSON文字列からインポート
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
      // エラーログ記録
      _logError('Failed to import from JSON', e);
      rethrow;
    }
  }
  
  // JSONデータの構造を検証
  void _validateJsonStructure(Map<String, dynamic> json) {
    if (!json.containsKey('metadata')) {
      throw FormatException('JSONデータにメタデータが含まれていません');
    }
    final metadata = json['metadata'] as Map<String, dynamic>?;
    if (metadata == null || 
        !metadata.containsKey('version') || 
        !metadata.containsKey('exportDate') || 
        !metadata.containsKey('appName')) {
      throw FormatException('メタデータの形式が不正です');
    }
    
    if (!json.containsKey('familyTree')) {
      throw FormatException('JSONデータに家系図データが含まれていません');
    }
    final familyTree = json['familyTree'] as Map<String, dynamic>?;
    if (familyTree == null || 
        !familyTree.containsKey('id') || 
        !familyTree.containsKey('name') || 
        !familyTree.containsKey('persons') || 
        !familyTree.containsKey('relationships') || 
        !familyTree.containsKey('lastModified')) {
      throw FormatException('家系図データの形式が不正です');
    }
    
    // さらに詳細な型チェックや必須フィールドの存在チェックもここに追加可能
  }
  
  // エラーログ記録（実際の実装ではロギングライブラリを使用）
  void _logError(String message, dynamic error) {
    print('ERROR (ExportImportService): $message - $error');
  }
}
```

## プロバイダ

```dart
// lib/services/export_import_service_provider.dart

final exportImportServiceProvider = Provider<ExportImportService>((ref) {
  final repository = ref.watch(familyTreeStorageRepositoryProvider);
  return ExportImportService(repository);
});
```

## メソッド詳細

### exportToJson

```dart
Future<void> exportToJson(String treeId, String fileName) async
```

指定された家系図IDのデータをJSONファイルとしてエクスポートします。Web環境ではブラウザのダウンロード機能を使用し、モバイル/デスクトップ環境では一時ファイルに保存して共有ダイアログを表示します。

**パラメータ:**
- `treeId`: `String` - エクスポートする家系図のID
- `fileName`: `String` - 保存するファイル名（例: `my_family_tree.json`）

**戻り値:**
- `Future<void>` - エクスポート処理の完了を示すFuture

**例外:**
- リポジトリでのデータ取得に失敗した場合やファイル操作に失敗した場合、例外がスローされます。

### importFromJson

```dart
Future<String> importFromJson(String jsonString, ImportMode mode) async
```

JSON文字列から家系図データをインポートします。インポート前にJSONデータの構造を検証します。

**パラメータ:**
- `jsonString`: `String` - インポートするJSONデータ文字列
- `mode`: `ImportMode` - インポートモード（新規作成/マージ/上書き）

**戻り値:**
- `Future<String>` - インポートされた家系図のID

**例外:**
- JSONのパースに失敗した場合やデータ構造が不正な場合、`FormatException`がスローされます。
- リポジトリでのインポート処理に失敗した場合、例外がスローされます。

## 使用例

```dart
// サービスの取得
final service = ref.read(exportImportServiceProvider);

// 家系図のエクスポート
final String treeIdToExport = 'some-tree-id';
final String exportFileName = 'my_family_tree_export.json';
try {
  await service.exportToJson(treeIdToExport, exportFileName);
  print('エクスポートが完了しました。');
} catch (e) {
  print('エクスポートエラー: $e');
}

// ファイルからJSON文字列を読み込む（例）
// final String jsonStringFromFile = await loadJsonStringFromFilePicker();

// 家系図のインポート（新規作成モード）
final String jsonToImport = '{"metadata":{...},"familyTree":{...}}'; // 実際のJSONデータ
try {
  final importedTreeId = await service.importFromJson(jsonToImport, ImportMode.create);
  print('インポートが完了しました。新しい家系図ID: $importedTreeId');
} catch (e) {
  print('インポートエラー: $e');
}
```

## 注意事項

- このサービスクラスは、エクスポート・インポート機能のビジネスロジックを担当します。
- Web環境とモバイル/デスクトップ環境でファイル操作の方法が異なるため、`kIsWeb`を使用してプラットフォームを判定しています。
- モバイル/デスクトップ環境でのファイル保存と共有には、`path_provider`および`share_plus`パッケージの利用を想定しています。これらのパッケージは`pubspec.yaml`に追加する必要があります。
- JSONデータのバリデーションは、インポート処理の堅牢性を高めるために重要です。`_validateJsonStructure`メソッドで基本的な構造チェックを行っていますが、必要に応じてより詳細な検証ルールを追加してください。
- エラーハンドリングは、呼び出し元で適切に行うか、このサービスクラス内でユーザーフレンドリーなエラーメッセージに変換してUI層に伝える必要があります。

## 関連するクラス・インターフェース

- [FamilyTreeStorageRepository](../リポジトリ/FamilyTreeStorageRepository.md) - 家系図データ全体の永続化と取得を担当するリポジトリ
- [ExportData](../モデル/ExportData.md) - エクスポート・インポート用のデータ構造
- [ImportMode](../リポジトリ/FamilyTreeStorageRepository.md) - インポートモードを定義する列挙型
