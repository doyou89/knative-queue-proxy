# knative-queue-proxy

ai 모델 추론 플랫폼인 knative 는 모델을 서빙하는 각 포드에 queue-proxy 라는 사이드카를 추가해서 트래픽을 관리해 주는데, queue-proxy 에 과도한 트래픽이 인가되면 queue-proxy 가 어떻게 처리하는가
ChatGPT의 말:
Knative에서 모델을 서빙하는 각 Pod에는 queue-proxy라는 사이드카 컨테이너가 추가되며, 이 컨테이너는 트래픽을 조절하고 관리하는 중요한 역할을 합니다.

과도한 트래픽(과부하) 상황에서 queue-proxy의 동작
동적 Autoscaling (Knative Pod Autoscaler, KPA)

Knative는 트래픽이 급증할 경우 자동 스케일링(Autoscaling) 을 수행합니다.
queue-proxy는 트래픽을 관찰하고, activator 및 KPA(Knative Pod Autoscaler)와 협력하여 새로운 Pod을 생성하도록 Kubernetes에 요청합니다.
결과적으로, 부하가 증가하면 Pod 개수가 늘어나면서 트래픽을 분산 처리합니다.
MaxConcurrency 설정을 통한 요청 제한

queue-proxy는 Pod 내에서 처리할 수 있는 최대 동시 요청 개수를 제한합니다.
containerConcurrency 설정 값이 설정되어 있으면, 이 값 이상으로 요청이 들어올 경우 추가 요청을 대기(큐잉)하거나 503 오류를 반환할 수 있습니다.
기본적으로 무제한(0)이지만, 적절한 값을 설정하면 과부하를 방지할 수 있습니다.
Backpressure (큐잉과 서킷 브레이킹)

queue-proxy는 처리할 수 없는 요청이 들어오면 내부적으로 큐에 적재합니다.
이 큐의 크기가 한계에 도달하면, 더 이상 요청을 받지 않고 503 Service Unavailable 오류를 반환합니다.
Knative Serving 설정에서 queue-proxy의 buffer capacity를 조정할 수 있습니다.
Rate Limiting (속도 제한)

Knative는 Activator를 활용하여 트래픽을 조절하는데, queue-proxy가 과부하 상태에 도달하면 Activator가 트래픽을 제어하여 더 많은 Pod이 준비될 때까지 요청을 지연시키거나 거부할 수 있습니다.
특정한 Rate Limit 설정이 없더라도, Pod의 리소스 한계를 초과하지 않도록 자동으로 트래픽을 분배하는 메커니즘이 내장되어 있습니다.
오버로드 시 Graceful Degradation

Pod이 트래픽을 감당하지 못하고 CPU/메모리 리소스가 부족해질 경우, queue-proxy는 일부 요청을 거부하고 기존 요청을 보호하려는 전략을 취합니다.
Knative Serving은 기본적으로 Kubernetes의 리소스 제한(CPU/Mem Limit) 설정을 따르므로, 한계를 초과하면 OOM(Out of Memory) 발생 시 Pod이 재시작될 수 있습니다.
결론
Knative의 queue-proxy는 과도한 트래픽이 인가될 경우 (1) 자동 스케일링을 유도하고, (2) 동시 요청 제한을 적용하며, (3) 트래픽 큐잉 및 제한을 수행하여 서비스 안정성을 유지하려고 합니다. 만약 설정된 한계를 초과하면 503 응답 반환, 트래픽 거부, 또는 Pod 재시작 등의 대응이 이루어질 수 있습니다.

더 효율적인 트래픽 관리를 위해서는 containerConcurrency, autoscaler 설정, 그리고 Kubernetes의 HPA(Horizontal Pod Autoscaler)와 연동하여 최적의 설정을 조정하는 것이 중요합니다. 🚀

나의 말:
Autoscaling 을 사용하지 않도록 설정하고 containerConcurrency 값도 0 으로 기본 설정하면 queue-proxy 는 어떻게 동작하나
ChatGPT의 말:
Knative에서 Autoscaling을 비활성화하고, containerConcurrency 값을 0(기본값)으로 설정하면, queue-proxy의 동작 방식이 달라집니다.

