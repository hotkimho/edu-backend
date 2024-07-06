
## watcher

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
