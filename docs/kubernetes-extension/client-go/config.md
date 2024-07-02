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