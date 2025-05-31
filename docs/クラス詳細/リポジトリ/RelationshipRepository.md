# RelationshipRepository クラス

## 概要

`RelationshipRepository` インターフェースと `FirestoreRelationshipRepository` 実装クラスは、人物間の関係性（親子関係や配偶者関係）データのCRUD操作を提供します。関係性データの永続化と取得を担当し、Firestoreとのデータやりとりをカプセル化します。

## インターフェース定義

```dart
// lib/repositories/relationship_repository.dart

abstract class RelationshipRepository {
  // 全ての関係性情報を取得
  Stream<List<Relationship>> getRelationships();
  
  // 特定の関係性情報を取得
  Stream<Relationship?> getRelationship(String id);
  
  // 一度だけ全ての関係性情報を取得
  Future<List<Relationship>> getRelationshipsOnce();
  
  // 特定の人物に関連する関係性を取得
  Stream<List<Relationship>> getRelationshipsForPerson(String personId);
  
  // 関係性情報を追加
  Future<String> addRelationship(Relationship relationship);
  
  // 関係性情報を更新
  Future<void> updateRelationship(Relationship relationship);
  
  // 関係性情報を削除
  Future<void> deleteRelationship(String id);
  
  // 特定の人物に関連する全ての関係性を削除
  Future<void> deleteRelationshipsForPerson(String personId);
}
```

## 実装クラス

