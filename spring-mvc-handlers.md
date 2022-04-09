Spring MVC Handlers
---

memo: ServletConainer から Controller まで
- ServletContainer
	- `<error-page>` -> ErrorController -> error.html
- Filter
	- Filter (Before Spring)
		- SessionRepositoryFilter
	- DelegatingFilterProxy
		- FilterChainProxy (Spring Security) ※2重実行ガード機能あり
			- SecurityFilterChain  
				※複数存在し、URLのパス等により、Filterのセットを切り替えられる  
				※フルセットは https://spring.pleiades.io/spring-security/reference/servlet/architecture.html#servlet-security-filters 参照
				- SecurityContextPersistenceFilter : SecurityContextHolder を設定する。
				- CorsFilter
				- CsrfFilter ※ハンドラ呼出しあり
				- OAuth2AuthorizationRequestRedirectFilter
				- OAuth2LoginAuthenticationFilter
				- UsernamePasswordAuthenticationFilter
				- BearerTokenAuthenticationFilter
				- BasicAuthenticationFilter
				- SessionManagementFilter
				- ExceptionTranslationFilter
					- AuthenticationEntryPoint : HttpSecurity.exceptionHandling() で設定
					- AccessDeniedHandler : HttpSecurity.exceptionHandling() で設定
				- FilterSecurityInterceptor
	- Filter (After Spring)
- Servlet (DispatchServlet)
- Interceptor
- `@ContollerAdvice`
	- `@ExceptionHandler`
- `@Controller`
	- `@ExceptionHandler`
- `@Service`
- `@Repository`

memo: Filterから呼ばれるハンドラ
- CsrfFilter
	- トークン不一致時
		- AccessDeniedHandler
			- 設定方法
				- HttpSecurity.sessionManagement().invalidSessionStrategy()
				- HttpSecurity.sessionManagement().invalidSessionUrl()
					- 実装：SimpleRedirectInvalidSessionStrategy
						- リダイレクトする
				- HttpSecurity.exceptionHandling().accessDeniedHandler()
				- HttpSecurity.exceptionHandling().accessDeniedPage()
					- AccessDeniedHandlerImpl でラップ
						- 403
			- デフォルト実装: DelegatingAccessDeniedHandler
				- MissingCsrfTokenException: InvalidSessionAccessDeniedHandler を呼び出す
				- それ以外: AccessDeniedHandler を呼び出す
		- トークンなし
			- AccessDeniedHandler#handle(,,MissingCsrfTokenException)
		- トークンあり
			- AccessDeniedHandler#handle(,,InvalidCsrfTokenException)

memo: ハンドラデフォルト
- InvalidSessionStrategy
	- 概要
		- セッションが無効の場合に呼ばれる。無効になったタイミングで呼ばれるのではなく、セッションがない初回アクセスの場合にも呼ばれる。セッションが無効になったことをエラー検知する目的では使えない。基本的にデフォルトのままでよい。
	- 設定
		- HttpSecurity.sessionManagement().invalidSessionStrategy()
		- HttpSecurity.sessionManagement().invalidSessionUrl()
	- デフォルト値
		- null
		- SimpleRedirectInvalidSessionStrategy: invalidSessionUrl() を設定していた場合
			- リダイレクトする
- AuthenticationFailureHandler
	- HttpSecurity.sessionManagement().sessionAuthenticationFailureHandler()
		- SimpleUrlAuthenticationFailureHandler
			- 401
- SessionAuthenticationStrategy
	- HttpSecurity.sessionManagement().sessionAuthenticationStrategy()
		- ChangeSessionIdAuthenticationStrategy
- AuthenticationEntryPoint
	- 概要
		- 認証を開始する処理。`http.oauth2Login()` 等で認証方法を設定した場合は上書きすべきではない。
	- HttpSecurity.exceptionHandling().authenticationEntryPoint() および defaultAuthenticationEntryPointFor()
		- Http403ForbiddenEntryPoint
			- 403
	- 呼び出し契機:
		- AuthenticationException: ExceptionTranslationFilter
		- SessionAuthenticationException: SessionManagementFilter
		- InternalAuthenticationServiceException: AbstractAuthenticationProcessingFilter
		- AuthenticationException: AuthenticationFilter
		- AuthenticationException: AbstractPreAuthenticatedProcessingFilter
		- BadCredentialsException: DigestAuthenticationFilter
		- AuthenticationException: BasicAuthenticationFilter

- AccessDeniedHandler
	- HttpSecurity.exceptionHandling().accessDeniedHandler() および defaultAccessDeniedHandlerFor()
		- AccessDeniedHandlerImpl
			- 403

memo: BasicErrorController
- HTTPステータス:
	- request の RequestDispatcher.ERROR_STATUS_CODE 属性から取得
- Model に載せるパラメータ
	- 標準パラメータをのせるかどうか
		- server.error.xxx プロパティで調整
	- カスタマイズ
		- ErrorAttributes Beanを定義
- ビュー名
	- ErrorViewResolver Beanを定義
		- 評価順は `@Order`, `@Priority` アノテーション、または `Ordered` インタフェース実装で制御する
		- 参考) DefaultErrorViewResolver
			- error/4xx に振り分けている Bean


memo: Spring Boot における Tomcat への Filter登録
- AbstractFilterRegistrationBean のサブクラス
	- org.springframework.boot.web.servlet.DelegatingFilterProxyRegistrationBean
		- ユーザが任意の Bean名に対する DelegatingFilterProxy を生成
	- org.springframework.boot.web.servlet.FilterRegistrationBean
		- ユーザが任意のFilterを生成
