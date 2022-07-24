Mockito によるテスト
===

JUnit5 の場合。

## 基本
- テストケースのクラスに `@ExtendWith(MockitoExtension.class)` を付与する。
- 以下のフィールドを用意する
	- `@InjectMocks テスト対象クラス target;`
	- テスト対象のクラスのDI対象のフィールドに対応する Mock定義
		- DI対象のフィールドの型が `コンポーネント型名`、`Optional<コンポーネント型名>`, `List<コンポーネント型名>`, `Map<String, コンポーネント型名>` のいずれかの場合の例
			- `@Mock コンポーネント型 フィールド名;`
			- `@Mock(lenient = true) コンポーネント型 フィールド名;`
				- テストケース共通のモック動作を定義する場合は、`lenient = true` の設定が必要。


```java
@ExtendWith(MockitoExtension.class)
class Hoge1ImplTest {
	@InjectMocks
	Hoge1Impl target;

	@Mock
	// @Mock(lenient = true)
	FooInterface foo;

	@Test
	void test() {
		when(foo.foo()).thenReturn("hello");
		assertThat(target.bar()).isEqualTo("hello");
	}
}
```