```dart
// lib/repositories/firestore_relationship_repository.dart

class FirestoreRelationshipRepository implements RelationshipRepository {
  final FirebaseFirestore _firestore;
  final String _userId;
  final String _treeId;
  
  FirestoreRelationshipRepository({
    required FirebaseFirestore firestore,
    required String userId,
    required String treeId,
  }) : _firestore = firestore,
       _userId = userId,
       _treeId = treeId;
  
  // コレクションへの参照を取得
  CollectionReference<Map<String, dynamic>> get _collection =>
      _firestore.collection('users/$_userId/familyTrees/$_treeId/relationships');
  
  @override
  Stream<List<Relationship>> getRelationships() {
    return _collection
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => Relationship.fromFirestore(doc))
            .toList());
  }
  
  @override
  Stream<Relationship?> getRelationship(String id) {
    return _collection
        .doc(id)
        .snapshots()
        .map((doc) => doc.exists ? Relationship.fromFirestore(doc) : null);
  }
  
  @override
  Future<List<Relationship>> getRelationshipsOnce() async {
    final snapshot = await _collection.get();
    return snapshot.docs
        .map((doc) => Relationship.fromFirestore(doc))
        .toList();
  }
  
  @override
  Stream<List<Relationship>> getRelationshipsForPerson(String personId) {
    return _collection
        .where(Filter.or(
          Filter('fromPersonId', isEqualTo: personId),
          Filter('toPersonId', isEqualTo: personId)
        ))
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => Relationship.fromFirestore(doc))
            .toList());
  }
  
  @override
  Future<String> addRelationship(Relationship relationship) async {
    // バリデーション
    await _validateRelationship(relationship);
    
    // 重複チェック
    final exists = await _checkDuplicateRelationship(relationship);
    if (exists) {
      throw Exception('この2人の間には既に同じ種類の関係が設定されています');
    }
    
    // 循環参照チェック（親子関係の場合）
    if (relationship.type == RelationType.parentChild) {
      final hasCircular = await _checkCircularParentChildRelationship(
        relationship.fromPersonId,
        relationship.toPersonId,
      );
      if (hasCircular) {
        throw Exception('循環する親子関係は設定できません');
      }
    }
    
    // 新規ドキュメントを追加
    final docRef = await _collection.add(relationship.toFirestore());
    return docRef.id;
  }
  
  @override
  Future<void> updateRelationship(Relationship relationship) async {
    // バリデーション
    await _validateRelationship(relationship);
    
    // ドキュメントの存在確認
    final docRef = _collection.doc(relationship.id);
    final doc = await docRef.get();
    if (!doc.exists) {
      throw Exception('更新対象の関係性が見つかりません: ${relationship.id}');
    }
    
    // 重複チェック（自分自身は除く）
    final exists = await _checkDuplicateRelationship(relationship, excludeId: relationship.id);
    if (exists) {
      throw Exception('この2人の間には既に同じ種類の関係が設定されています');
    }
    
    // 循環参照チェック（親子関係の場合）
    if (relationship.type == RelationType.parentChild) {
      final hasCircular = await _checkCircularParentChildRelationship(
        relationship.fromPersonId,
        relationship.toPersonId,
      );
      if (hasCircular) {
        throw Exception('循環する親子関係は設定できません');
      }
    }
    
    // ドキュメントを更新
    await docRef.update(relationship.toFirestore());
  }
  
  @override
  Future<void> deleteRelationship(String id) async {
    // ドキュメントの存在確認
    final docRef = _collection.doc(id);
    final doc = await docRef.get();
    if (!doc.exists) {
      throw Exception('削除対象の関係性が見つかりません: $id');
    }
    
    // ドキュメントを削除
    await docRef.delete();
  }
  
  @override
  Future<void> deleteRelationshipsForPerson(String personId) async {
    // 人物に関連する関係性を取得
    final queryFrom = await _collection
        .where('fromPersonId', isEqualTo: personId)
        .get();
    
    final queryTo = await _collection
        .where('toPersonId', isEqualTo: personId)
        .get();
    
    // バッチ処理で効率的に削除
    final batch = _firestore.batch();
    
    for (var doc in [...queryFrom.docs, ...queryTo.docs]) {
      batch.delete(doc.reference);
    }
    
    await batch.commit();
  }
  
  // バリデーション
  Future<void> _validateRelationship(Relationship relationship) async {
    // 関係元の人物が存在するか確認
    final fromPersonRef = _firestore
        .collection('users/$_userId/familyTrees/$_treeId/persons')
        .doc(relationship.fromPersonId);
    final fromPersonDoc = await fromPersonRef.get();
    if (!fromPersonDoc.exists) {
      throw Exception('関係元の人物が見つかりません: ${relationship.fromPersonId}');
    }
    
    // 関係先の人物が存在するか確認
    final toPersonRef = _firestore
        .collection('users/$_userId/familyTrees/$_treeId/persons')
        .doc(relationship.toPersonId);
    final toPersonDoc = await toPersonRef.get();
    if (!toPersonDoc.exists) {
      throw Exception('関係先の人物が見つかりません: ${relationship.toPersonId}');
    }
    
    // 自己参照をチェック
    if (relationship.fromPersonId == relationship.toPersonId) {
      throw Exception('自分自身との関係は設定できません');
    }
  }
  
  // 重複する関係性をチェック
  Future<bool> _checkDuplicateRelationship(
    Relationship relationship, {
    String? excludeId,
  }) async {
    // 同じ種類の関係を検索
    final query = _collection.where('type', isEqualTo: relationship.type.toString());
    final snapshot = await query.get();
    
    return snapshot.docs.any((doc) {
      // 自分自身は除外
      if (excludeId != null && doc.id == excludeId) {
        return false;
      }
      
      final rel = Relationship.fromFirestore(doc);
      
      // 親子関係の場合、方向が重要
      if (relationship.type == RelationType.parentChild) {
        return rel.fromPersonId == relationship.fromPersonId &&
               rel.toPersonId == relationship.toPersonId;
      }
      
      // 配偶者関係の場合、方向は関係ない
      return (rel.fromPersonId == relationship.fromPersonId &&
              rel.toPersonId == relationship.toPersonId) ||
             (rel.fromPersonId == relationship.toPersonId &&
              rel.toPersonId == relationship.fromPersonId);
    });
  }
  
  // 循環する親子関係をチェック
  Future<bool> _checkCircularParentChildRelationship(
    String fromPersonId,
    String toPersonId,
  ) async {
    // すべての親子関係を取得
    final query = _collection.where('type', isEqualTo: RelationType.parentChild.toString());
    final snapshot = await query.get();
    final relationships = snapshot.docs.map((doc) => Relationship.fromFirestore(doc)).toList();
    
    // fromPersonId（親）の先祖にtoPersonId（子）が含まれるかをチェック
    final ancestorIds = <String>{};
    _findAncestors(fromPersonId, relationships, ancestorIds);
    
    return ancestorIds.contains(toPersonId);
  }
  
  // 先祖を再帰的に探索する補助メソッド
  void _findAncestors(
    String personId,
    List<Relationship> parentChildRelationships,
    Set<String> ancestorIds,
  ) {
    // 親を探す
    final parentRelations = parentChildRelationships.where(
      (rel) => rel.toPersonId == personId && rel.type == RelationType.parentChild,
    );
    
    for (final relation in parentRelations) {
      final parentId = relation.fromPersonId;
      ancestorIds.add(parentId);
      _findAncestors(parentId, parentChildRelationships, ancestorIds);
    }
  }
}
```

## プロバイダ

```dart
// リポジトリプロバイダ
final relationshipRepositoryProvider = Provider<RelationshipRepository>((ref) {
  final firestore = ref.watch(firestoreProvider);
  final userId = ref.watch(userIdProvider);
  final treeId = ref.watch(selectedTreeIdProvider);
  
  if (userId == null || treeId == null) {
    throw UnimplementedError('リポジトリの初期化に必要な情報が不足しています');
  }
  
  return FirestoreRelationshipRepository(
    firestore: firestore,
    userId: userId,
    treeId: treeId,
  );
});
```

## メソッド詳細

### getRelationships

```dart
Stream<List<Relationship>> getRelationships()
```

家系図に登録されているすべての関係性情報をリアルタイムで取得します。

**戻り値:**
- `Stream<List<Relationship>>` - 関係性リストのストリーム。Firestoreのデータが更新されると自動的に新しい値が流れます。

### getRelationship

```dart
Stream<Relationship?> getRelationship(String id)
```

指定されたIDの関係性情報をリアルタイムで取得します。

**パラメータ:**
- `id`: `String` - 取得する関係性のID

**戻り値:**
- `Stream<Relationship?>` - 関係性情報のストリーム。該当する関係性が存在しない場合は`null`が流れます。

