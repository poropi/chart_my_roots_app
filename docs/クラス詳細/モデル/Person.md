# Person クラス

## 概要

`Person` クラスは家系図内の人物を表すモデルクラスです。人物の基本情報（氏名、生年月日、性別など）を保持し、Firestoreとの連携に必要なシリアライズ/デシリアライズ機能を提供します。

## クラス定義

```dart
@freezed
class Person with _$Person {
  const factory Person({
    required String id,
    required String name,
    String? maidenName,
    DateTime? birthDate,
    DateTime? deathDate,
    Gender? gender,
    String? memo,
    @Default(0.0) double x,
    @Default(0.0) double y,
  }) = _Person;

  factory Person.fromJson(Map<String, dynamic> json) => _$PersonFromJson(json);
  
  factory Person.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Person.fromJson({
      'id': doc.id,
      ...data,
      'birthDate': data['birthDate'] != null 
        ? (data['birthDate'] as Timestamp).toDate().toIso8601String() 
        : null,
      'deathDate': data['deathDate'] != null 
        ? (data['deathDate'] as Timestamp).toDate().toIso8601String() 
        : null,
    });
  }
  
  Map<String, dynamic> toFirestore() {
    final json = toJson();
    json.remove('id'); // IDはドキュメントIDとして使用するため削除
    
    // DateTimeをTimestampに変換
    if (birthDate != null) {
      json['birthDate'] = Timestamp.fromDate(birthDate!);
    }
    if (deathDate != null) {
      json['deathDate'] = Timestamp.fromDate(deathDate!);
    }
    
    return json;
  }
}
```

## プロパティ

| 名前 | 型 | 説明 | デフォルト値 |
|------|------|------|------------|
| `id` | `String` | 人物の一意識別子 | なし（必須） |
| `name` | `String` | 人物の氏名 | なし（必須） |
| `maidenName` | `String?` | 旧姓（結婚前の姓） | `null` |
| `birthDate` | `DateTime?` | 生年月日 | `null` |
| `deathDate` | `DateTime?` | 没年月日 | `null` |
| `gender` | `Gender?` | 性別（男性/女性/その他） | `null` |
| `memo` | `String?` | 人物に関するメモ情報 | `null` |
| `x` | `double` | 家系図上のX座標 | `0.0` |
| `y` | `double` | 家系図上のY座標 | `0.0` |

## メソッド

### Person.fromJson

```dart
factory Person.fromJson(Map<String, dynamic> json)
```

JSONオブジェクトから`Person`インスタンスを生成します。`freezed`パッケージによって自動生成されます。

**パラメータ:**
- `json`: `Map<String, dynamic>` - 変換元のJSONオブジェクト

**戻り値:**
- `Person` - 生成された`Person`インスタンス

### Person.fromFirestore

```dart
factory Person.fromFirestore(DocumentSnapshot doc)
```

Firestore `DocumentSnapshot`から`Person`インスタンスを生成します。Firestoreの`Timestamp`型を`DateTime`型に変換する処理を含みます。

**パラメータ:**
- `doc`: `DocumentSnapshot` - Firestoreから取得したドキュメント

**戻り値:**
- `Person` - 生成された`Person`インスタンス

### toFirestore

```dart
Map<String, dynamic> toFirestore()
```

`Person`インスタンスをFirestoreに保存可能な形式に変換します。`DateTime`型を`Timestamp`型に変換する処理を含みます。

**戻り値:**
- `Map<String, dynamic>` - Firestoreに保存可能な形式のマップ

## 使用例

```dart
// 新しい人物の作成
final person = Person(
  id: '',  // 新規作成時は空文字列、Firestoreが自動生成
  name: '山田 太郎',
  birthDate: DateTime(1980, 4, 1),
  gender: Gender.male,
);

// Firestoreへの保存用データの取得
final firestoreData = person.toFirestore();

// Firestoreから取得したデータの変換
final retrievedPerson = Person.fromFirestore(docSnapshot);

// 値の変更（freezedのcopyWithを使用）
final updatedPerson = person.copyWith(
  memo: '会社役員。趣味は釣り。',
);
```

## 注意事項

- `id`プロパティは、FirestoreのドキュメントIDに対応します。新規作成時は空文字列で、Firestoreに保存後に自動生成されたIDが設定されます。
- `birthDate`と`deathDate`は`null`許容であり、未設定の場合は表示されません。
- 座標値（`x`, `y`）は、家系図上での表示位置を表し、レイアウトエンジンによって計算されます。
- このクラスは`freezed`パッケージを使用した不変（イミュータブル）クラスです。値を変更する場合は`copyWith`メソッドを使用します。

## 関連するクラス・列挙型

### Gender 列挙型

```dart
enum Gender {
  male,    // 男性
  female,  // 女性
  other,   // その他
}
```
