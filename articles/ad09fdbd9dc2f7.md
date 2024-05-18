---
title: "Dartのマイクロプログラミングを触ってみた"
emoji: "😎"
type: "tech"
topics: ["Flutter", "Dart", "マイクロプログラミング", "JsonCodable"]
publication_name: "minma"
published: true
---
## はじめに

2024年5月、Flutter 3.22がリリースされました。その中でも注目すべき新機能が、Dartのマイクロプログラミングです。
特に、JSONシリアライズとデシリアライズを大幅に簡素化する[JsonCodable](https://pub.dev/packages/json/versions/0.20.0)という機能が追加されました。
本記事では、`JsonCodable`の使い方、利点などを解説します。

## 注意点

この記事の執筆時点（2024年5月）で、`Dart Macros`はまだ開発中であり、実験的に使用されることになります。そのため、本番環境での使用は慎重にご検討ください。
また、これらの機能は将来的に変更される可能性があるため、最新のドキュメントやリリースノートを常に確認しましょう。

## 環境構築

`Dart Macros`を使うには`Dart 3.5.0-152以降`の環境が必須です。この記事では、以下の開発環境を使って説明していきます。

- Flutter version 3.22.0-36.0.pre.48 on channel master
- Engine revision 552a965b70
- Dart version 3.5.0 (build 3.5.0-167.0.dev)
- DevTools version 2.36.0-dev.10

## Dart Macrosとは？

`Dart Macros`は、静的メタプログラミングまたはコード生成のためのソリューションです。`build_runner`のようなランタイムコード生成とは異なり、マクロはDart言語に完全に統合されており、Dartツールによってバックグラウンドで自動的に実行されます。これにより、外部ツールに依存するよりもはるかに効率的です。

- **追加の操作が不要**: コードを書くときにリアルタイムでマクロがビルドされる。
- **重複作業や再コンパイルの負担がない**: すべてのビルドとコード生成がコンパイラ内で自動的に行われる。
- **ディスクに書き込まれない**: 生成された参照へのポインタやパートファイルが不要で、マクロは既存のクラスを直接拡張する。
- **混乱や難解なテストがない**: カスタム診断メッセージが他のアナライザーメッセージと同様にIDEに直接表示される。

また、従来のように手動で解決策を作成する方法と比べて、`Dart Macros`はシンプルでエラーも発生しにくくなります。

## JsonCodableとは？

`JsonCodable`は、Dartにおける面倒なJSONシリアライズとデシリアライズの一般的な問題に対するシームレスなソリューションを提供するマクロです。

### JsonCodableの使い方

#### 1. 導入

まずはパッケージを`pubspec.yaml`に追加しましょう。

```yaml
dependencies:
  json: ^0.20.1
```

**実験的な機能を利用するために、以下の設定を行う必要があります。**

- `analysis_options.yaml`にて`Dart Macros`を有効にする

```yaml
analyzer:
 enable-experiment:
   - macros
```

- 実験用のフラグをつけてプロジェクトを実行する

```bash
dart run --enable-experiment=macros bin/my_app.dart
```

#### 2. アノテーションの追加

シリアライズ・デシリアライズしたいモデルに`@JsonCodable()`というアノテーションをつけましょう。

```dart
import 'package:json/json.dart';

@JsonCodable() // Macro annotation
class User {
  final String name;
  final String? nickname;
  final int age;
}
```

#### 3. コード生成

これですべての作業が完了です。
変更を保存するだけで即時に以下のコードが作成されます。

*VSCode上、アノテーションの下にある`Go to Augmentation`というリンクをクリックすると表示されます。*

```dart
augment library 'package:macro_test/user.dart';

import 'dart:core' as prefix0;

augment class User {
  external User.fromJson(prefix0.Map<prefix0.String, prefix0.Object?> json);
  external prefix0.Map<prefix0.String, prefix0.Object?> toJson();
  augment User.fromJson(prefix0.Map<prefix0.String, prefix0.Object?> json, )
      : this.name = json['name'] as prefix0.String,
        this.nickname = json['nickname'] as prefix0.String?,
        this.age = json['age'] as prefix0.int;
  augment prefix0.Map<prefix0.String, prefix0.Object?> toJson() {
    final json = <prefix0.String, prefix0.Object?>{};
    json['name'] = this.name;
    if (this.nickname != null) {
      json['nickname'] = this.nickname!;
    }
    json['age'] = this.age;
    return json;
  }
}
```

### JsonCodableの利点

`JsonCodable`を使用することで、以下のような利点があります。

#### 1. コードの簡潔さ

`JsonCodable`は、アノテーションを使用してシリアライズとデシリアライズのロジックをクラスに直接関連付けることができます。
これにより、冗長なコードを削減し、クラス定義が簡潔になります。特に従来の`json_serializable`を利用した方法と比べて、安全かつ高速に開発することが可能です。

```dart
import 'package:json_annotation/json_annotation.dart';

part 'user.g.dart';

@JsonSerializable()
class User {
  final String name;
  final String? nickname;
  final int age;

  User({required this.name, this.nickname, required this.age});

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}
```

#### 2. メンテナンス性の向上

自動生成されるコードにより、手動でシリアライズとデシリアライズのロジックを記述する必要がなくなるため、バグの発生リスクが減少し、コードのメンテナンスが容易になります。クラスの構造が変更された場合でも、`JsonCodable`は対応するコードを自動的に更新します。

#### 3. パフォーマンスの向上

Dart VMの最適化により、`JsonCodable`は非常に高速に動作します。これは特に、大量のデータを扱うアプリケーションでその違いが顕著に現れます。

#### 4. 一貫性

`JsonCodable`は、Dart言語の標準機能として提供されるため、一貫性があり信頼性も高いです。

## まとめ

Flutter 3.22と共に登場した`Dart Macros`と、その一例としての`JsonCodable`を利用することで、開発効率が大幅に向上することは間違いありません。

公式が発表した[タイムライン](https://dart.dev/language/macros#timeline)によると、`JsonCodable`の安定バージョンは今年後半に発表され、`Dart Macros`のカスタマイズ機能は来年登場する見通しです。

Flutterエンジニアの筆者にとって、これらの新機能は日々の開発を大幅に改善するものであり、`build_runner`の煩雑さから解放されることが期待されます。
