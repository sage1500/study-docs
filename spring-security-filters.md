# Spring Security Filters

## フィルタとConfiguration（Servlet版）

デフォルトで有効なフィルタは明示的に関連Configuration に対して `disable()` を呼び出さないと有効になる。

デフォルトで有効となっているのは、以下の実装による。
- Bean定義版
  - HttpSecurityConfiguration のデフォルト定義で設定されているため。  
参照：HttpSecurityConfiguration.httpSecurity() 
- WebSecurityConfigurerAdapter のサブクラス版
  - WebSecurityConfigurerAdapter で、そのように HttpSecurityConfiguration を設定しているから。なお、その設定では、コンストラクタで `super(true)` すると、デフォルトのフィルタを設定しないようになっている。   
参照: WebSecurityConfigurerAdapter.applyDefaultConfiguration(HttpSecurity)


|フィルタ|説明|関連Configuration|デフォルト
|--|--|--|--|
|AuthorizationFilter| | `http.authorizeHttpRequests()` |-
|FilterSecurityInterceptor| | `http.authorizeRequests()` |-
|ExceptionTranslationFilter | 認証系の例外時に AuthenticationEntryPoint を呼び、認証シーケンスを開始したり、AccessDeniedHandler を呼び出したりする。| `http.exceptionHandling()` |有効
|SessionManagementFilter | SessionAuthenticationStrategy（CSRFトークン変更など）、InvalidSessionStrategy を呼び出す。| `http.sessionManagement()`|有効
|AnonymousAuthenticationFilter | この時点で SecurityContextHolder の Authentication が未設定なら AnonymousAuthenticationToken を仕込む。|`http.anonymous()`|有効
|SecurityContextHolderAwareRequestFilter | ServletRequestの認証系メソッドに仕込みを入れる。| `http.servletApi()`|有効
|RequestCacheAwareFilter | RequestCache のための仕込み。| `http.requestCache()`|有効
|BasicAuthenticationFilter | Basic認証 | `http.httpBasic()` |-
|DefaultLogoutPageGeneratingFilter |デフォルトのログアウトページ。ログイン画面が必要な認証方式かつログインページを設定していなく、さらにログアウトが有効の場合に有効化される。|`http.formLogin()`、`http.oauth2Login()`|準有効
|DefaultLoginPageGeneratingFilter |デフォルトのログインページ。ログイン画面が必要な認証方式かつログインページを設定していない場合に有効化される。|`http.formLogin()`、`http.oauth2Login()`|準有効
|UsernamePasswordAuthenticationFilter |ユーザ名とパスワードによるログイン処理する。デフォルトでは、`/login` を処理するフィルタ。 |`http.formLogin()`|-
|??OAuth2LoginAuthenticationFilter??|??|`http.oauth2Login()`|-
|??OAuth2AuthorizationRequestRedirectFilter??|??|`http.oauth2Login()`|-
|LogoutFilter   |ログアウト処理する。デフォルトでは、`/logout` を処理するフィルタ。| `http.logout()`|有効
|CsrfFilter |CSRFトークンチェック。| `http.csrf()`|有効
|HeaderWriterFilter |HeaderWriterによる（X-Frame-Options等の）HTTPヘッダ追記の仕込み。| `http.headers()`|有効
|SecurityContextPersistenceFilter   |SecurityContextHolder の仕込み| `http.securityContext()`|有効
|WebAsyncManagerIntegrationFilter   |WebAsyncManager に対する仕込み | なし


## memo: SessionCreationPolicy の効果
- SecurityContextRepository
    - STATELESS: NullSecurityContextRepository
    - ALWAYS or IF_REQUIRED: HttpSessionSecurityContextRepository。allowSessionCreation は true
    - それ以外: HttpSessionSecurityContextRepository。allowSessionCreation は false
- SecurityContextPersistenceFilter.forceEagerSessionCreation
    - SessionManagementConfigurer がない : false
    - ALWAYS : true
    - ALWAYS以外 : false
- RequestCache
    - STATELESS : NullRequestCache
    - SessionManagementConfigurer がない : Bean定義 → HttpSessionRequestCache
    - それ以外 : Bean定義 → HttpSessionRequestCache 

## memo: http.authorizeRequests() と authorizeHttpRequests() の違い

```java
	public ExpressionUrlAuthorizationConfigurer<HttpSecurity>.ExpressionInterceptUrlRegistry authorizeRequests()
			throws Exception {
		ApplicationContext context = getContext();
		return getOrApply(new ExpressionUrlAuthorizationConfigurer<>(context)).getRegistry();
	}
```

```java
	public AuthorizeHttpRequestsConfigurer<HttpSecurity>.AuthorizationManagerRequestMatcherRegistry authorizeHttpRequests()
			throws Exception {
		ApplicationContext context = getContext();
		return getOrApply(new AuthorizeHttpRequestsConfigurer<>(context)).getRegistry();
	}
```

## memo: URLを処理するフィルタ

|メソッド|パス|処理クラス|処理内容|
|-|-|-|-|
|GET |/logout      |DefaultLogoutPageGeneratingFilter|ログアウト画面
|POST|/logout      |LogoutFilter|ログアウト処理
|GET |/login       |DefaultLoginPageGeneratingFilter|ログイン画面（メッセージなし）
|GET |/login?error |DefaultLoginPageGeneratingFilter|ログイン画面（メッセージあり：エラーメッセージ）
|GET |/login?logout|DefaultLoginPageGeneratingFilter|ログイン画面（メッセージあり：ログアウト成功）
|POST|/login       |UsernamePasswordAuthenticationFilter|ログインボタン押下処理（FormLogin）
|ANY?|/oauth2/authorization/{registrationId}|OAuth2AuthorizationRequestRedirectFilter|ログイン画面（認可エンドポイント）
|GET |/login/oauth2/code/*|OAuth2LoginAuthenticationFilter|ログインボタン押下処理（OAuth2Login）