1. Autoscaling이 비활성화되면 어떤 일이 발생하는가?
기본적으로 Knative는 자동 스케일링(Autoscaling, KPA/HPA)을 통해 Pod을 추가 및 제거하면서 트래픽을 분산시킵니다.
하지만, Autoscaling을 사용하지 않도록 설정하면 Pod 개수가 고정되며, 과부하가 걸려도 새로운 Pod이 생성되지 않습니다.
이 경우, 모든 요청은 현재 실행 중인 Pod에서만 처리됩니다.
2. containerConcurrency = 0 설정 시 동작
containerConcurrency = 0은 무제한 동시 요청을 허용하는 설정입니다.
즉, queue-proxy가 들어오는 요청을 제한하지 않으며, Pod의 애플리케이션 컨테이너로 가능한 모든 요청을 전달합니다.
이로 인해 과도한 요청이 유입되면 애플리케이션이 과부하 상태에 빠질 가능성이 큽니다.
3. 과부하(Overload) 상황에서 queue-proxy의 반응
① 리소스 부족 (CPU/메모리 한계 도달)
queue-proxy는 트래픽을 제한하지 않으므로, Pod 내부의 애플리케이션이 감당할 수 없는 수준의 요청이 쌓이면 CPU 및 메모리 사용량이 폭증할 수 있습니다.
Kubernetes에서 리소스 제한이 설정된 경우, 컨테이너가 OOMKilled(Out of Memory Killed) 되어 강제 재시작될 수 있습니다.
② 응답 속도 지연 및 타임아웃 발생
무제한으로 요청을 받으면 애플리케이션 컨테이너가 모든 요청을 처리하지 못해 응답 시간이 길어지거나 요청이 큐에 계속 쌓이는 현상이 발생할 수 있습니다.
클라이언트 측에서 응답 지연(timeouts) 이 발생할 수 있습니다.
③ 502/503 오류 반환 가능
Knative의 queue-proxy는 자체적으로 버퍼(큐) 크기 제한이 있음.
이 제한을 초과하면 더 이상 요청을 수용할 수 없고, 503(Service Unavailable) 오류를 반환할 수 있음.
만약 애플리케이션이 일정 시간 내 응답하지 못하면, Kubernetes의 livenessProbe/readinessProbe 체크에 의해 Pod이 비정상 상태로 판단되어 재시작될 수도 있음.
4. 정리: Autoscaling 없이 containerConcurrency=0 설정 시 결과
설정	결과
Autoscaling 비활성화	Pod 개수가 고정됨 (트래픽이 증가해도 새 Pod이 생성되지 않음)
containerConcurrency = 0	queue-proxy가 요청을 무제한 전달함 (과부하 가능성↑)
요청 과부하 발생 시	응답 지연, OOMKilled 발생 가능, 503 오류 반환 가능
극단적인 경우	애플리케이션이 다운되거나 Pod이 CrashLoopBackOff 상태가 될 수 있음
5. 해결책
Autoscaling을 사용하지 않는 경우, 적절한 containerConcurrency 값을 설정하는 것이 중요합니다.
예를 들어, containerConcurrency = 10으로 설정하면, 하나의 Pod이 동시에 최대 10개의 요청만 처리하도록 제한할 수 있음.
CPU/메모리 리소스 요청(Request) 및 제한(Limit)을 적절히 설정하여 과부하를 방지해야 함.
결론
Autoscaling 없이 containerConcurrency = 0으로 설정하면, queue-proxy는 요청을 무제한 전달하므로 애플리케이션 과부하, 성능 저하, 응답 실패(503) 등의 문제가 발생할 가능성이 높음.
따라서 적절한 containerConcurrency 값과 리소스 제한을 설정하여 서비스 안정성을 유지하는 것이 중요함. 🚀
나의 말:
위의 상황에서 app container 가 아니라 queue-proxy container 에서 OOM 이 발생할수 있나
ChatGPT의 말:
queue-proxy 컨테이너에서 OOM이 발생할 수 있는가?
네, queue-proxy 컨테이너에서 OOM(Out of Memory) 오류가 발생할 가능성이 있습니다.
특히 Autoscaling이 비활성화되고, containerConcurrency=0(무제한 동시 요청 허용) 인 경우 다음과 같은 이유로 OOM이 발생할 수 있습니다.

1. queue-proxy에서 OOM이 발생하는 원인
① 요청이 과도하게 밀려서 큐가 넘칠 경우
queue-proxy는 들어오는 요청을 내부 큐에 보관한 후, 순차적으로 애플리케이션 컨테이너에 전달합니다.
containerConcurrency=0 설정 시 queue-proxy가 제한 없이 요청을 받아들이므로, Pod 내부의 애플리케이션이 요청을 처리하지 못하고 쌓이면 queue-proxy의 큐 메모리 사용량이 급증할 수 있습니다.
결국, queue-proxy 컨테이너가 OOMKilled 될 수 있습니다.
② 장시간 처리되지 않는 요청(슬로우 요청)
queue-proxy는 애플리케이션 컨테이너에 요청을 전달하고 응답을 기다립니다.
만약 애플리케이션이 응답을 늦게 반환하거나 멈추면, queue-proxy 내부에서 요청을 계속 보관하고 있어야 하므로 메모리 사용량이 증가합니다.
많은 요청이 동시에 오고, 애플리케이션이 지연되면 queue-proxy의 메모리 사용량이 누적되어 결국 OOM이 발생할 수 있습니다.
③ Keep-Alive로 인한 메모리 누수
Knative의 queue-proxy는 HTTP Keep-Alive를 지원하여 클라이언트와의 연결을 유지할 수 있습니다.
요청이 많아지고 오래된 연결이 적절히 닫히지 않으면, queue-proxy의 메모리 사용량이 지속적으로 증가하여 OOM이 발생할 수 있습니다.
2. queue-proxy OOM 발생 시 나타나는 현상
queue-proxy 컨테이너가 종료(OOMKilled)

