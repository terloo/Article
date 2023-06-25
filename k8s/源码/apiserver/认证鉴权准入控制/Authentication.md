# Authentication
客户端发起的请求，只要有一个认证成功，即允许访问

## 初始化
1. 初始化默认命令行参数，默认启用以下几种认证方式
   1. Anonymous
   2. Bootstrap
   3. ClientCA
   4. OIDC
   5. RequestHeader
   6. ServiceAccounts
   7. TokenFile
   8. WebHook
2. 绑定Cobra命令行参数，补全BuiltInAuthenticationOptions
3. 从BuiltInAuthenticationOptions生成authenticatorConfig
4. 从authenticatorConfig生成authenticator
5. 将所有的authenticator整合为一个
```go
// 1. 初始化默认命令行参数
// cmd/kube-apiserver/app/options/options.go
func NewServerRunOptions() *ServerRunOptions {
	s := ServerRunOptions{
        // ...
		Authentication:          kubeoptions.NewBuiltInAuthenticationOptions().WithAll(),
		// ...
	}

	// ...

	return &s
}

// 2. 绑定Cobra命令行参数
// cmd/kube-apiserver/app/options/options.go
func (s *ServerRunOptions) Flags() (fss cliflag.NamedFlagSets) {
	// ...
    // 字段Authentication是BuiltInAuthenticationOptions
	s.Authentication.AddFlags(fss.FlagSet("authentication"))
    // ...
}

// 3. 从BuiltInAuthenticationOptions生成authenticatorConfig
// pkg/kubeapiserver/options/authentication.go
func (o *BuiltInAuthenticationOptions) ApplyTo(authInfo *genericapiserver.AuthenticationInfo, secureServing *genericapiserver.SecureServingInfo, egressSelector *egressselector.EgressSelector, openAPIConfig *openapicommon.Config, extclient kubernetes.Interface, versionedInformer informers.SharedInformerFactory) error {
	if o == nil {
		return nil
	}

	if openAPIConfig == nil {
		return errors.New("uninitialized OpenAPIConfig")
	}

	authenticatorConfig, err := o.ToAuthenticationConfig()
	if err != nil {
		return err
	}

    // ClientCA配置
	if authenticatorConfig.ClientCAContentProvider != nil {
		if err = authInfo.ApplyClientCert(authenticatorConfig.ClientCAContentProvider, secureServing); err != nil {
			return fmt.Errorf("unable to load client CA file: %v", err)
		}
	}
    // RequestHeader配置
	if authenticatorConfig.RequestHeaderConfig != nil && authenticatorConfig.RequestHeaderConfig.CAContentProvider != nil {
		if err = authInfo.ApplyClientCert(authenticatorConfig.RequestHeaderConfig.CAContentProvider, secureServing); err != nil {
			return fmt.Errorf("unable to load client CA file: %v", err)
		}
	}

	authInfo.APIAudiences = o.APIAudiences
	if o.ServiceAccounts != nil && len(o.ServiceAccounts.Issuers) != 0 && len(o.APIAudiences) == 0 {
		authInfo.APIAudiences = authenticator.Audiences(o.ServiceAccounts.Issuers)
	}

    // ServiceAccounts配置
	authenticatorConfig.ServiceAccountTokenGetter = serviceaccountcontroller.NewGetterFromClient(
		extclient,
		versionedInformer.Core().V1().Secrets().Lister(),
		versionedInformer.Core().V1().ServiceAccounts().Lister(),
		versionedInformer.Core().V1().Pods().Lister(),
	)

    // Bootstrap配置
	authenticatorConfig.BootstrapTokenAuthenticator = bootstrap.NewTokenAuthenticator(
		versionedInformer.Core().V1().Secrets().Lister().Secrets(metav1.NamespaceSystem),
	)

	if egressSelector != nil {
		egressDialer, err := egressSelector.Lookup(egressselector.ControlPlane.AsNetworkContext())
		if err != nil {
			return err
		}
		authenticatorConfig.CustomDial = egressDialer
	}

    // 从配置生成authenticator
	authInfo.Authenticator, openAPIConfig.SecurityDefinitions, err = authenticatorConfig.New()
	if err != nil {
		return err
	}

	return nil
}

// 4. 从authenticatorConfig生成authenticator
// pkg/kubeapiserver/authenticator/config.go
func (config Config) New() (authenticator.Request, *spec.SecurityDefinitions, error) {
	var authenticators []authenticator.Request
    // authenticator.Token是所有authenticator的接口
    // tokenAuthenticators保存了所有实例化的authenticator
	var tokenAuthenticators []authenticator.Token
	securityDefinitions := spec.SecurityDefinitions{}

	// front-proxy, BasicAuth methods, local first, then remote
	// Add the front proxy authenticator if requested
	if config.RequestHeaderConfig != nil {
		requestHeaderAuthenticator := headerrequest.NewDynamicVerifyOptionsSecure(
			config.RequestHeaderConfig.CAContentProvider.VerifyOptions,
			config.RequestHeaderConfig.AllowedClientNames,
			config.RequestHeaderConfig.UsernameHeaders,
			config.RequestHeaderConfig.GroupHeaders,
			config.RequestHeaderConfig.ExtraHeaderPrefixes,
		)
		authenticators = append(authenticators, authenticator.WrapAudienceAgnosticRequest(config.APIAudiences, requestHeaderAuthenticator))
	}

	// X509 methods
	if config.ClientCAContentProvider != nil {
		certAuth := x509.NewDynamic(config.ClientCAContentProvider.VerifyOptions, x509.CommonNameUserConversion)
		authenticators = append(authenticators, certAuth)
	}

	// Bearer token methods, local first, then remote
	if len(config.TokenAuthFile) > 0 {
		tokenAuth, err := newAuthenticatorFromTokenFile(config.TokenAuthFile)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, tokenAuth))
	}
	if len(config.ServiceAccountKeyFiles) > 0 {
		serviceAccountAuth, err := newLegacyServiceAccountAuthenticator(config.ServiceAccountKeyFiles, config.ServiceAccountLookup, config.APIAudiences, config.ServiceAccountTokenGetter)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, serviceAccountAuth)
	}
	if len(config.ServiceAccountIssuers) > 0 {
		serviceAccountAuth, err := newServiceAccountAuthenticator(config.ServiceAccountIssuers, config.ServiceAccountKeyFiles, config.APIAudiences, config.ServiceAccountTokenGetter)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, serviceAccountAuth)
	}
	if config.BootstrapToken {
		if config.BootstrapTokenAuthenticator != nil {
			// TODO: This can sometimes be nil because of
			tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, config.BootstrapTokenAuthenticator))
		}
	}
	// NOTE(ericchiang): Keep the OpenID Connect after Service Accounts.
	//
	// Because both plugins verify JWTs whichever comes first in the union experiences
	// cache misses for all requests using the other. While the service account plugin
	// simply returns an error, the OpenID Connect plugin may query the provider to
	// update the keys, causing performance hits.
	if len(config.OIDCIssuerURL) > 0 && len(config.OIDCClientID) > 0 {
		// TODO(enj): wire up the Notifier and ControllerRunner bits when OIDC supports CA reload
		var oidcCAContent oidc.CAContentProvider
		if len(config.OIDCCAFile) != 0 {
			var oidcCAErr error
			oidcCAContent, oidcCAErr = dynamiccertificates.NewDynamicCAContentFromFile("oidc-authenticator", config.OIDCCAFile)
			if oidcCAErr != nil {
				return nil, nil, oidcCAErr
			}
		}

		oidcAuth, err := newAuthenticatorFromOIDCIssuerURL(oidc.Options{
			IssuerURL:            config.OIDCIssuerURL,
			ClientID:             config.OIDCClientID,
			CAContentProvider:    oidcCAContent,
			UsernameClaim:        config.OIDCUsernameClaim,
			UsernamePrefix:       config.OIDCUsernamePrefix,
			GroupsClaim:          config.OIDCGroupsClaim,
			GroupsPrefix:         config.OIDCGroupsPrefix,
			SupportedSigningAlgs: config.OIDCSigningAlgs,
			RequiredClaims:       config.OIDCRequiredClaims,
		})
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, oidcAuth))
	}
	if len(config.WebhookTokenAuthnConfigFile) > 0 {
		webhookTokenAuth, err := newWebhookTokenAuthenticator(config)
		if err != nil {
			return nil, nil, err
		}

		tokenAuthenticators = append(tokenAuthenticators, webhookTokenAuth)
	}

	if len(tokenAuthenticators) > 0 {
		// Union the token authenticators
		tokenAuth := tokenunion.New(tokenAuthenticators...)
		// Optionally cache authentication results
		if config.TokenSuccessCacheTTL > 0 || config.TokenFailureCacheTTL > 0 {
			tokenAuth = tokencache.New(tokenAuth, true, config.TokenSuccessCacheTTL, config.TokenFailureCacheTTL)
		}
		authenticators = append(authenticators, bearertoken.New(tokenAuth), websocket.NewProtocolAuthenticator(tokenAuth))
		securityDefinitions["BearerToken"] = &spec.SecurityScheme{
			SecuritySchemeProps: spec.SecuritySchemeProps{
				Type:        "apiKey",
				Name:        "authorization",
				In:          "header",
				Description: "Bearer Token authentication",
			},
		}
	}

	if len(authenticators) == 0 {
		if config.Anonymous {
			return anonymous.NewAuthenticator(), &securityDefinitions, nil
		}
		return nil, &securityDefinitions, nil
	}

    // 最后将所有的authenticator组合为一个
	authenticator := union.New(authenticators...)

	authenticator = group.NewAuthenticatedGroupAdder(authenticator)

	if config.Anonymous {
		// If the authenticator chain returns an error, return an error (don't consider a bad bearer token
		// or invalid username/password combination anonymous).
		authenticator = union.NewFailOnError(authenticator, anonymous.NewAuthenticator())
	}

	return authenticator, &securityDefinitions, nil
}

// 5. 将所有的authenticator整合为一个，unionAuthRequetHandlers会遍历所有authenticator执行一次
// vendor/k8s.io/apiserver/pkg/authentication/reuqest/union/union.go
func New(authRequestHandlers ...authenticator.Request) authenticator.Request {
	if len(authRequestHandlers) == 1 {
		return authRequestHandlers[0]
	}
	// 配置为报错不失败模式
	return &unionAuthRequestHandler{Handlers: authRequestHandlers, FailOnError: false}
}
```

