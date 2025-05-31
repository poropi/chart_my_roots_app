# FamilyTreeStorageRepository クラス

## 概要

`FamilyTreeStorageRepository` インターフェースと `FirestoreFamilyTreeRepository` 実装クラスは、家系図データ全体（`FamilyTreeData`）の永続化と取得、およびエクスポート・インポート機能を提供します。Firestoreとのデータやりとりをカプセル化し、家系図のメタデータ、人物リスト、関係性リストを一括で管理します。

## インターフェース定義

```dart
// lib/repositories/family_tree_storage_repository.dart

abstract class FamilyTreeStorageRepository {
  // ユーザーが持つ全ての家系図情報を取得（リアルタイム）
  Stream<List<FamilyTreeData>> getFamilyTrees();
  
  // 特定の家系図情報を取得（リアルタイム）
  Stream<FamilyTreeData?> getFamilyTree(String treeId);
  
  // 新しい家系図を作成
  Future<String> createFamilyTree(String name);
  
  // 家系図情報を更新（人物・関係性も含む）
  Future<void> updateFamilyTree(FamilyTreeData familyTree);
  
  // 家系図を削除（関連する人物・関係性も全て削除）
  Future<void> deleteFamilyTree(String treeId);
  
  // 家系図データをエクスポート用に取得
  Future<ExportData> exportFamilyTree(String treeId);
  
  // 家系図データをインポート
  Future<String> importFamilyTree(ExportData exportData, ImportMode mode);
}

// インポートモードの列挙型
enum ImportMode {
  create,    // 新規作成
  merge,     // 既存データとマージ
  overwrite, // 既存データを上書き
}
```

## 実装クラス