### getRelationshipsOnce

```dart
Future<List<Relationship>> getRelationshipsOnce()
```

家系図に登録されているすべての関係性情報を一度だけ取得します。

**戻り値:**
- `Future<List<Relationship>>` - 関係性リストを含むFuture

### getRelationshipsForPerson

```dart
Stream<List<Relationship>> getRelationshipsForPerson(String personId)
```

特定の人物に関連するすべての関係性（親子関係、配偶者関係）をリアルタイムで取得します。

**パラメータ:**
- `personId`: `String` - 関連する関係性を取得したい人物のID

**戻り値:**
- `Stream<List<Relationship>>` - 人物に関連する関係性リストのストリーム

### addRelationship

```dart
Future<String> addRelationship(Relationship relationship)
```

新しい関係性情報を追加します。重複する関係性や循環する親子関係が検出された場合は例外をスローします。

**パラメータ:**
- `relationship`: `Relationship` - 追加する関係性データ。`id`プロパティは空文字列にしておき、Firestoreが自動生成したIDが返されます。

**戻り値:**
- `Future<String>` - 追加された関係性の新しいID

**例外:**
- 関係元または関係先の人物が存在しない場合
- 同じ人物間に同じ種類の関係が既に存在する場合
- 循環する親子関係となる場合
- 自己参照関係の場合

### updateRelationship

```dart
Future<void> updateRelationship(Relationship relationship)
```

既存の関係性情報を更新します。重複する関係性や循環する親子関係が検出された場合は例外をスローします。

**パラメータ:**
- `relationship`: `Relationship` - 更新する関係性データ。`id`プロパティには既存の関係性IDを指定する必要があります。

**戻り値:**
- `Future<void>` - 更新が完了したことを示すFuture

**例外:**
- 関係元または関係先の人物が存在しない場合
- 同じ人物間に同じ種類の関係が既に存在する場合
- 循環する親子関係となる場合
- 自己参照関係の場合
- 指定されたIDの関係性が存在しない場合

### deleteRelationship

```dart
Future<void> deleteRelationship(String id)
```

指定されたIDの関係性を削除します。

**パラメータ:**
- `id`: `String` - 削除する関係性のID

**戻り値:**
- `Future<void>` - 削除が完了したことを示すFuture

**例外:**
- 指定されたIDの関係性が存在しない場合

### deleteRelationshipsForPerson

```dart
Future<void> deleteRelationshipsForPerson(String personId)
```

指定された人物に関連するすべての関係性を削除します。人物を削除する際の整合性確保に使用されます。

**パラメータ:**
- `personId`: `String` - 削除対象の関係性を持つ人物のID

**戻り値:**
- `Future<void>` - 削除が完了したことを示すFuture

## 使用例

```dart
// リポジトリの取得
final repository = ref.read(relationshipRepositoryProvider);

// すべての関係性を監視
final relationshipsStream = repository.getRelationships();
relationshipsStream.listen((relationships) {
  print('関係性リストが更新されました: ${relationships.length}件');
});

// 親子関係の追加
final parentChildRelationship = Relationship(
  id: '',
  fromPersonId: 'parent-id',
  toPersonId: 'child-id',
  type: RelationType.parentChild,
);
final newId = await repository.addRelationship(parentChildRelationship);

// 配偶者関係の追加
final spouseRelationship = Relationship(
  id: '',
  fromPersonId: 'person1-id',
  toPersonId: 'person2-id',
  type: RelationType.spouse,
);
await repository.addRelationship(spouseRelationship);

// 特定の人物に関連する関係性を取得
final personRelationships = repository.getRelationshipsForPerson('person-id');
personRelationships.listen((relationships) {
  print('人物に関連する関係性: ${relationships.length}件');
  for (var rel in relationships) {
    print('- タイプ: ${rel.type}, 相手: ${rel.fromPersonId == 'person-id' ? rel.toPersonId : rel.fromPersonId}');
  }
});

// 関係性の削除
await repository.deleteRelationship(newId);

// 人物に関連するすべての関係性を削除
await repository.deleteRelationshipsForPerson('person-id');
```

## 注意事項

- このリポジトリは、関係性データの永続化と取得に関する操作をカプセル化します。
- 重複する関係性や循環する親子関係を防止するためのバリデーションが組み込まれています。
- 親子関係では方向が重要（fromPersonIdが親、toPersonIdが子）ですが、配偶者関係では方向は重要ではありません。
- 関係性の削除は慎重に行う必要があります。特に、人物を削除する際は関連する関係性も必ず削除してください。
- バッチ処理を使用して、複数の関係性の削除を効率的に行います。
- Firestoreのインデックスを適切に設定し、クエリのパフォーマンスを最適化してください。

## 関連するクラス・インターフェース

- [Relationship](../モデル/Relationship.md) - 人物間の関係性を表すモデルクラス
- [RelationshipController](../コントローラ/RelationshipController.md) - 関係性操作を管理するコントローラクラス
- [PersonRepository](./PersonRepository.md) - 人物データを管理するリポジトリインターフェース
