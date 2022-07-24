Spring Boot Test
===

参考） https://spring.pleiades.io/spring-boot/docs/current/reference/html/features.html#features.testing


テストケースに `@SpringBootTest` アノテーションをつける。

※JUnit5の場合はこれでOK。JUnit4の場合は、さらに `@RunWith(SpringRunner.class)` が必要。


## 1. 自分のクラス以外のBeanはMockでよい場合
### 1.1 基本
- テストケースのクラスに `@SpringBootTest(classes = テスト対象クラス.class)` を付与する。
- 以下のフィールドを用意する
	- `@Autowired テスト対象クラス target;`
		- MockMvc を使う場合など、テストケース中から直接的にテスト対象クラスを操作しない場合は必要なし。
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
実際の動作としては問題ないはずだが、引っ掛かってしまうのでビュー名とパスは一致しないように設計する必要あり。


### 1.3 MockMVC を使う場合（Spring Securityが依存関係に含まれる場合）
- 通常の MockMVC の使用に加え、以下の対応を追加。
- CSRFを無効にした `SecurityFilterChain` をBean定義する。
- Bean定義したクラスを `@SpringBootTest(classes = {})` に追記する。

Bean定義：
```java
@Configuration
public class TestConfig {
	@Bean
	public SecurityFilterChain testSecurityFilterChain(HttpSecurity http) throws Exception {
		http.csrf().disable();
		return http.build();
	}
}
```

テストケース：
```java
@SpringBootTest(classes = { FooRestController.class, TestConfig.class })
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