```dart
// lib/repositories/firestore_family_tree_repository.dart

class FirestoreFamilyTreeRepository implements FamilyTreeStorageRepository {
  final FirebaseFirestore _firestore;
  final String _userId;
  
  FirestoreFamilyTreeRepository({
    required FirebaseFirestore firestore,
    required String userId,
  }) : _firestore = firestore,
       _userId = userId;
  
  // 家系図一覧のコレクション参照
  CollectionReference<Map<String, dynamic>> get _familyTreesCollection =>
      _firestore.collection('users/$_userId/familyTrees');
  
  // 特定の家系図のドキュメント参照
  DocumentReference<Map<String, dynamic>> _familyTreeDoc(String treeId) =>
      _familyTreesCollection.doc(treeId);
  
  // 特定の家系図内の人物コレクション参照
  CollectionReference<Map<String, dynamic>> _personsCollection(String treeId) =>
      _familyTreeDoc(treeId).collection('persons');
  
  // 特定の家系図内の関係性コレクション参照
  CollectionReference<Map<String, dynamic>> _relationshipsCollection(String treeId) =>
      _familyTreeDoc(treeId).collection('relationships');

  @override
  Stream<List<FamilyTreeData>> getFamilyTrees() {
    return _familyTreesCollection
        .orderBy('lastModified', descending: true)
        .snapshots()
        .asyncMap((snapshot) async {
          final List<FamilyTreeData> result = [];
          for (final doc in snapshot.docs) {
            final treeData = await _mapDocToFamilyTreeData(doc);
            if (treeData != null) {
              result.add(treeData);
            }
          }
          return result;
        });
  }

  @override
  Stream<FamilyTreeData?> getFamilyTree(String treeId) {
    return _familyTreeDoc(treeId)
        .snapshots()
        .asyncMap((snapshot) => _mapDocToFamilyTreeData(snapshot));
  }
  
  // ドキュメントスナップショットからFamilyTreeDataを構築するヘルパー
  Future<FamilyTreeData?> _mapDocToFamilyTreeData(DocumentSnapshot<Map<String, dynamic>> doc) async {
    if (!doc.exists) return null;
    
    final treeId = doc.id;
    final data = doc.data()!;
    
    final personsSnapshot = await _personsCollection(treeId).get();
    final persons = personsSnapshot.docs.map((d) => Person.fromFirestore(d)).toList();
    
    final relationshipsSnapshot = await _relationshipsCollection(treeId).get();
    final relationships = relationshipsSnapshot.docs.map((d) => Relationship.fromFirestore(d)).toList();
    
    return FamilyTreeData(
      id: treeId,
      name: data['name'] as String,
      persons: persons,
      relationships: relationships,
      lastModified: (data['lastModified'] as Timestamp).toDate(),
    );
  }

  @override
  Future<String> createFamilyTree(String name) async {
    if (name.trim().isEmpty) {
      throw Exception('家系図名は必須です');
    }
    final docRef = await _familyTreesCollection.add({
      'name': name,
      'lastModified': FieldValue.serverTimestamp(),
    });
    return docRef.id;
  }

  @override
  Future<void> updateFamilyTree(FamilyTreeData familyTree) async {
    if (familyTree.name.trim().isEmpty) {
      throw Exception('家系図名は必須です');
    }
    
    final batch = _firestore.batch();
    
    // 家系図メタデータを更新
    batch.set(
      _familyTreeDoc(familyTree.id),
      {
        'name': familyTree.name,
        'lastModified': FieldValue.serverTimestamp(),
      },
      SetOptions(merge: true), // 既存フィールドを保持しつつ更新
    );
    
    // 既存の人物・関係性IDを取得
    final existingPersonsSnapshot = await _personsCollection(familyTree.id).get();
    final existingPersonIds = existingPersonsSnapshot.docs.map((doc) => doc.id).toSet();
    
    final existingRelationshipsSnapshot = await _relationshipsCollection(familyTree.id).get();
    final existingRelationshipIds = existingRelationshipsSnapshot.docs.map((doc) => doc.id).toSet();
    
    // 人物データの更新/追加
    for (final person in familyTree.persons) {
      final personRef = _personsCollection(familyTree.id).doc(person.id.isEmpty ? null : person.id);
      batch.set(personRef, person.toFirestore());
      existingPersonIds.remove(person.id);
    }
    // 削除された人物を処理
    for (final idToDelete in existingPersonIds) {
      batch.delete(_personsCollection(familyTree.id).doc(idToDelete));
    }
    
    // 関係性データの更新/追加
    for (final relationship in familyTree.relationships) {
      final relationshipRef = _relationshipsCollection(familyTree.id).doc(relationship.id.isEmpty ? null : relationship.id);
      batch.set(relationshipRef, relationship.toFirestore());
      existingRelationshipIds.remove(relationship.id);
    }
    // 削除された関係性を処理
    for (final idToDelete in existingRelationshipIds) {
      batch.delete(_relationshipsCollection(familyTree.id).doc(idToDelete));
    }
    
    await batch.commit();
  }

  @override
  Future<void> deleteFamilyTree(String treeId) async {
    // バッチ処理で関連データを全て削除
    final batch = _firestore.batch();
    
    // 人物データの削除
    final personsSnapshot = await _personsCollection(treeId).get();
    for (final doc in personsSnapshot.docs) {
      batch.delete(doc.reference);
    }
    
    // 関係性データの削除
    final relationshipsSnapshot = await _relationshipsCollection(treeId).get();
    for (final doc in relationshipsSnapshot.docs) {
      batch.delete(doc.reference);
    }
    
    // 家系図ドキュメント自体を削除
    batch.delete(_familyTreeDoc(treeId));
    
    await batch.commit();
  }

  @override
  Future<ExportData> exportFamilyTree(String treeId) async {
    final familyTree = await getFamilyTree(treeId).first;
    if (familyTree == null) {
      throw Exception('エクスポート対象の家系図が見つかりません: $treeId');
    }
    
    final metadata = ExportMetadata(
      version: '1.0',
      exportDate: DateTime.now(),
      appName: 'ChartMyRoots',
    );
    
    return ExportData(
      metadata: metadata,
      familyTree: familyTree,
    );
  }

  @override
  Future<String> importFamilyTree(ExportData exportData, ImportMode mode) async {
    switch (mode) {
      case ImportMode.create:
        return _importAsNew(exportData);
      case ImportMode.merge:
        return _mergeWithExisting(exportData);
      case ImportMode.overwrite:
        return _overwriteExisting(exportData);
    }
  }
  
  // 新規作成モードでのインポート
  Future<String> _importAsNew(ExportData exportData) async {
    final newTreeId = await createFamilyTree(exportData.familyTree.name);
    final treeToImport = exportData.familyTree.copyWith(id: newTreeId);
    await updateFamilyTree(treeToImport); // 人物・関係性も保存
    return newTreeId;
  }
  
  // マージモードでのインポート
  Future<String> _mergeWithExisting(ExportData exportData) async {
    final treeId = exportData.familyTree.id;
    final existingTree = await getFamilyTree(treeId).first;
    
    if (existingTree == null) {
      // 存在しない場合は新規作成と同じ
      return _importAsNew(exportData);
    }
    
    // 人物と関係性をマージ
    final mergedPersons = List<Person>.from(existingTree.persons);
    final importedPersonIds = exportData.familyTree.persons.map((p) => p.id).toSet();
    mergedPersons.removeWhere((p) => importedPersonIds.contains(p.id)); // 重複を削除してインポートデータ優先
    mergedPersons.addAll(exportData.familyTree.persons);
    
    final mergedRelationships = List<Relationship>.from(existingTree.relationships);
    final importedRelationshipIds = exportData.familyTree.relationships.map((r) => r.id).toSet();
    mergedRelationships.removeWhere((r) => importedRelationshipIds.contains(r.id));
    mergedRelationships.addAll(exportData.familyTree.relationships);
    
    final mergedTree = existingTree.copyWith(
      name: exportData.familyTree.name, // 名前はインポートデータで上書き
      persons: mergedPersons,
      relationships: mergedRelationships,
      lastModified: DateTime.now(), // 更新日時を更新
    );
    
    await updateFamilyTree(mergedTree);
    return treeId;
  }
  
  // 上書きモードでのインポート
  Future<String> _overwriteExisting(ExportData exportData) async {
    final treeId = exportData.familyTree.id;
    final existingTree = await getFamilyTree(treeId).first;
    
    if (existingTree != null) {
      await deleteFamilyTree(treeId); // 既存を削除
    }
    // 新規作成と同じ処理
    return _importAsNew(exportData.copyWith(
      familyTree: exportData.familyTree.copyWith(id: '') // IDをリセットして新規作成
    ));
  }
}
```

