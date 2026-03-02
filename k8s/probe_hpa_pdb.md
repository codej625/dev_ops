# Next.js 쿠버네티스 배포 설정

<br />
<br />

* Next.js 앱을 K3s 클러스터에 안정적으로 배포하기 위한 설정

---

```
Pod가 떴다고 바로 트래픽을 받을 수 있는 건 아니다.

내부적으로 죽어있어도 Running으로 보일 수 있다.

트래픽이 몰릴 때 Pod 수를 자동으로 조절해야 한다.

이 세 가지 문제를 해결하는 게 Probe, PDB, HPA다.
```

<br />
<br />
<br />
<br />

1. Probe (프로브)

<br />


```
Probe는 쿠버네티스가 Pod한테 주기적으로 상태를 체크하는 헬스체크이다.

Probe는 두 종류가 있고 역할이 완전히 다르다.
+------------------+------------------------------------------+
| Readiness Probe  | 트래픽 받을 준비 됐어?                        |
+------------------+------------------------------------------+
| Liveness Probe   | 지금도 정상 동작 중이야?                       |
+------------------+------------------------------------------+
```

<br />

`1) Readiness Probe — 트래픽 받을 준비 됐어?`

```
준비 안 된 Pod는 Service 엔드포인트 목록에서 제외된다.

트래픽 자체가 가지 않으므로 유저는 502를 볼 수 없다.


* Readiness Probe 없으면
유저 요청 -> Service -> 준비 안 된 Pod -> 502 -> 유저가 502 봄

* Readiness Probe 있으면
준비 안 된 Pod는 Service 목록에서 제외
유저 요청 -> Service -> 준비된 Pod에만 전달 -> 정상 응답
```

```
replicas 2개 중 1개가 배포 중일 때


* Probe 없음 -> Service가 2개 Pod 모두에 트래픽 분산
               -> 배포 중인 Pod에 맞은 요청은 502

* Probe 있음 -> Service가 준비된 1개 Pod에만 트래픽
               -> 502 없음 (무중단 배포)
```

<br />
<br />

`2) Liveness Probe — 지금도 정상 동작 중이야?`

```
Pod가 Running 상태여도 내부적으로 죽어있는 경우가 있다.

-> 메모리 누수로 응답 불가 상태
-> 데드락 걸려서 스레드 전부 멈춤

kubectl get pods 하면 Running인데 실제론 502만 반환하는 상태.

Liveness Probe가 실패를 감지하면 쿠버네티스가 자동으로 Pod를 재시작한다.
```

<br />
<br />
<br />

2. PDB (PodDisruptionBudget)

<br />

`노드 점검 / 배포 시 최소 몇 개의 Pod는 항상 살아있어야 하는지 강제하는 규칙`

```
replicas를 2로 설정해도 PDB가 없으면,
쿠버네티스가 노드 점검 시 2개를 동시에 내려버릴 수 있다.


* PDB 없을 때
노드 점검 시작 -> Pod 1 종료 -> Pod 2 종료 (동시) -> 서비스 중단

* PDB 있을 때 (minAvailable: 1)
  노드 점검 시작
    -> Pod 1 종료
    -> 새 Pod Readiness 통과 확인
    -> Pod 2 종료
    -> 서비스 무중단
```

<br />
<br />
<br />

3. HPA (HorizontalPodAutoscaler)

<br />

`CPU / 메모리 사용량을 보고 Pod 수를 자동으로 늘리거나 줄이는 기능`

```
평상시 -> replicas 2개로 충분  
피크타임 -> 갑자기 트래픽 급증


위의 상황에서 적용 시

* HPA 없음 -> Pod 2개가 버티다 응답 지연 or 503

* HPA 있음 -> CPU 70% 넘으면 자동으로 Pod 추가 -> 최대 4개까지 확장
             트래픽 줄면 -> 자동으로 Pod 수 다시 감소
```

```
+--------------------------------------------+
|   HPA 동작 구조                              |
|                                            |
|   1. metrics-server                        |
|   (CPU/메모리 수집)                           |
|                                            |
|   2. HPA가 사용률 확인                        | 
|                                            |
|   3. 70% 초과 → Deployment replicas 자동 증가 |
|      70% 이하 → Deployment replicas 자동 감소 |
|                                            |
+--------------------------------------------+

* K3s는 metrics-server가 기본 포함되어 있어 별도 설치 불필요
```

<br />
<br />
<br />

4. 전체 관계

```
* 트래픽 급증
    -> HPA가 Pod 자동 추가
    -> Readiness Probe 통과 후 트래픽 수신 시작 (502 방지)

* 노드 점검 / 배포
    -> PDB가 최소 1개 보장
    -> Readiness Probe 통과한 Pod에만 트래픽 (무중단)

* Pod 내부 장애
    -> Liveness Probe 실패 감지
    -> 자동 재시작 (수동 개입 불필요)
```
<br />
<br />
<br />

5. 적용 예시

<br />

`app/api/health/route.ts`

```ts
// 헬스체크 API Route 생성 (Probe 엔드포인트용)

export async function GET() {
  return Response.json({ status: 'ok' })
}
```

```
/ 대신 /api/health를 쓰는 이유

/ 로 체크하면
- 메인 페이지 전체 렌더링이 매번 발생
- DB 조회, 외부 API 호출 등 불필요한 부하
- 5초마다 실제 페이지를 그리는 셈

/api/health 로 체크하면
- { status: 'ok' } 만 반환
- 서버가 살아있는지만 확인
- 부하 거의 없음
```

<br />

`next-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: next
  namespace: default
spec:
  replicas: 2 # HPA minReplicas와 동일하게
  selector:
    matchLabels:
      app: next
  template:
    metadata:
      labels:
        app: next
    spec:
      containers:
      - name: next
        image: <your-next-image>:<tag>
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1500m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 5 # Next는 빠르게 뜨니까 5초면 충분
          periodSeconds: 5 # 5초마다 체크
          failureThreshold: 3 # 3번 연속 실패해야 "준비 안 됨" 판정
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10 # 10초마다 체크
          failureThreshold: 3 # 3번 연속 실패하면 Pod 재시작
```

<br />

`next-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: next-svc
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: next # Deployment label과 반드시 일치
  ports:
  - name: http
    port: 3000
    targetPort: 3000
    protocol: TCP
```

<br />

`next-pdb.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: next-pdb
  namespace: default
spec:
  minAvailable: 1 # 최소 1개는 항상 살아있어야 함
  selector:
    matchLabels:
      app: next # Deployment label과 반드시 일치
```

<br />

`next-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: next-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: next # Deployment label과 반드시 일치
  minReplicas: 2
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

<br />

`적용`

```zsh
kubectl apply -f next-deployment.yaml
kubectl apply -f next-svc.yaml
kubectl apply -f next-pdb.yaml
kubectl apply -f next-hpa.yaml
```

```zsh
# Probe 상태 확인
kubectl describe pod <pod-name> | grep -A 10 Readiness
kubectl describe pod <pod-name> | grep -A 10 Liveness

# PDB 확인
kubectl get pdb
# NAME       MIN AVAILABLE   ALLOWED DISRUPTIONS
# next-pdb   1               1   ← replicas 2개 중 1개까지만 동시에 내릴 수 있음

# HPA 확인
kubectl get hpa
# NAME       REFERENCE         TARGETS   MINPODS   MAXPODS   REPLICAS
# next-hpa   Deployment/next   15%/70%   2         4         2
```
