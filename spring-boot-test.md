Spring Boot Test
---

参考） https://spring.pleiades.io/spring-boot/docs/current/reference/html/features.html#features.testing


テストケースに `@SpringBootTest` アノテーションをつける。

※JUnit5の場合はこれでOK。JUnit4の場合は、さらに `@RunWith(SpringRunner.class)` が必要。


# SpringBootApplication を配置していないモジュールのテスト

一部、MyBatisなどの本物のBeanを使いたい場合は、
いまのところ、以下のような感じでないと動かせない。

- src/test/java
	- `@SpringBootApplication` の代わりに以下のいずれかで対応	
		- `@SpringBootApplication` のついたダミーのクラスを用意する
		- `@SpringBootApplication` の中身のアノテーションのついたダミーのクラスを用意する
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

# 自分のクラス以外のBeanはMockでよい場合

`@SpringBootTest` の classes に 自身のクラスを書き、
DI対象のクラスは、`@MockBean` で定義する。

```java
@SpringBootTest(classes = HelloController.class)
public class HelloControllerTest {
	@MockBean
	OrderRepository orderRepository;
	
	@Test
	public void test_hoge() {
		// ...
	}
}
```

MockMvc を使う場合は、さらに `@AutoConfigureMockMvc` をつける。

```java
@SpringBootTest(classes = HelloController.class)
@AutoConfigureMockMvc(addFilters = false) // これをつけることで Spring Security がフィルタとしてセットアップされることを避ける
public class HelloControllerTest {
	// ...
}
```

なお、ビュー名がパス名と一致すると、循環として検知されエラーになる。
実際の動作としては問題ないはずだが、引っ掛かってしまうのでビュー名とパスは一致しないように設計する必要あり。
