
## watcher

### 낙관적 잠금
- 낙관적 잠금은 데이터 변경이 발생할 때 리소스버전이 etcd에 있는 리소스버전과 일치해야 데이터가 변경됨을 의미
- 리소스버전이 일치하지 않은 경우 변경 요청은 거부되고 `status 409` 반환

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	"os"
	"path/filepath"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"

)

func main() {
	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	deploymentsClient := clientset.AppsV1().Deployments(apiv1.NamespaceDefault)
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
				Spec: apiv1.PodSpec{
					Containers: []apiv1.Container{
						{
							Name:  "web",
							Image: "nginx:1.12",
							Ports: []apiv1.ContainerPort{
								{
									Name:          "http",
									Protocol:      apiv1.ProtocolTCP,
									ContainerPort: 80,
								},
							},
						},
					},
				},
			},
		},
	}

	_, err = deploymentsClient.Create(context.TODO(), deployment, metav1.CreateOptions{})
	if err != nil {
		panic(err)
	}

	time.Sleep(5 * time.Second)
	// 생성된 Deployment 조회
	originDeploy, err := deploymentsClient.Get(context.TODO(), "demo-deployment", metav1.GetOptions{})
	if err != nil {
		panic(fmt.Errorf("Failed to get latest version of Deployment: %v", err))
	}
	// 생성된 Deployment의 리소스버전 조회
	fmt.Println("origin ResourceVersion: ", originDeploy.ResourceVersion)

	originDeploy.Spec.Replicas = int32Ptr(2)                           // reduce replica count
	originDeploy.Spec.Template.Spec.Containers[0].Image = "nginx:1.13" // change nginx version
	originDeploy.Annotations = map[string]string{
		"test-v1": "test-v1",
	}

	// 첫 번째 업데이트
	updatedDeploy, err := deploymentsClient.Update(context.TODO(), originDeploy, metav1.UpdateOptions{})
	if err != nil {
		panic(fmt.Errorf("[f] Failed to update Deployment: %v", err))
	}
	// 업데이트된 Deployment의 리소스버전 조회
	fmt.Println("updated ResourceVersion: ", updatedDeploy.ResourceVersion)

	originDeploy.Spec.Replicas = int32Ptr(5)                           // reduce replica count
	originDeploy.Spec.Template.Spec.Containers[0].Image = "nginx:1.14" // change nginx version
	originDeploy.Annotations = map[string]string{
		"version": "update-1",
	}

	// 두 번째 업데아트(첫 리소스 버전을 가지고 있으므로 에러 발생)
	_, err = deploymentsClient.Update(context.TODO(), originDeploy, metav1.UpdateOptions{})
	if err != nil {
		panic(fmt.Errorf("[s] Failed to update Deployment: %v", err))
	}
}
```
```
origin ResourceVersion:  17217419
updated ResourceVersion:  17217442
panic: [s] Failed to update Deployment: Operation cannot be fulfilled on deployments.apps "demo-deployment": the object has been modified; please apply your changes to the latest version and try again
```
- 첫 번째 업데이트는 리소스버전이 일치하여 업데이트 성공
- 두 번째 업데이트는 리소스버전이 일치하지 않아 업데이트 실패

### watcher 메소드
```go
// 1. watch 메소드
func (c *Client[T]) Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	opts.Watch = true
	return c.client.Get().
		NamespaceIfScoped(c.namespace, c.namespace != "").
		Resource(c.resource).
		VersionedParams(&opts, c.parameterCodec).
		Timeout(timeout).
		Watch(ctx)
}

