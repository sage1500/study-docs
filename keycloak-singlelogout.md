# KeyCloak Single Logout


- k_logout を処理する部分
    - org.keycloak.broker.oidc.KeycloakOIDCIdentityProvider.KeycloakEndpoint.backchannelLogout(String)

- k_logout の JWTと内容を保持するクラス
    - org.keycloak.representations.adapters.action.LogoutAction
      - AdminAction のサブクラス
  

- ログアウト時の adapterSessionIds の収集
    - org.keycloak.services.managers.ResourceAdminManager.logoutClientSessions(URI, RealmModel, ClientModel, List<AuthenticatedClientSessionModel>)  
        ```java
        adapterSessionIds = new MultivaluedHashMap<String, String>();
        for (AuthenticatedClientSessionModel clientSession : clientSessions) {
            String adapterSessionId = clientSession.getNote(AdapterConstants.CLIENT_SESSION_STATE);
            if (adapterSessionId != null) {
                String host = clientSession.getNote(AdapterConstants.CLIENT_SESSION_HOST);
                adapterSessionIds.add(host, adapterSessionId);
            }
            if (clientSession.getUserSession() != null) userSessions.add(clientSession.getUserSession().getId());
        }

        if (adapterSessionIds == null || adapterSessionIds.isEmpty()) {
            logger.debugv("Can't logout {0}: no logged adapter sessions", resource.getClientId());
            return false;
        }
        ```  
        →CLIENT_SESSION_STATE が載っていないと k_logout が呼ばれない

        ```java
        if (managementUrl.contains(CLIENT_SESSION_HOST_PROPERTY)) {
            boolean allPassed = true;
            // Send logout separately to each host (needed for single-sign-out in cluster for non-distributable apps - KEYCLOAK-748)
            for (Map.Entry<String, List<String>> entry : adapterSessionIds.entrySet()) {
                String host = entry.getKey();
                List<String> sessionIds = entry.getValue();
                String currentHostMgmtUrl = managementUrl.replace(CLIENT_SESSION_HOST_PROPERTY, host);
                allPassed = sendLogoutRequest(realm, resource, sessionIds, userSessions, 0, currentHostMgmtUrl) && allPassed;
            }

            return allPassed;
        } else {
            // Send single logout request
            List<String> allSessionIds = new ArrayList<String>();
            for (List<String> currentIds : adapterSessionIds.values()) {
                allSessionIds.addAll(currentIds);
            }

            return sendLogoutRequest(realm, resource, allSessionIds, userSessions, 0, managementUrl);
        }
        ```  
        →CLIENT_SESSION_HOST_PROPERTY = "${application.session.host}" を AdminUrl に設定しておくと、
        その部分は、CLIENT_SESSION_HOST で置換してくれる。  
        →一つのクライアント設定で複数取り扱うのに便利。URLを全置換するように CLIENT_SESSION_HOST を設定できるのだろうか？

- 認可コードでトークンエンドポイントにアクセス時
  - org.keycloak.adapters.ServerRequest.invokeAccessCodeToToken(KeycloakDeployment, String, String, String, String)  
    ```java
    public static AccessTokenResponse invokeAccessCodeToToken(KeycloakDeployment deployment, String code, String redirectUri, String sessionId, String codeVerifier) throws IOException, HttpFailure {
        List<NameValuePair> formparams = new ArrayList<>();
        redirectUri = stripOauthParametersFromRedirect(redirectUri);
        formparams.add(new BasicNameValuePair(OAuth2Constants.GRANT_TYPE, "authorization_code"));
        formparams.add(new BasicNameValuePair(OAuth2Constants.CODE, code));
        formparams.add(new BasicNameValuePair(OAuth2Constants.REDIRECT_URI, redirectUri));
        if (sessionId != null) {
            formparams.add(new BasicNameValuePair(AdapterConstants.CLIENT_SESSION_STATE, sessionId));
            formparams.add(new BasicNameValuePair(AdapterConstants.CLIENT_SESSION_HOST, HostUtils.getHostName()));
        }
        // https://tools.ietf.org/html/rfc7636#section-4
        if (codeVerifier != null) {
            logger.debugf("add to POST parameters of Token Request, codeVerifier = %s", codeVerifier);
            formparams.add(new BasicNameValuePair(OAuth2Constants.CODE_VERIFIER, codeVerifier));
        } else {
            logger.debug("add to POST parameters of Token Request without codeVerifier");
        }

        HttpPost post = new HttpPost(deployment.getTokenUrl());
        ClientCredentialsProviderUtils.setClientCredentials(deployment, post, formparams);

        UrlEncodedFormEntity form = new UrlEncodedFormEntity(formparams, "UTF-8");
        post.setEntity(form);
        HttpResponse response = deployment.getClient().execute(post);
    ``` 