## プロバイダ

```dart
// リポジトリプロバイダ
final familyTreeStorageRepositoryProvider = Provider<FamilyTreeStorageRepository>((ref) {
  final firestore = ref.watch(firestoreProvider);
  final userId = ref.watch(userIdProvider);
  
  if (userId == null) {
    throw UnimplementedError('リポジトリの初期化に必要なユーザーIDが不足しています');
  }
  
  return FirestoreFamilyTreeRepository(
    firestore: firestore,
    userId: userId,
  );
});
```

## メソッド詳細

### getFamilyTrees

```dart
Stream<List<FamilyTreeData>> getFamilyTrees()
```

ユーザーが持つ全ての家系図情報をリアルタイムで取得します。最終更新日時の降順でソートされます。

**戻り値:**
- `Stream<List<FamilyTreeData>>` - 家系図データリストのストリーム

### getFamilyTree

```dart
Stream<FamilyTreeData?> getFamilyTree(String treeId)
```

指定されたIDの家系図情報をリアルタイムで取得します。

**パラメータ:**
- `treeId`: `String` - 取得する家系図のID

**戻り値:**
- `Stream<FamilyTreeData?>` - 家系図データのストリーム（存在しない場合は`null`）

### createFamilyTree

```dart
Future<String> createFamilyTree(String name)
```

新しい家系図を作成します。

**パラメータ:**
- `name`: `String` - 作成する家系図の名前

**戻り値:**
- `Future<String>` - 新しく作成された家系図のID

**例外:**
- 家系図名が空の場合

### updateFamilyTree

```dart
Future<void> updateFamilyTree(FamilyTreeData familyTree)
```

既存の家系図情報（メタデータ、人物、関係性）を更新します。

**パラメータ:**
- `familyTree`: `FamilyTreeData` - 更新する家系図データ

