## 목차
- [목차](#목차)
- [clientcmd.BuildConfigFromFlags](#clientcmdbuildconfigfromflags)
  - [함수 뎁스](#함수-뎁스)
  - [InClusterConfig](#inclusterconfig)
  - [DeferredLoadingClientConfig.ClientConfig](#deferredloadingclientconfigclientconfig)
  
##  clientcmd.BuildConfigFromFlags
- 쿠버네티스 API 서버와 통신하기 위한 설정을 생성

```golang
func BuildConfigFromFlags(masterUrl, kubeconfigPath string) (*restclient.Config, error) {
	if kubeconfigPath == "" && masterUrl == "" {
		klog.Warning("Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.")
		kubeconfig, err := restclient.InClusterConfig()
		if err == nil {
			return kubeconfig, nil
		}
		klog.Warning("error creating inClusterConfig, falling back to default config: ", err)
	}
	return NewNonInteractiveDeferredLoadingClientConfig(
		&ClientConfigLoadingRules{ExplicitPath: kubeconfigPath},
		&ConfigOverrides{ClusterInfo: clientcmdapi.Cluster{Server: masterUrl}}).ClientConfig()
}
```
- masterURL, kubeconfigPath가 둘다 빈 스트링인 경우, `restclient.InClusterConfig()` 함수 호출

### 함수 뎁스
- BuildConfigFromFlags
  - InClusterConfig
  - NewNonInteractiveDeferredLoadingClientConfig
    - DeferredLoadingClientConfig.ClientConfig
      

### InClusterConfig
```golang
func InClusterConfig() (*Config, error) {
	const (
		tokenFile  = "/var/run/secrets/kubernetes.io/serviceaccount/token"
		rootCAFile = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
	)
	host, port := os.Getenv("KUBERNETES_SERVICE_HOST"), os.Getenv("KUBERNETES_SERVICE_PORT")
	if len(host) == 0 || len(port) == 0 {
		return nil, ErrNotInCluster
	}

	token, err := os.ReadFile(tokenFile)
	if err != nil {
		return nil, err
	}

	tlsClientConfig := TLSClientConfig{}

	if _, err := certutil.NewPool(rootCAFile); err != nil {
		klog.Errorf("Expected to load root CA config from %s, but got err: %v", rootCAFile, err)
	} else {
		tlsClientConfig.CAFile = rootCAFile
	}

	return &Config{
		// TODO: switch to using cluster DNS.
		Host:            "https://" + net.JoinHostPort(host, port),
		TLSClientConfig: tlsClientConfig,
		BearerToken:     string(token),
		BearerTokenFile: tokenFile,
	}, nil
}
```
- 로컬 경로에 설치된 BearerToken, 인증서 파일을 탐색
- 환경 변수를 통해 쿠버네티스 API 서버의 주소와 포트를 탐색
- API 서버 정보, 인증 정보를 포함한 Config 객체 생성 후 반환

### DeferredLoadingClientConfig.ClientConfig
```golang
func NewNonInteractiveDeferredLoadingClientConfig(loader ClientConfigLoader, overrides *ConfigOverrides) ClientConfig {
	return &DeferredLoadingClientConfig{loader: loader, overrides: overrides, icc: &inClusterClientConfig{overrides: overrides}}
}
```

```golang
func (config *DeferredLoadingClientConfig) ClientConfig() (*restclient.Config, error) {

	mergedClientConfig, err := config.createClientConfig()
	if err != nil {
		return nil, err
	}
	// load the configuration and return on non-empty errors and if the
	// content differs from the default config
	mergedConfig, err := mergedClientConfig.ClientConfig()
	switch {
	case err != nil:
		if !IsEmptyConfig(err) {
			// return on any error except empty config
			return nil, err
		}
	case mergedConfig != nil:
		// the configuration is valid, but if this is equal to the defaults we should try
		// in-cluster configuration
		if !config.loader.IsDefaultConfig(mergedConfig) {
			return mergedConfig, nil
		}
	}

	// check for in-cluster configuration and use it
	if config.icc.Possible() {
		klog.V(4).Infof("Using in-cluster configuration")
		return config.icc.ClientConfig()
	}

	// return the result of the merged client config
	return mergedConfig, err
}
```
- `createClientConfig` 메소드를 호출하여 설정 파일을 생성
- 생성된 설정을 검증하고, 검증이 완료된 경우 반환


## NewForConfig
```go
func NewForConfig(c *rest.Config) (*Clientset, error) {
	configShallowCopy := *c

	if configShallowCopy.UserAgent == "" {
		configShallowCopy.UserAgent = rest.DefaultKubernetesUserAgent()
	}

	// share the transport between all clients
	httpClient, err := rest.HTTPClientFor(&configShallowCopy)
	if err != nil {
		return nil, err
	}

	return NewForConfigAndClient(&configShallowCopy, httpClient)
}
```
- `HTTPClientFor`: config 값을 통해 서버에 인증할 클라이언트 생성
- `NewForConfigAndClient`: 인증 클라이언트와 클라이언트 설정을 통해 API 리소스 클라이언트 생성
- API 리소스를 가지는 클라이언트 반환

### 함수 깊이
- NewForConfig
  - HTTPClientFor
    - TransportFor
      - config.TransportConfig
    - transport.New
  - NewForConfigAndClient
    - <api-group-version>.NewForConfigAndClient
    - RESTClientForConfigAndClient

### HTTPClientFor(서버 인증 클라이언트 생성)
```go
// 1. HTTPClientFor
func HTTPClientFor(config *Config) (*http.Client, error) {
	transport, err := TransportFor(config)

	if err != nil {
		return nil, err
	}
	var httpClient *http.Client
	if transport != http.DefaultTransport || config.Timeout > 0 {
		httpClient = &http.Client{
			Transport: transport,
			Timeout:   config.Timeout,
		}
	} else {
		httpClient = http.DefaultClient
	}

	return httpClient, nil
}

// 2. TransportFor
func TransportFor(config *Config) (http.RoundTripper, error) {
	cfg, err := config.TransportConfig()
	if err != nil {
		return nil, err
	}
	return transport.New(cfg)
}

// 3. config.TransportConfig
func (c *Config) TransportConfig() (*transport.Config, error) {
	conf := &transport.Config{
		UserAgent:          c.UserAgent,
		Transport:          c.Transport,
		WrapTransport:      c.WrapTransport,
		DisableCompression: c.DisableCompression,
		TLS: transport.TLSConfig{
			Insecure:   c.Insecure,
			ServerName: c.ServerName,
			CAFile:     c.CAFile,
			CAData:     c.CAData,
			CertFile:   c.CertFile,
			CertData:   c.CertData,
			KeyFile:    c.KeyFile,
			KeyData:    c.KeyData,
			NextProtos: c.NextProtos,
		},
		Username:        c.Username,
		Password:        c.Password,
		BearerToken:     c.BearerToken,
		BearerTokenFile: c.BearerTokenFile,
		Impersonate: transport.ImpersonationConfig{
			UserName: c.Impersonate.UserName,
			UID:      c.Impersonate.UID,
			Groups:   c.Impersonate.Groups,
			Extra:    c.Impersonate.Extra,
		},
		Proxy: c.Proxy,
	}
    ...
	return conf, nil
}

// 4. transport.New
func New(config *Config) (http.RoundTripper, error) {
	// Set transport level security
	if config.Transport != nil && (config.HasCA() || config.HasCertAuth() || config.HasCertCallback() || config.TLS.Insecure) {
		return nil, fmt.Errorf("using a custom transport with TLS certificate options or the insecure flag is not allowed")
	}

	if !isValidHolders(config) {
		return nil, fmt.Errorf("misconfigured holder for dialer or cert callback")
	}

	var (
		rt  http.RoundTripper
		err error
	)

	if config.Transport != nil {
		rt = config.Transport
	} else {
		rt, err = tlsCache.get(config)
		if err != nil {
			return nil, err
		}
	}

	return HTTPWrappersForConfig(config, rt)
}
```
- 쿠버네티스 설정(config)를 TransportConfig로 변환
  - HTTP 형식으로 보내기 위한 형식으로 변환
- `transport.New` 함수를 통해 인증 정보(인증서, token)를 포함한 클라이언트 랩핑
- `HTTPClientFor` 함수에서 통해 클라이언트 리턴


### NewForConfigAndClient(API 리소스 클라이언트 생성)
```go
// 1. NewForConfigAndClient
func NewForConfigAndClient(c *rest.Config, httpClient *http.Client) (*Clientset, error) {
	configShallowCopy := *c
	...
	var cs Clientset
	var err error
	cs.appsV1, err = appsv1.NewForConfigAndClient(&configShallowCopy, httpClient)
	if err != nil {
		return nil, err
	}
	cs.appsV1beta1, err = appsv1beta1.NewForConfigAndClient(&configShallowCopy, httpClient)
	if err != nil {
		return nil, err
	}
	cs.appsV1beta2, err = appsv1beta2.NewForConfigAndClient(&configShallowCopy, httpClient)
	if err != nil {
		return nil, err
	}
	...
	return &cs, nil
}

// 2. <api-group-version>.NewForConfigAndClient
func NewForConfigAndClient(c *rest.Config, h *http.Client) (*AppsV1Client, error) {
	config := *c
	if err := setConfigDefaults(&config); err != nil {
		return nil, err
	}
	client, err := rest.RESTClientForConfigAndClient(&config, h)
	if err != nil {
		return nil, err
	}
	return &AppsV1Client{client}, nil
}

// 3. RESTClientForConfigAndClient
func RESTClientForConfigAndClient(config *Config, httpClient *http.Client) (*RESTClient, error) {
	if config.GroupVersion == nil {
		return nil, fmt.Errorf("GroupVersion is required when initializing a RESTClient")
	}
	if config.NegotiatedSerializer == nil {
		return nil, fmt.Errorf("NegotiatedSerializer is required when initializing a RESTClient")
	}

	baseURL, versionedAPIPath, err := DefaultServerUrlFor(config)
	if err != nil {
		return nil, err
	}

	var gv schema.GroupVersion
	if config.GroupVersion != nil {
		gv = *config.GroupVersion
	}
	clientContent := ClientContentConfig{
		AcceptContentTypes: config.AcceptContentTypes,
		ContentType:        config.ContentType,
		GroupVersion:       gv,
		Negotiator:         runtime.NewClientNegotiator(config.NegotiatedSerializer, gv),
	}

	restClient, err := NewRESTClient(baseURL, versionedAPIPath, clientContent, rateLimiter, httpClient)
	if err == nil && config.WarningHandler != nil {
		restClient.warningHandler = config.WarningHandler
	}
	return restClient, err
}
```
- `NewForConfigAndClient` 함수를 통해 API 리소스 클라이언트 생성
- API 리소스 클라이언트 생성 시, `<api-group-version>.NewForConfigAndClient` 함수 호출
  - `setConfigDefaults` 함수로 기본 설정 값 설정
  - `RESTClientForConfigAndClient` 함수로 REST 클라이언트 생성