// 2. Watch 메소드(요청)
func (r *Request) Watch(ctx context.Context) (watch.Interface, error) {
	if r.err != nil {
		return nil, r.err
	}

	// 클라이언트가 없으면 기본 클라이언트
	// 하지만 transport가 없으므로 인증 정보가 없어서 에러가 난다
	client := r.c.Client
	if client == nil {
		client = http.DefaultClient
	}

	// 재시도 여부 결정
	isErrRetryableFunc := func(request *http.Request, err error) bool {
		if net.IsProbableEOF(err) || net.IsTimeout(err) {
			return true
		}
		return false
	}

	// 디폴트 값은 10
	retry := r.retryFn(r.maxRetries)
	for {
		// 요청을 보내기 전 필요한 작업 수행
		// 재시도 전에 일정 시간 간격을 조정
		// After에서 retryAfter 값을 수정하고 이 값을 통해 재시도인 경우 시간을 계산
		if err := retry.Before(ctx, r); err != nil {
			return nil, retry.WrapPreviousError(err)
		}

		req, err := r.newHTTPRequest(ctx)
		if err != nil {
			return nil, err
		}
		
		// http/2 로 요청을 보내기 때문에 스트림이 맺어짐
		// http/1.1에서 Connection: keep-alive를 사용하면 커넥션을 재사용할 수 있음
		// http/1.1에서 Transfer-Encoding: chunked 데이터를 청크(프레임) 단위로 받을 수 있음
		resp, err := client.Do(req)
		retry.After(ctx, r, resp, err)

		if err == nil && resp.StatusCode == http.StatusOK {
			return r.newStreamWatcher(resp)
		}

		done, transformErr := func() (bool, error) {

			// 바디의 데이터를 읽고 닫음
			defer readAndCloseResponseBody(resp)

			// 재시도가 필요한지 확인
			if retry.IsNextRetry(ctx, r, req, resp, err, isErrRetryableFunc) {
				return false, nil
			}

			if resp == nil {
				return true, nil
			}
			if result := r.transformResponse(resp, req); result.err != nil {
				return true, result.err
			}
			return true, fmt.Errorf("for request %s, got status: %v", url, resp.StatusCode)
		}()

		if done {
			if isErrRetryableFunc(req, err) {
				return watch.NewEmptyWatch(), nil
			}
			if err == nil {
				err = transformErr
			}
			return nil, retry.WrapPreviousError(err)
		}
	}
}