- トークンエンドポイントの処理  
  - org.keycloak.protocol.oidc.endpoints.TokenEndpoint.updateClientSession(AuthenticatedClientSessionModel)  
    ```java
    private void updateClientSession(AuthenticatedClientSessionModel clientSession) {

        if(clientSession == null) {
            ServicesLogger.LOGGER.clientSessionNull();
            return;
        }

        String adapterSessionId = formParams.getFirst(AdapterConstants.CLIENT_SESSION_STATE);
        if (adapterSessionId != null) {
            String adapterSessionHost = formParams.getFirst(AdapterConstants.CLIENT_SESSION_HOST);
            logger.debugf("Adapter Session '%s' saved in ClientSession for client '%s'. Host is '%s'", adapterSessionId, client.getClientId(), adapterSessionHost);

            event.detail(AdapterConstants.CLIENT_SESSION_STATE, adapterSessionId);
            String oldClientSessionState = clientSession.getNote(AdapterConstants.CLIENT_SESSION_STATE);
            if (!adapterSessionId.equals(oldClientSessionState)) {
                clientSession.setNote(AdapterConstants.CLIENT_SESSION_STATE, adapterSessionId);
            }

            event.detail(AdapterConstants.CLIENT_SESSION_HOST, adapterSessionHost);
            String oldClientSessionHost = clientSession.getNote(AdapterConstants.CLIENT_SESSION_HOST);
            if (!Objects.equals(adapterSessionHost, oldClientSessionHost)) {
                clientSession.setNote(AdapterConstants.CLIENT_SESSION_HOST, adapterSessionHost);
            }
        }
    }
    ```

- アダプタ共通
  - k_logout を処理
    - org.keycloak.adapters.PreAuthActionsHandler.handleLogout()
      - アダプタが実装している UserSessionManagement のログアウト用メソッドを呼び出す。

- Spring Security アダプタ用実装
  - k_logout を処理するフィルタ
    - org.keycloak.adapters.springsecurity.filter.KeycloakPreAuthActionsFilter
  - KeyCloak の UserSessionManagement の実装  
    - org.keycloak.adapters.springsecurity.management.HttpSessionManager
      - HttpSessionListener を利用して、すべてのセッションをメモリ上に記録することで実現している
      - → HttpSessionListener は Spring Session を使う場合は使えない。また、メモリに記録していることもあり、冗長構成では使えない。
  

## つまり

1. 認可コードからトークンエンドポイントにアクセスする際に以下のパラメータの設定が必要
   - `client_session_state` : Client Session Id
   - `client_session_host` : Client Session Host
2. k_logout 受信時に冗長化構成でも対応できる仕組みの構築が必要

Spring Security で認可コードからトークンエンドポイントにアクセスする際のパラメータの設定をカスタマイズするには、
`OAuth2AuthorizationCodeGrantRequestEntityConverter` のサブクラスを作成し、そこで設定することになる。
また、このサブクラスのインスタンスは以下のような感じで仕込むことができると思われる。

  1. `var responseClient = new DefaultAuthorizationCodeTokenResponseClient();`
  2. `responseClient.setRequestEntityConverter(new MyOAuth2AuthorizationCodeGrantRequestEntityConverter());`
  3. `http.oauth2Login().tokenEndpoint().accessTokenResponseClient(responseClient);`

