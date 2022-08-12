---
title: "アノテーションプロセッシング"
weight: 25
description: >
---

## 概要 {#overview}

コンパイル時、Komapperは [マッピング定義]({{< relref "entity-class/#mapping-definition" >}}) 内のアノテーションを処理し、
結果をメタモデルのソースコードとして生成します。
アノテーションの処理とコードの生成には [Kotlin Symbol Processing API](https://github.com/google/ksp) (KSP)を利用します。

KSPを実行するためには、Gradleビルドスクリプトを次のように記述します。

```kotlin
plugins {
  id("com.google.devtools.ksp") version "1.7.10-1.0.6"
  kotlin("jvm") version "1.7.10"
}

dependencies {
  val komapperVersion = "1.2.0"
  ksp("org.komapper:komapper-processor:$komapperVersion")
}
```

`komapper-processor`モジュールにはKSPのアノテーションプロセッサが含まれます。

上記設定後、Gradleのbuildタスクを実行すると`build/generated/ksp/main/kotlin`ディレクトリ以下にコードが生成されます。

## オプション {#options}

オプションによりアノテーションプロセッサの挙動を変更できます。
利用可能なオプションは以下の5つです。

- komapper.prefix
- komapper.suffix
- komapper.enumStrategy
- komapper.namingStrategy
- komapper.metaObject

オプションを指定するにはGradleのビルドスクリプトで次のように記述します。

```kotlin
ksp {
  arg("komapper.prefix", "")
  arg("komapper.suffix", "Metamodel")
  arg("komapper.enumStrategy", "ordinal")
  arg("komapper.namingStrategy", "UPPER_SNAKE_CASE")
  arg("komapper.metaObject", "example.Metamodels")
}
```

### komapper.prefix

生成されるメタモデルクラスのプレフィックスです。
デフォルト値は`_`（アンダースコア）です。

### komapper.suffix

生成されるメタモデルクラスのサフィックスです。
デフォルト値は空文字です。

### komapper.enumStrategy

Enum型のプロパティをデータベースのカラムどうマッピングするかの戦略です。
値には`name`または`ordinal`のいずれかを選択できます。
デフォルト値は`name`です。
なお、`@KomapperEnum`による指定はこの戦略よりも優先されます。

`komapper.enumStrategy`オプションに指定可能な値の定義は次の通りです。

name
: Enumクラスの`name`プロパティを文字列型のカラムにマッピングする。

ordinal
: Enumクラスの`ordinal`プロパティを整数型のカラムにマッピングする。

### komapper.namingStrategy

Kotlinのエンティクラスとプロパティからデータベースのテーブルとカラムの名前をどう解決するのかの戦略です。
値には`implicit`、`lower_snake_case`、`UPPER_SNAKE_CASE`のいずれかを選択できます。
デフォルト値は`lower_snake_case`です。
解決されたデータベースのテーブルとカラムの名前は生成されるメタモデルのコードの中に含まれます。
なお、`@KomapperTable`や`@KomapperColumn`で名前が指定される場合この戦略で決定される名前よりも優先されます。

`komapper.namingStrategy`オプションに指定可能な値の定義は次の通りです。

implicit
: エンティティクラスやプロパティの名前をそのままテーブルやカラムの名前とする。

lower_snake_case
: エンティティクラスやプロパティの名前をキャメルケースからスネークケースに変換した上で全て小文字にしテーブルやカラムの名前とする。

UPPER_SNAKE_CASE
: エンティティクラスやプロパティの名前をキャメルケースからスネークケースに変換した上で全て大文字にしテーブルやカラムの名前とする。

### komapper.metaObject

メタモデルのインスタンスを拡張プロパティとして提供するobjectを指定します。
デフォルト値は`org.komapper.core.dsl.Meta`です。