// 3. newStreamWatcher 메소드
func (r *Request) newStreamWatcher(resp *http.Response) (watch.Interface, error) {

	contentType := resp.Header.Get("Content-Type")
	mediaType, params, err := mime.ParseMediaType(contentType)

	if err != nil {
		klog.V(4).Infof("Unexpected content type from the server: %q: %v", contentType, err)
	}

	// 스트림 디코더
	// 데이터를 해석할 수 있는 단위(프레임)으로 데이터를 수신
	objectDecoder, streamingSerializer, framer, err := r.c.content.Negotiator.StreamDecoder(mediaType, params)
	if err != nil {
		return nil, err
	}

	handleWarnings(resp.Header, r.warningHandler)

	frameReader := framer.NewFrameReader(resp.Body)
	// 시리얼라이저는 인코딩/디코딩할 수 있는 메타정보를 가지고 있음 그래서 프레임 디코딩에 필요
	watchEventDecoder := streaming.NewDecoder(frameReader, streamingSerializer)

	// 데이터를 프레임 단위로 처리하는 디코더를 받는 스트림 왓쳐 생성
	return watch.NewStreamWatcher(
		restclientwatch.NewDecoder(watchEventDecoder, objectDecoder),
		errors.NewClientErrorReporter(http.StatusInternalServerError, r.verb, "ClientWatchDecoding"),
	), nil

func (sw *StreamWatcher) receive() {
	defer utilruntime.HandleCrash()
	defer close(sw.result)
	defer sw.Stop()
	for {
		action, obj, err := sw.source.Decode()
		if err != nil {
			switch err {
			case io.EOF:
				// watch closed normally
			case io.ErrUnexpectedEOF:
				klog.V(1).Infof("Unexpected EOF during watch stream event decoding: %v", err)
			default:
				if net.IsProbableEOF(err) || net.IsTimeout(err) {
					klog.V(5).Infof("Unable to decode an event from the watch stream: %v", err)
				} else {
					select {
					case <-sw.done:
					case sw.result <- Event{
						Type:   Error,
						Object: sw.reporter.AsObject(fmt.Errorf("unable to decode an event from the watch stream: %v", err)),
					}:
					}
				}
			}
			return
		}
		select {
		case <-sw.done:
			return
		case sw.result <- Event{
			Type:   action,
			Object: obj,
		}:
		}
	}
}
```
- `watch` 메소드를 통해 쿠버네티스 API 서버에 요청을 보내고 `StreamWatcher`를 반환
- http/2 를 사용하여 요청을 보내기 때문에 스트림이 맺어짐
- 스트림왓쳐의 `receive` 고루틴이 데이터를 수신하고 `sw.result` 채널로 데이터를 전달

```go
for event := range watcher.ResultChan() {
		if pod, ok := event.Object.(*corev1.Pod); ok {
			klog.Infof("%s: %s %s", event.Type, pod.GetName())
		}
	}
```
- `watcher.ResultChan()`을 통해 `StreamWatcher`의 `result` 채널을 대기하고 데이터를 수신
- 수신된 데이터를 이벤트 타입과 객체로 변환하여 사용

### 이벤트 타입
```go
type Event struct {
	Type EventType
	Object runtime.Object
}

type EventType string

const (
Added    EventType = "ADDED"
Modified EventType = "MODIFIED"
Deleted  EventType = "DELETED"
Bookmark EventType = "BOOKMARK"
Error    EventType = "ERROR"
)
```
- `Event` 구조체는 이벤트 타입을 가짐
- 이벤트 타입은 `ADDED`, `MODIFIED`, `DELETED`, `BOOKMARK`, `ERROR`가 있음
- ADDED, MODIFIED
  - 새로운 리소스가 생성되었거나, 리소스의 상태가 변경되었을 때 발생
- DELETED
  - 리소스가 삭제되었을 때 발생
- ERROR
  - 이벤트 수신 중 에러가 발생했을 때 발생
- BOOKMARK
  - 현재 수신된 이벤트의 위치를 나타냄
  - ResourceVersion을 기준으로 이벤트를 수신하고, BOOKMARK 이벤트를 수신하면 해당 ResourceVersion을 저장하여 다음 이벤트를 수신할 때 사용

```go
package main

import (
	"context"
	"flag"
	"path/filepath"
	"time"

	corev1 "k8s.io/api/core/v1"

	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/klog/v2"
)

func main() {
	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	client, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	podWatcher, err := client.CoreV1().Pods("default").
		Watch(context.TODO(), metav1.ListOptions{
			AllowWatchBookmarks: true,
		})

	if err != nil {
		klog.Fatal(err)
	}
	var lastResourceVersion string

	go func() {
		for event := range podWatcher.ResultChan() {
			// 북마크 이벤트를 수신하면 리소스버전 갱신
			if event.Type == "BOOKMARK" {
				klog.Infof("[POD] BOOKMARK: %s", event.Object.(*corev1.Pod).GetResourceVersion())
				lastResourceVersion = event.Object.(*corev1.Pod).GetResourceVersion()
			} else if pod, ok := event.Object.(*corev1.Pod); ok {
				klog.Infof("[POD] %s: %s", event.Type, pod.GetName())
				// 이벤트 수신 시마다 리소스버전 갱신
				lastResourceVersion = pod.GetResourceVersion()
			}
		}
	}()

	time.Sleep(5 * time.Second)
	podWatcher.Stop()

	/*
		이 시간에 새로운 파드를 생성/삭제
	*/
	
	time.Sleep(20 * time.Second)

	// 북마크 이벤트 간격은 10분(default)
	podWatcher, err = client.CoreV1().Pods("default").
		Watch(context.TODO(), metav1.ListOptions{
			ResourceVersion:     lastResourceVersion,
			AllowWatchBookmarks: true,
		})
	if err != nil {
		klog.Fatal(err)
	}

	for event := range podWatcher.ResultChan() {
		if event.Type == "BOOKMARK" {
			klog.Infof("[POD] BOOKMARK: %s", event.Object.(*corev1.Pod).GetResourceVersion())
			lastResourceVersion = event.Object.(*corev1.Pod).GetResourceVersion()
		} else if pod, ok := event.Object.(*corev1.Pod); ok {
			klog.Infof("[POD] %s: %s", event.Type, pod.GetName())
			lastResourceVersion = pod.GetResourceVersion()
		}
	}

	// close watcher
	podWatcher.Stop()
}

```
- Bookmarks 이벤트 예제
- 첫 `watch` 메소드 사용 시, ResourceVersion을 갱신하고, 북마크 이벤트를 수신하면 해당 ResourceVersion을 저장
- 연결이 끊기고 다시 `watch` 메소드 사용 시, ResourceVersion 이후 이벤트를 수신하고,: 북마크 이벤트를 수신하면 해당 ResourceVersion을 저장