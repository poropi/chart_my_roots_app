# Relationship クラス

## 概要

`Relationship` クラスは家系図内の人物間の関係性を表すモデルクラスです。親子関係や配偶者関係などの関係タイプと、関係を持つ二人の人物のIDを管理します。Firestoreとの連携に必要なシリアライズ/デシリアライズ機能も提供します。

## クラス定義

```dart
@freezed
class Relationship with _$Relationship {
  const factory Relationship({
    required String id,
    required String fromPersonId,
    required String toPersonId,
    required RelationType type,
  }) = _Relationship;

  factory Relationship.fromJson(Map<String, dynamic> json) => _$RelationshipFromJson(json);
  
  factory Relationship.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Relationship.fromJson({
      'id': doc.id,
      ...data,
      'type': data['type'], // enumの変換はg.dartで自動生成
    });
  }
  
  Map<String, dynamic> toFirestore() {
    final json = toJson();
    json.remove('id'); // IDはドキュメントIDとして使用するため削除
    return json;
  }
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `id` | `String` | 関係性の一意識別子 | なし（必須） |
| `fromPersonId` | `String` | 関係元の人物ID | なし（必須） |
| `toPersonId` | `String` | 関係先の人物ID | なし（必須） |
| `type` | `RelationType` | 関係タイプ（親子/配偶者） | なし（必須） |

## メソッド

### Relationship.fromJson

```dart
factory Relationship.fromJson(Map<String, dynamic> json)
```

JSONオブジェクトから`Relationship`インスタンスを生成します。`freezed`パッケージによって自動生成されます。

**パラメータ:**
- `json`: `Map<String, dynamic>` - 変換元のJSONオブジェクト

**戻り値:**
- `Relationship` - 生成された`Relationship`インスタンス

### Relationship.fromFirestore

```dart
factory Relationship.fromFirestore(DocumentSnapshot doc)
```

Firestore `DocumentSnapshot`から`Relationship`インスタンスを生成します。

**パラメータ:**
- `doc`: `DocumentSnapshot` - Firestoreから取得したドキュメント

**戻り値:**
- `Relationship` - 生成された`Relationship`インスタンス

### toFirestore

```dart
Map<String, dynamic> toFirestore()
```

`Relationship`インスタンスをFirestoreに保存可能な形式に変換します。

**戻り値:**
- `Map<String, dynamic>` - Firestoreに保存可能な形式のマップ

## 使用例

```dart
// 親子関係の作成
final parentChildRelationship = Relationship(
  id: '',  // 新規作成時は空文字列、Firestoreが自動生成
  fromPersonId: 'parent-id',  // 親の人物ID
  toPersonId: 'child-id',     // 子の人物ID
  type: RelationType.parentChild,
);

// 配偶者関係の作成
final spouseRelationship = Relationship(
  id: '',
  fromPersonId: 'person1-id',
  toPersonId: 'person2-id',
  type: RelationType.spouse,
);

// Firestoreへの保存用データの取得
final firestoreData = parentChildRelationship.toFirestore();

// Firestoreから取得したデータの変換
final retrievedRelationship = Relationship.fromFirestore(docSnapshot);

// 値の変更（freezedのcopyWithを使用）
final updatedRelationship = parentChildRelationship.copyWith(
  type: RelationType.spouse,
);
```

## 注意事項

- `id`プロパティは、FirestoreのドキュメントIDに対応します。新規作成時は空文字列で、Firestoreに保存後に自動生成されたIDが設定されます。
- 親子関係（`RelationType.parentChild`）の場合、`fromPersonId`が親、`toPersonId`が子を表します。
- 配偶者関係（`RelationType.spouse`）の場合、`fromPersonId`と`toPersonId`の順序に意味はありませんが、一貫性のために同じ関係を重複して保存しないよう注意してください。
- このクラスは`freezed`パッケージを使用した不変（イミュータブル）クラスです。値を変更する場合は`copyWith`メソッドを使用します。

## 関連するクラス・列挙型

### RelationType 列挙型

```dart
enum RelationType {
  parentChild,  // 親子関係
  spouse,       // 配偶者関係
}
```

## 循環参照に関する注意

親子関係を設定する際は、循環参照が発生しないように注意してください。たとえば、A→B→C→Aのような循環的な親子関係を作成するとレイアウトエンジンが正常に動作しなくなる可能性があります。アプリケーションレベルでこのような循環参照を検出して防止する必要があります。