- WebApplicationInitializer の実装クラス
	- org.springframework.security.web.context.AbstractSecurityWebApplicationInitializer
		- DelegatingFilterProxy を生成("springSecurityFilterChain"に対する Proxy)
	- org.springframework.session.web.context.AbstractHttpSessionApplicationInitializer.onStartup(ServletContext)
		- DelegatingFilterProxy を生成("springSessionRepositoryFilter"に対する Proxy)
			- order : Integer.MIN_VALUE + 50
	- org.springframework.web.servlet.support.AbstractDispatcherServletInitializer.onStartup(ServletContext)
		- ??

# Filter

`@Component`を付けた Filter を実装すると、Spring Security の内部のフィルターとして取り込まれる（？）。

※TODO `@Order` で Spring Security より前に設定しても Spring Security の Filter の後に組み込まれていたはず。

Sessionや認証情報のチェック後に配置されるため、おおよそ期待した前準備ができている状態で
実装したフィルタが呼び出されると思ってよい。

なお、1回の Request で 1度しか Filter 処理されないようにしたい場合は `OncePerRequestFilter` を拡張するとよい。

```java
@Component
@Order(1) // 書かなくてもよい。 TODO Order の書きっぷり
public class DemoFilter extends OncePerRequestFilter {

	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		
		new Exception().fillInStackTrace().printStackTrace();
		
		filterChain.doFilter(request, response);
	}

}
```

Stack Traceの例）  
`@Component`を付けた Filter でのスタックトレースのうち、フィルター関連のものを抽出したものは以下のとおり。

※Spring Security の設定内容等により増減する。

```
com.example.demo.web.DemoFilter.doFilterInternal
org.springframework.security.web.access.intercept.FilterSecurityInterceptor
org.springframework.security.web.access.ExceptionTranslationFilter
org.springframework.security.web.session.SessionManagementFilter
org.springframework.security.web.authentication.AnonymousAuthenticationFilter
org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter
org.springframework.security.web.savedrequest.RequestCacheAwareFilter
org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter
org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter
org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter
org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter
org.springframework.security.web.authentication.logout.LogoutFilter
org.springframework.security.web.csrf.CsrfFilter
org.springframework.security.web.header.HeaderWriterFilter
org.springframework.security.web.context.SecurityContextPersistenceFilter
org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter
org.springframework.security.web.FilterChainProxy
org.springframework.web.filter.DelegatingFilterProxy
org.springframework.web.filter.RequestContextFilter
org.springframework.web.filter.FormContentFilter
org.springframework.session.web.http.SessionRepositoryFilter
org.springframework.web.filter.CharacterEncodingFilter
```


## 対象パスを絞り込みたい場合
Spring Security と一緒に使用する場合は、
Spring Security の対象となるパスに絞り込まれている。
多くの場合、これで十分なはず。

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	// ...

	@Override
	public void configure(WebSecurity web) throws Exception {
		web.ignoring().antMatchers("/css/**", "/js/**", "/img/**", "/management/health");
	}

	// ...
}
```

Spring Security の取り込まれずにパスの絞り込みを制御したい場合は、
`Filter` 自体は Bean定義せず、対象の `Filter` をラップした
`FilterRegistrationBean` を Bean定義する。

```java
@Bean
public FilterRegistrationBean someFilter() {
	FilterRegistrationBean bean = new FilterRegistrationBean(new someFilter());
	bean.addUrlPatterns("/*");
	bean.setOrder(Integer.MIN_VALUE);
	return bean;
}
```


# Interceptor

Interceptor は Bean定義しただけでは自動登録されない。
自前で InterceptorRegistry に登録する必要がある。

Interceptorの実装：

```java
@Component
public class DemoInterceptor implements HandlerInterceptor {

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		new Exception().fillInStackTrace().printStackTrace();
		return true;
	}

}
```

Interceptorの登録：
```java
@Configuration
@RequiredArgsConstructor
public class MvcConfig implements WebMvcConfigurer {
	private final DemoInterceptor demoInterceptor;
	
	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(demoInterceptor);
	}

}
```

順序を制御したり、パスを絞り込みたい場合は、`addInterceptor()` の返値である `InterceptorRegistration` に対して各種メソッドを呼び出す。

TODO サンプル
```java
// TODO サンプルコード
```

# ControllerAdvice / RestControllerAdvice

クラスのアノテーションに `@ControllerAdvice`, `@RestControllerAdvice` をつけるだけでOK。

`@Controller`, `@RestController` をつけたクラスと同様に
`@ExceptionHandler`, `@InitBinder`, `@ModelAttribute` をつけたメソッドを全Controller共通で呼ばれる。


# ログイン後の初回アクセスのハンドリング

標準ではいい感じのハンドラは提供されていないっぽい。

※`AuthenticationSuccessHandler` というインタフェースがあるが、これは Spring Security の内部で認証シーケンスを処理するために使用するインタフェースのため使えない。

アイデアとしては以下のものがある。
1. `Interceptor` の `preHandle()` で以下を判定し、独自のハンドラを呼び出す Interceptor を実装する。
	- 認証済である(後述)
	- ログイン後の初回アクセス時である(後述)
1. 「認証済である」の判定方法
	1. SecurityContextHolder.getContext().getAuthentication() が AnonymousAuthenticationToken ではない(もしくは、期待した xxAuthenticationTokenである)
	1. かつ、isAuthenticated() が true
1. 「ログイン後の初回アクセス時である」の判定方法
	1. 独自のセッション属性が未設定である。  
		※独自のハンドラを呼び出した後は独自のセッション属性に値を設定する。

