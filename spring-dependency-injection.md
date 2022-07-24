Spring Dependency Injection
===

## 1. インジェクションの種類
- コンストラクタインジェクション
- フィールドインジェクション
- セッターインジェクション

## 2. 解決方法の種類
- 型による解決
- 名前による解決

## 3. 解決コンポーネント数
- 1件
- 0または1件
- 0件以上

## 4. 実装方法

### 4.1 コンストラクタインジェクション

#### 4.1.1 基本
- `@RequiredArgsConstructor` アノテーションをクラスに付与する。
- DI先のフィールドを `final 型名 任意のフィールド名;` で定義する。  
  ※通常、`private` 修飾も記載する。
- 「型による解決」は型名で対応
- 「名前による解決」はフィールド名で対応
- 解決コンポーネント数が「0または1件」は `Optional<コンポーネント型名>` で対応
- 解決コンポーネント数が「1件以上」は `List<コンポーネント型名>` および `Map<コンポーネント型名>` で対応

#### 4.1.2 制限事項
- 「名前による解決」は「型のよる解決」のみの場合に対象コンポーネントが2件以上ある場合にのみ有効。
- `org.springframework:spring-context` を依存関係に追加する必要あり。（Spring Bootを使用する場合は通常、意識する必要なし）

#### 4.1.3 フィールド定義パターン

|フィールド定義|解決方法の種類|解決コンポーネント数|備考
|-|-|-|-|
|`final コンポーネント型名 任意のフィールド名;`|型による解決|1件
|`final コンポーネント型名 コンポーネント名;`|型による解決、名前による解決|1件
|`final Optional<コンポーネント型名> 任意のフィールド名;`|型による解決|0または1件
|`final Optional<コンポーネント型名> コンポーネント名;`|型による解決、名前による解決|0または1件
|`final List<コンポーネント型名> 任意のフィールド名;`|型による解決|0件以上
|`final Map<String, コンポーネント型名> 任意のフィールド名;`|型による解決|0件以上|コンポーネント名付き

#### 4.1.4. 実装例

```java
@Component
@RequiredArgsConstructor
public class MyComponent {
	private final HogeInterface hoge;
}
```

```java
@Component
@RequiredArgsConstructor
public class MyComponent {
	private final HogeInterface hogeImpl; // 名前による解決
   	private final Optional<FooInterface> foo;
	private final List<HogeInterface> hoges; // 0件以上（コンポーネントのみ）
	private final Map<String, HogeInterface> hogeMap; // 0件以上（コンポーネント名含む）
}
```

### 4.2 フィールドインジェクション
#### 4.2.1 基本
- DI先のフィールドを `@Autowired 型名 任意のフィールド名;` で定義する。  
  ※通常、型名の前に `private` 修飾も記載する。  
  ※`@Autowired` の部分には、以下のアノテーションが使用できる。ただし、それぞれ若干、挙動が違うことに注意。ここでは `@Autowired` についてのみ言及する。
    - `@Autowired`
    - `@Resource`
    - `@Inject`
- 「型による解決」は型名で対応
- 「名前による解決」はフィールド名、または、`@Qualifier` で対応
- 解決コンポーネント数が「0または1件」は `Optional<コンポーネント型名>` または `@Autowired(required = false)` で対応
- 解決コンポーネント数が「1件以上」は `List<コンポーネント型名>` および `Map<コンポーネント型名>` で対応

#### 4.2.2 制限事項

#### 4.2.3 フィールド定義パターン

|フィールド定義|解決方法の種類|解決コンポーネント数|備考
|-|-|-|-|
|`@Autowired コンポーネント型名 任意のフィールド名`|型による解決|1件
|`@Autowired コンポーネント型名 コンポーネント名`|型による解決、名前による解決|1件|対象型のコンポーネント数が1件の場合はコンポーネント名をチェックしない。
|`@Autowired @Qualifier("コンポーネント名") コンポーネント型名 任意のフィールド名`|型による解決、名前による解決|1件|コンポーネント名を厳格にチェックする。
|`@Autowired Optional<コンポーネント型名> 任意のフィールド名;`|型による解決|0または1件
|`@Autowired(required = false) コンポーネント型名 任意のフィールド名;`|型による解決|0または1件
|`@Autowired Optional<コンポーネント型名> コンポーネント名;`|型による解決、名前による解決|0または1件|対象型のコンポーネント数が1件の場合はコンポーネント名をチェックしない。
|`@Autowired(required = false) @Qualifier("コンポーネント名") コンポーネント型名 任意のフィールド名;`|型による解決、名前による解決|0または1件|コンポーネント名を厳格にチェックする。
|`@Autowired List<コンポーネント型名> 任意のフィールド名;`|型による解決|0件以上|Spring Framework のバージョンによっては、`= new ArrayList<>();` などの初期化子が必要。
|`@Autowired Map<String, コンポーネント型名> 任意のフィールド名;`|型による解決|0件以上|コンポーネント名付き。Spring Framework のバージョンによっては、`= new HashMap<>();` などの初期化子が必要。

#### 4.2.4. 実装例

```java
@Component
public class MyComponent {
    @Autowired
	private HogeInterface hoge;
}
```

```java
@Component
public class MyComponent {
    @Autowired
	private HogeInterface hogeImpl; // 名前による解決

    @Autowired
    @Qualifier("hoge2Impl")
	private HogeInterface hoge2Impl; // 名前による解決

    @Autowired
   	private Optional<FooInterface> foo;

    @Autowired(required = false)
   	private Optional<FooInterface> foo2;

    @Autowired
	private List<HogeInterface> hoges; // 0件以上（コンポーネントのみ）

    @Autowired
	private Map<String, HogeInterface> hogeMap; // 0件以上（コンポーネント名含む）
}
```

### 4.3 セッターインジェクション

省略