kubectl get pod -n <namespace> 실행 시 queue-proxy 컨테이너가 OOMKilled됨을 확인할 수 있음.
bash
복사
편집
NAME                            READY   STATUS      RESTARTS   AGE
my-app-deployment-1234          0/2     OOMKilled   3          5m
kubectl describe pod my-app-deployment-1234 -n <namespace> 실행 시, 다음과 같은 OOM 로그를 확인할 수 있음.
yaml
복사
편집
Last State: 
  Terminated:
    Reason:       OOMKilled
    Exit Code:    137
503 Service Unavailable 오류

queue-proxy가 OOMKilled되면 트래픽을 처리하지 못해 503 오류가 발생할 수 있음.
클라이언트가 API 호출 시 다음과 같은 응답을 받을 수 있음.
json
복사
편집
{
  "error": "503 Service Unavailable",
  "message": "queue-proxy failed to process request"
}
Knative Revision의 Pod이 CrashLoopBackOff 상태에 빠짐

queue-proxy 컨테이너가 지속적으로 OOMKilled되면, Pod이 CrashLoopBackOff 상태가 될 수 있음.
bash
복사
편집
kubectl get pods -n <namespace>
perl
복사
편집
NAME                              READY   STATUS             RESTARTS   AGE
my-app-deployment-5678            0/2     CrashLoopBackOff   5          10m
3. 해결책
① queue-proxy의 메모리 리소스 제한 설정
Knative에서 queue-proxy의 메모리 사용량을 제한할 수 있습니다.
아래와 같이 Kubernetes 리소스 제한을 설정하면, 과도한 메모리 사용을 방지할 수 있음.

yaml
복사
편집
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        # queue-proxy의 메모리 요청 및 제한 설정
        autoscaling.knative.dev/target: "10"  # 적절한 동시 요청 수 제한 (0이 아니라 적절한 값 사용)
    spec:
      containers:
        - name: queue-proxy
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "256Mi"
              cpu: "200m"
② containerConcurrency 값 조정
containerConcurrency=0(무제한) 대신 적절한 값을 설정하여 queue-proxy가 과부하되지 않도록 조절 가능.
예를 들어, containerConcurrency=10으로 설정하면 queue-proxy가 동시 요청을 10개까지만 받아들이고 나머지는 큐잉 또는 503 오류 반환.
yaml
복사
편집
autoscaling.knative.dev/containerConcurrency: "10"
③ 요청 타임아웃 설정
queue-proxy의 기본 타임아웃은 600초(10분)인데, 너무 긴 경우 queue-proxy가 메모리를 많이 사용하게 됨.
타임아웃을 30초로 줄이면, 오래된 요청을 자동으로 정리 가능.
yaml
복사
편집
autoscaling.knative.dev/target-burst-capacity: "0"
queue.sidecar.serving.knative.dev/request-timeout-seconds: "30"
④ Liveness Probe 및 Readiness Probe 설정
queue-proxy가 이상 상태에 빠졌을 때 자동으로 재시작되도록 Kubernetes Liveness/Readiness Probe 설정을 조정.
yaml
복사
편집
livenessProbe:
  httpGet:
    path: /healthz
    port: 8022
  initialDelaySeconds: 3
  periodSeconds: 10
  failureThreshold: 3
4. 결론
queue-proxy 컨테이너에서도 OOM이 발생할 수 있음.
특히 Autoscaling이 비활성화되고 containerConcurrency=0(무제한)이면 요청이 queue-proxy 내부에서 과도하게 쌓이면서 메모리 사용량이 폭증할 수 있음.
해결을 위해서는:
queue-proxy 메모리 리소스 제한 설정
containerConcurrency 값을 적절한 값(예: 10)으로 조정
요청 타임아웃 및 Keep-Alive 설정을 최적화
Liveness/Readiness Probe 설정으로 문제 발생 시 자동 복구
이렇게 설정하면 queue-proxy의 OOM 문제를 방지하면서 안정적으로 트래픽을 처리할 수 있습니다. 🚀