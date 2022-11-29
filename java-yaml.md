# Java で Yaml を扱う

## 1. ライブラリ

Java で Yaml を扱うライブラリ。

1. Jackson  
   内部実装に SnakeYaml を利用している。
   `jackson-dataformat-yaml` を利用することで、Jackson で Yaml を利用できるようになる。
   ただし、json で実現可能な範囲の yaml しか読み込めない。複数ドキュメントや、アンカー、エイリアスはサポートしていない。
   json で実現可能な範囲の yamlしか扱わないのであれば、ちょっとの修正で json と yaml の両方を対応することが可能であるため、便利。
1. SnakeYaml  
   Spring Boot が内部で使用している。

## 2. Jackson

### 2.1 依存関係

```xml
<dependency>
	<groupId>com.fasterxml.jackson.dataformat</groupId>
	<artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```

Spring Boot の依存関係を取り込んでいる場合は、`<version>` の指定は不要。

### 2.2. 基本

Jackson で json を利用する場合と同じように使える。
異なるのは、`new ObjectMapper()`
の代わりに `new ObjectMapper(new YAMLFactory())`
を使用するだけでよい。

なお、JSON のファイルフォーマットとして Yaml を扱えるようにしたものであるため、処理できる範囲は JSON のそれに限られる。具体的には、複数ドキュメント、アンカーとエイリアスは期待した動きにならない。

### 2.3. 実装例

```java
@Data
public static class Hello6Dto1 {
    private String message;
    private int intProp1;
    private boolean boolProp2;
}
```

```java
var m = new ObjectMapper(new YAMLFactory());
var v = m.readValue(//
        "" + //
        "message: hoge\n" + //
        "intProp1: 123\n" + //
        "boolProp2: true\n" + //
        "", //
        Hello6Dto1.class);
```

- 期待した結果になる
- 上記の書き方の場合、
  - yaml 側の key名が Java側のフィールド名に完全に一致しないと例外がスローされる
    - `boolProp2` ではなく、 `bool-prop2` だと例外がスローされる。

### 2.4. readValue() が返す型

```java
var m = new ObjectMapper(new YAMLFactory());
var v = m.readValue("...YAML文字列...", Object.class);
```

1. 最初の項目が `-` の場合: ArrayList
2. 最初の項目が `xxx:` の場合: LinkedHashMap
3. `---` を使った場合: 最初のドキュメント分のみ取得対象になる
4. アンカーとエイリアス: 期待した動きにならない


## 3. SnakeYaml

### 3.1 依存関係

```xml
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
</dependency>
```

Spring Boot Starter が SnakeYaml への依存関係を持っているため、
Spring Boot を利用する場合は、依存関係を追加する必要なく利用できる。

### 3.2. 例

```java
var y = new Yaml();
var v1 = y.load("...YAML文字列...");
var v2 = y.loadAll("...YAML文字列-複数ドキュメントOK...");
var v3 = y.loadAs("...YAML文字列...", <マッピング先クラス>.class);

// ※注
// load(), loadAll(), loadAs() の第一引数は
// String, InputStream, Reader のバージョンがある。
```

- load() : 単一ドキュメントのロード（※1）。型指定なし。
- loadAll() : 複数ドキュメントのロード（単一ドキュメントでも可）。型指定なし。
- loadAs() : 単一ドキュメントのロード（※1）。型指定あり。

※１ 複数ドキュメントのデータを与えると、例外がスローされる。

### 3.3. load() が返す型
1. 最初の項目が `-` の場合: ArrayList
2. 最初の項目が `xxx:` の場合: LinkedHashMap

### 3.4. loadAs()
- yaml 側の key名が Java側のフィールド名に完全に一致しないと例外がスローされる
  - `boolProp2` ではなく、 `bool-prop2` だと例外がスローされる。
