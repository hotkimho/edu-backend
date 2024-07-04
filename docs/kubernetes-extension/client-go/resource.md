## 목차
- [리소스 생성](#리소스-생성)
- [리소스 조회](#리소스-조회)
- [리소스 수정](#리소스-수정)
- [리소스 삭제](#리소스-삭제)

### 리소스 클라이언트 생성(deployment)
```go
// 1. 리소스 클라이언트 생성
    deploymentsClient := clientset.AppsV1().Deployments(apiv1.NamespaceDefault)
        
    // 2. 리소스 구현체 반환	
    func (c *AppsV1Client) Deployments(namespace string) DeploymentInterface {
        return newDeployments(c, namespace)
    }
    
    func newDeployments(c *AppsV1Client, namespace string) *deployments {
        // 디플로이먼트 구조체를 생성 후 반환
        return &deployments{
            gentype.NewClientWithListAndApply[*v1.Deployment, *v1.DeploymentList, *appsv1.DeploymentApplyConfiguration](
                "deployments",
                c.RESTClient(),
                scheme.ParameterCodec,
                namespace,
                func() *v1.Deployment { return &v1.Deployment{} },
                func() *v1.DeploymentList { return &v1.DeploymentList{} }),
        }
    }
```
- 클라이언트셋 그룹에 명시된 리소스의 클라이언트 생성
- <clientset>.<api-group-version>().<리소스명>(namespace)으로 리소스 구현체 반환


### 리소스 생성
```go
    // 1. 생성할 리소스 선언
	deployment := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name: "demo-deployment",
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: int32Ptr(3),
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": "demo",
				},
			},
			Template: apiv1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"app": "demo",
					},
				},
				...
	}
	
	// 2. 리소스 생성
	result, err := deploymentsClient.Create(context.TODO(), deployment, metav1.CreateOptions{})
	
	// 3. create 메소드
	func (c *Client[T]) Create(ctx context.Context, obj T, opts metav1.CreateOptions) (T, error) {
	// 생성된 데이터를 담을 객체 생성
	result := c.newObject()
	err := c.client.Post().
		NamespaceIfScoped(c.namespace, c.namespace != ""). // 네임스페이스 설정
		Resource(c.resource).							   // 리소스 타입 명시
		VersionedParams(&opts, c.parameterCodec).          // 인코딩 설정
		Body(obj).										   // 생성할 데이터
		Do(ctx).									       // API 서버에 요청
		Into(result)									   // 결과값을 result 객체에 저장
	return result, err
}
```
- 리소스 클라이언트에 정의된 `Create` 메소드를 사용하여 리소스 생성
- 리소스 생성시 `create(context, 리소스 선언, 생성 옵션)`을 사용하여 리소스 생성
  - 생성 옵션은 다음과 같음
    - DryRun: 리소스 생성 전에 유효성 검사만 수행(실제 생성되지 않고 시뮬레이션만 진행) ex) `DryRun: []string{"All"}
- API 서버에 요청을 보낼 때 method, namespace, 리소스, 인코딩, 데이터를 서버에 보내고 응답을 받음