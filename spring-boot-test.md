Spring Boot Test
===

参考） https://spring.pleiades.io/spring-boot/docs/current/reference/html/features.html#features.testing


テストケースに `@SpringBootTest` アノテーションをつける。

※JUnit5の場合はこれでOK。JUnit4の場合は、さらに `@RunWith(SpringRunner.class)` が必要。


## 1. 自分のクラス以外のBeanはMockでよい場合
### 1.1 基本
- テストケースのクラスに `@SpringBootTest(classes = テスト対象クラス.class)` または `@SpringBootTest(classes = { テスト対象クラス.class, 追加のConfigクラス.class })`を付与する。  
	※ 前者の書き方を採用し、`追加のConfigクラス` を `Import`アノテーションを利用して、`@Import(追加のConfigクラス.class)` のように書いても同じ動きする（内部的にも完全に同じかどうかは不明）。
- 以下のフィールドを用意する
	- `@Autowired テスト対象クラス target;`
		- MockMvc を使う場合など、テストケース中から直接的にテスト対象クラスを操作しない場合は用意しなくてもよい。
	- テスト対象のクラスのDI対象のフィールドに対応する Mock定義
		- DI対象のフィールドの型が `コンポーネント型名`、`Optional<コンポーネント型名>`, `List<コンポーネント型名>`, `Map<String, コンポーネント型名>` のいずれかの場合の例
			- `@MockBean コンポーネント型 フィールド名;`  
				※Mockito を直接使用する場合と違い、`lenient = true` の設定は不要。

```java
@SpringBootTest(classes = Hoge1Impl.class)
class Hoge1ImplTest {
	@Autowired
	Hoge1Impl target;

	@MockBean
	FooInterface foo;

	@Test
	void test() {
		when(foo.foo()).thenReturn("hello");
		assertThat(target.bar()).isEqualTo("hello");
	}
}
```

### 1.2 MockMVC を使う場合
- テストケースのクラスに以下のアノテーションを追加する。
	- `@AutoConfigureMockMvc`
	- `@AutoConfigureWebMvc`  
		※JSONデータを POST しない場合はなくても構わない。  
		※`@SpringBootTest`アノテーション の `classes` に追加のConfigクラスを定義している場合は、追加の Configクラス側に本アノテーションを追加してもよい。
- フィールドに以下を追加する。
	- `@Autowired MockMvc mvc;`

```java
@SpringBootTest(classes = FooRestController.class)
@AutoConfigureMockMvc
@AutoConfigureWebMvc
class FooRestControllerTest {
	@Autowired
	MockMvc mvc;

	ObjectMapper mapper = new ObjectMapper();

	@Test
	void test() throws Exception {
		var body = mvc.perform(post("/foo").content("{\"name\": \"hello\"}").contentType(MediaType.APPLICATION_JSON)) //
				.andExpect(status().isOk()) //
				.andReturn().getResponse().getContentAsString();
		var resource = mapper.readValue(body, FooRestController.FooResource.class);
		assertThat(resource.getName()).isEqualTo("foo");
	}
}
```

なお、ビュー名がパス名と一致すると、循環として検知されエラーになる。
実際の動作としては問題ないはずだが、MockMVC のチェックには引っ掛かってしまうため、ビュー名とパスは一致しないように設計する必要あり。


### 1.2.1 Spring Securityが依存関係に含まれる場合

Spring Security が依存関係に含まれると、
自動構成により、Spring Security のデフォルトのフィルタが設定される。
このデフォルトのフィルタが MockMVC を使用したテストを阻害することが多い。

そのため、通常の MockMVC の使用に加え、以下の対応が必要。

- テスト対象のクラスを試験するうえで、Spring Security が必要な場合
	- 以下のいずれかで対応
		- デフォルトのフィルタが設定されないように必要最低限の SecurityConfig を実装する。
			- 対応方法: 通常、CSRFを無効にした `SecurityFilterChain` をBean定義することで十分。この SecurityConfig を`@SpringBootTest(classes = {})` に追記する。
		- 実際の SecurityConfig のクラスを利用することでデフォルトのフィルタが適用されないようにする。
			- 対応方法: 実際の SecurityConfig のクラスを`@SpringBootTest(classes = {})` に追記する。SecurityConfigクラスの実装によっては、特定のプロパティの設定等が必要になることがあるため、必要に応じて設定を追加する。SecurityConfigクラスの実装の影響を受けるため、この方法はあまりお勧めしない。
- テスト対象のクラスを試験するうえで、Spring Security が不要な場合
	- Spring Security の自動構成を無効にすることでデフォルトのフィルタが設定されないようにする。
		- 対応方法: `@SpringBootTest(classes = {})` に記載しているいずれかのクラスに `@ImportAutoConfiguration(exclude = SecurityAutoConfiguration.class)` を付与する。

Bean定義：
```java
// 以下、(1)(2) はいずれか一方を採用する。Spring Security が必要な場合は(2)、不要な場合は(1)を採用する。
// なお、Spring Security を依存関係に含まないプロジェクトの場合は (1)(2)ともに採用しない。
@Configuration
@AutoConfigureWebMvc
@ImportAutoConfiguration(exclude = SecurityAutoConfiguration.class) // ...(1)
public class WebMvcTestConfig {
	// ...(2)
	@Bean
	public SecurityFilterChain testSecurityFilterChain(HttpSecurity http) throws Exception {
		http.csrf().disable();
		return http.build();
	}
}
```

テストケース：
```java
// WebMvcTestConfig 側に記載しているため、@AutoConfigureWebMvc の記載は省略している
@SpringBootTest(classes = { FooRestController.class, WebMvcTestConfig.class })
@AutoConfigureMockMvc
class FooRestControllerTest {
	@Autowired
	MockMvc mvc;

	ObjectMapper mapper = new ObjectMapper();

	@Test
	void test() throws Exception {
		var body = mvc.perform(post("/foo").content("{\"name\": \"hello\"}").contentType(MediaType.APPLICATION_JSON)) //
				.andExpect(status().isOk()) //
				.andReturn().getResponse().getContentAsString();
		var resource = mapper.readValue(body, FooRestController.FooResource.class);
		assertThat(resource.getName()).isEqualTo("foo");
	}
}
```

## 2. 本物のBeanを使用したい場合（SpringBootApplication を配置していないモジュールのテスト）

MyBatisなどの本物のBeanを使いたい場合を想定。

- src/test/java
	- `@SpringBootConfiguration` のついたダミーのクラスを用意する
		```java
		@SpringBootConfiguration
		@EnableAutoConfiguration
		//@ComponentScan
		public class DemoDomainTestConfiguration {
		}
		```
		`@ComponentScan` は基本的に不要と思われるが、テスト対象以外も本物のBeanを使いたい場合は記載してもよいかも（？）。また、`@SpringBootApplication` を使ったときと同様に `@MapperScan` は書かなくても動く。
- src/test/resources
	- application.yml を置く
		- 分割しているプロファイルを spring.profiles.active や spring.profiles.default で指定する。
			SpringBootApplication を配置しているモジュールの application.yml を参考にする。
			```yml
			spring.profiles.active:
			- common
			- h2
			```
			- テストケースのクラスに `@ActiveProfiles` アノテーションを付与することでも対応できるが、
				全テストケースに書くよりは、application.yml に書く方が楽。
			- スーパークラスに `@ActiveProfiles` を書くという方法もあるようだ。この場合、サブクラス側で`@ActiveProfiles` を書くと、プロファイルの設定が追加される（？）
		- その他必要な設定を追記する。
