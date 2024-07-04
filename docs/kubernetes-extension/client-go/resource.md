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