## unionAuthRequetHandlers
```go
// vendor/k8s.io/apiserver/pkg/authentication/request/union/union.go
type unionAuthRequestHandler struct {
	// Handlers is a chain of request authenticators to delegate to
	Handlers []authenticator.Request
	// FailOnError determines whether an error returns short-circuits the chain
	FailOnError bool
}

func (authHandler *unionAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	var errlist []error
	for _, currAuthRequestHandler := range authHandler.Handlers {
		resp, ok, err := currAuthRequestHandler.AuthenticateRequest(req)
		if err != nil {
			if authHandler.FailOnError {
				return resp, ok, err
			}
			errlist = append(errlist, err)
			continue
		}

		// 只要有一个Authenticator成功就返回成功
		if ok {
			return resp, ok, err
		}
	}

	return nil, false, utilerrors.NewAggregate(errlist)
}
```

## 认证
```go
// vendor/k8s.io/apiserver/pkg/endpoints/filter/authentication.go
func WithAuthentication(handler http.Handler, auth authenticator.Request, failed http.Handler, apiAuds authenticator.Audiences) http.Handler {
	return withAuthentication(handler, auth, failed, apiAuds, recordAuthMetrics)
}

// auth认证器
func withAuthentication(handler http.Handler, auth authenticator.Request, failed http.Handler, apiAuds authenticator.Audiences, metrics recordMetrics) http.Handler {
	if auth == nil {
		klog.Warning("Authentication is disabled")
		return handler
	}
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		authenticationStart := time.Now()

		if len(apiAuds) > 0 {
			req = req.WithContext(authenticator.WithAudiences(req.Context(), apiAuds))
		}
		// 调用认证器进行认证
		resp, ok, err := auth.AuthenticateRequest(req)
		authenticationFinish := time.Now()
		defer func() {
			metrics(req.Context(), resp, ok, err, apiAuds, authenticationStart, authenticationFinish)
		}()
		if err != nil || !ok {
			if err != nil {
				klog.ErrorS(err, "Unable to authenticate the request")
			}
			failed.ServeHTTP(w, req)
			return
		}

		if !audiencesAreAcceptable(apiAuds, resp.Audiences) {
			err = fmt.Errorf("unable to match the audience: %v , accepted: %v", resp.Audiences, apiAuds)
			klog.Error(err)
			failed.ServeHTTP(w, req)
			return
		}

		// authorization header is not required anymore in case of a successful authentication.
		req.Header.Del("Authorization")

		req = req.WithContext(genericapirequest.WithUser(req.Context(), resp.User))
		handler.ServeHTTP(w, req)
	})
}
```