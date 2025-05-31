# FamilyTreeData クラス

## 概要

`FamilyTreeData` クラスは家系図データ全体を表すモデルクラスです。家系図の基本情報（ID、名前、最終更新日時）と、含まれる人物（Person）および関係性（Relationship）のコレクションを管理します。データのエクスポート・インポート機能での利用や、アプリ内での一括操作に使用されます。

## クラス定義

```dart
@freezed
class FamilyTreeData with _$FamilyTreeData {
  const factory FamilyTreeData({
    required String id,
    required String name,
    @Default([]) List<Person> persons,
    @Default([]) List<Relationship> relationships,
    required DateTime lastModified,
  }) = _FamilyTreeData;

  factory FamilyTreeData.fromJson(Map<String, dynamic> json) => _$FamilyTreeDataFromJson(json);
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `id` | `String` | 家系図の一意識別子 | なし（必須） |
| `name` | `String` | 家系図の名前 | なし（必須） |
| `persons` | `List<Person>` | 家系図に含まれる人物リスト | `[]` |
| `relationships` | `List<Relationship>` | 家系図に含まれる関係性リスト | `[]` |
| `lastModified` | `DateTime` | 最終更新日時 | なし（必須） |

## メソッド

### FamilyTreeData.fromJson

```dart
factory FamilyTreeData.fromJson(Map<String, dynamic> json)
```

JSONオブジェクトから`FamilyTreeData`インスタンスを生成します。`freezed`パッケージによって自動生成されます。

**パラメータ:**
- `json`: `Map<String, dynamic>` - 変換元のJSONオブジェクト

**戻り値:**
- `FamilyTreeData` - 生成された`FamilyTreeData`インスタンス

### toJson

```dart
Map<String, dynamic> toJson()
```

`FamilyTreeData`インスタンスをJSON形式に変換します。`freezed`パッケージによって自動生成されます。

**戻り値:**
- `Map<String, dynamic>` - JSON形式のマップ

## 拡張メソッド（Extension Methods）

以下の拡張メソッドを定義して、家系図データの操作を容易にします。

```dart
extension FamilyTreeDataExtension on FamilyTreeData {
  // 特定の人物を取得
  Person? findPersonById(String personId) {
    return persons.firstWhereOrNull((p) => p.id == personId);
  }
  
  // 特定の関係性を取得
  Relationship? findRelationshipById(String relationshipId) {
    return relationships.firstWhereOrNull((r) => r.id == relationshipId);
  }
  
  // 特定の人物に関連する関係性を取得
  List<Relationship> getRelationshipsForPerson(String personId) {
    return relationships.where(
      (rel) => rel.fromPersonId == personId || rel.toPersonId == personId
    ).toList();
  }
  
  // 特定の人物の親を取得
  List<Person> getParentsOfPerson(String personId) {
    final parentIds = relationships
        .where((rel) => 
            rel.type == RelationType.parentChild && 
            rel.toPersonId == personId)
        .map((rel) => rel.fromPersonId)
        .toList();
    
    return persons
        .where((person) => parentIds.contains(person.id))
        .toList();
  }
  
  // 特定の人物の子を取得
  List<Person> getChildrenOfPerson(String personId) {
    final childrenIds = relationships
        .where((rel) => 
            rel.type == RelationType.parentChild && 
            rel.fromPersonId == personId)
        .map((rel) => rel.toPersonId)
        .toList();
    
    return persons
        .where((person) => childrenIds.contains(person.id))
        .toList();
  }
  
  // 特定の人物の配偶者を取得
  List<Person> getSpousesOfPerson(String personId) {
    final spouseIds = relationships
        .where((rel) => 
            rel.type == RelationType.spouse && 
            (rel.fromPersonId == personId || rel.toPersonId == personId))
        .map((rel) => 
            rel.fromPersonId == personId ? rel.toPersonId : rel.fromPersonId)
        .toList();
    
    return persons
        .where((person) => spouseIds.contains(person.id))
        .toList();
  }
  
  // 最新の更新日時で家系図を更新
  FamilyTreeData withUpdatedTimestamp() {
    return copyWith(lastModified: DateTime.now());
  }
}
```

## 使用例

```dart
// 新しい家系図の作成
final familyTree = FamilyTreeData(
  id: 'tree-id',
  name: '山田家',
  persons: [person1, person2, person3],
  relationships: [relationship1, relationship2],
  lastModified: DateTime.now(),
);

// 家系図データのJSON形式への変換
final jsonData = familyTree.toJson();

// JSONからの復元
final restoredFamilyTree = FamilyTreeData.fromJson(jsonData);

// 特定の人物を検索
final person = familyTree.findPersonById('person-id');

// 特定の人物の親を取得
final parents = familyTree.getParentsOfPerson('person-id');

// 家系図データの更新（freezedのcopyWithを使用）
final updatedFamilyTree = familyTree.copyWith(
  name: '山田家の歴史',
).withUpdatedTimestamp();
```

## 注意事項

- `persons`と`relationships`リストは、アプリケーション内のデータ操作やエクスポート/インポート時に使用されます。通常のデータ保存は各オブジェクトを個別にFirestoreに保存します。
- このクラスは`freezed`パッケージを使用した不変（イミュータブル）クラスです。値を変更する場合は`copyWith`メソッドを使用します。
- `lastModified`は、家系図の更新日時を追跡するために使用され、複数デバイス間での同期時に利用できます。
- 拡張メソッドは、家系図データの操作を便利にするためのものですが、大規模な家系図の場合はパフォーマンスに注意してください。必要に応じてインデックス付きのマップ構造に変換するなどの最適化を検討してください。

## 関連するクラス

- [Person](Person.md) - 家系図内の人物を表すクラス
- [Relationship](Relationship.md) - 人物間の関係性を表すクラス
- [ExportData](ExportData.md) - エクスポートデータ全体を表すクラス