**例外:**
- 家系図名が空の場合

### deleteFamilyTree

```dart
Future<void> deleteFamilyTree(String treeId)
```

指定されたIDの家系図と、それに関連する全ての人物・関係性データを削除します。

**パラメータ:**
- `treeId`: `String` - 削除する家系図のID

### exportFamilyTree

```dart
Future<ExportData> exportFamilyTree(String treeId)
```

指定されたIDの家系図データをエクスポート用の形式で取得します。

**パラメータ:**
- `treeId`: `String` - エクスポートする家系図のID

**戻り値:**
- `Future<ExportData>` - エクスポートデータ

**例外:**
- 指定されたIDの家系図が存在しない場合

### importFamilyTree

```dart
Future<String> importFamilyTree(ExportData exportData, ImportMode mode)
```

エクスポートされた家系図データをインポートします。

**パラメータ:**
- `exportData`: `ExportData` - インポートするデータ
- `mode`: `ImportMode` - インポートモード（新規作成/マージ/上書き）

**戻り値:**
- `Future<String>` - インポートされた家系図のID（新規作成または上書きの場合）または既存のID（マージの場合）

## 使用例

```dart
// リポジトリの取得
final repository = ref.read(familyTreeStorageRepositoryProvider);

// 家系図一覧の監視
final treesStream = repository.getFamilyTrees();
treesStream.listen((trees) {
  print('家系図一覧が更新されました: ${trees.length}件');
});

// 新しい家系図の作成
final newTreeId = await repository.createFamilyTree('新しい家系図');

// 家系図の取得と更新
final treeStream = repository.getFamilyTree(newTreeId);
treeStream.first.then((tree) async {
  if (tree != null) {
    final updatedTree = tree.copyWith(
      name: '更新された家系図名',
      persons: [...tree.persons, Person(id: '', name: '追加人物')],
      lastModified: DateTime.now(),
    );
    await repository.updateFamilyTree(updatedTree);
  }
});

// 家系図のエクスポート
final exportData = await repository.exportFamilyTree(newTreeId);
final jsonString = jsonEncode(exportData.toJson());
// jsonStringをファイルに保存など

// 家系図のインポート（新規作成モード）
// final importedData = ExportData.fromJson(jsonDecode(jsonStringFromFile));
// final importedTreeId = await repository.importFamilyTree(importedData, ImportMode.create);

// 家系図の削除
// await repository.deleteFamilyTree(newTreeId);
```

## 注意事項

- このリポジトリは、家系図データ全体の永続化と取得、およびエクスポート・インポート機能を提供します。
- `updateFamilyTree`メソッドは、家系図のメタデータだけでなく、含まれる人物と関係性のリストも一括で更新します。差分更新ではなく、渡されたリストで上書き（または追加/削除）する形になります。
- `deleteFamilyTree`メソッドは、家系図に関連するサブコレクション（persons, relationships）も全て削除します。
- インポートモード（`ImportMode`）によって、データの取り扱い方が異なります。
    - `create`: 常に新しい家系図として作成します。
    - `merge`: 既存の家系図IDがあれば、その家系図にデータをマージします。人物や関係性はIDが一致すれば上書き、なければ追加されます。
    - `overwrite`: 既存の家系図IDがあれば、その家系図を一度削除してから新しいデータで再作成します。
- Firestoreのバッチ書き込み制限（通常500操作）を考慮し、大量のデータを一度に更新・削除する場合は分割処理が必要です（`updateFamilyTree`と`deleteFamilyTree`内のバッチ処理を参照）。

## 関連するクラス・インターフェース

- [FamilyTreeData](../モデル/FamilyTreeData.md) - 家系図データ全体を表すモデルクラス
- [ExportData](../モデル/ExportData.md) - エクスポート・インポート用のデータ構造
- [PersonRepository](./PersonRepository.md) - 個別の人物データを管理するリポジトリ
- [RelationshipRepository](./RelationshipRepository.md) - 個別の関係性データを管理するリポジトリ
- [FirestoreService](./FirestoreService.md) - Firestoreアクセスを抽象化するサービス
