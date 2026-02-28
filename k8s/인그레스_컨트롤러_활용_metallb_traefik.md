# 인그레스 컨트롤러 활용하기

<br />
<br />

* 쿠버네티스 환경에서 Ingress Controller를 사용한 외부 트래픽 처리

---

```
Ingress 리소스의 주된 목적은 Ingress Controller를 통해, 
외부 트래픽을 클러스터 내부의 특정 Service 리소스와 연결하는 것이다.

VM을 사용해서 K3s 클러스터 구조를 만드는 예시이다.
(MetalLB + Traefik 사용, 하나의 클러스터에 여러 개의 인스턴스 사용)
```

<br />
<br />
<br />
<br />

1. 생성할 VM 사양 (예시)

| VM  | 역할      | CPU    | RAM | DISK  | VM IP 예시       |
|:---:|----------|--------|-----|-------|-----------------|
| VM1 | Master 1 | 2 Core | 2GB | 20GB  | 192.168.0.101   |
| VM2 | Worker 1 | 2 Core | 4GB | 50GB+ | 192.168.0.102   |
| VM3 | Worker 2 | 2 Core | 4GB | 50GB+ | 192.168.0.103   |

`* MetalLB LoadBalancer IP 범위 "192.168.0.240 ~ 192.168.0.250" 이라고 가정.`

<br />
<br />
<br />

2. 각 VM 사전 설정

`0) NTP 시간 동기화 (각 VM에서)`

```zsh
# ⚠️ 쿠버네티스는 노드 간 시간이 크게 어긋나면 TLS 인증 오류, etcd 이상, 토큰 검증 실패 등
# 원인을 찾기 어려운 문제가 발생함. VM 생성 직후 반드시 확인할 것.

# 시간 동기화 상태 확인
timedatectl status

# NTP 동기화 활성화
sudo timedatectl set-ntp true

# 동기화 완료 확인 (System clock synchronized: yes 가 되어야 함)
timedatectl status

# 노드 간 시간 차이 확인 (각 VM에서 실행 후 비교)
date
```

<br />

`1) 호스트명 설정 (각 VM에서)`

```zsh
# VM1 에서 실행
sudo hostnamectl set-hostname master-1

# VM2 에서 실행
sudo hostnamectl set-hostname worker-1

# VM3 에서 실행
sudo hostnamectl set-hostname worker-2

# ⚠️ 호스트명 변경은 현재 세션에 즉시 반영되지 않음
# 변경 후 SSH 재접속 또는 아래 명령으로 새 쉘 실행
exec bash
```

<br />

`2) /etc/hosts에 등록 (각 VM에서, IP는 예시이다.)`

```zsh
sudo tee -a /etc/hosts << EOF
192.168.0.101 master-1
192.168.0.102 worker-1
192.168.0.103 worker-2
EOF
```

<br />

`3) 방화벽 (테스트 시 포트만 오픈)`

```zsh
# ⚠️ Pod 간 통신 (CNI - Flannel VXLAN) 주의사항
# K3s는 기본 CNI로 Flannel을 사용하며, 노드 간 Pod 트래픽은 VXLAN 터널링으로 전달됨.
# Pod 네트워크는 VM 네트워크와 별개의 오버레이 네트워크이므로,
# VM끼리 ping이 되더라도 UDP 8472 포트가 막혀 있으면 Pod 간 통신이 안 됨.
# → 반드시 모든 노드에서 UDP 8472를 오픈해야 함.
#
# 증상: VM ping은 되는데 Pod 간 통신 실패, 다른 노드의 Pod 호출 시 timeout
# 원인: Flannel VXLAN이 UDP 8472로 노드 간 Pod 트래픽을 터널링하는데 방화벽이 차단
```

```zsh
# Master 노드 (VM1)
sudo ufw allow 6443/tcp        # K3s API Server
sudo ufw allow 10250/tcp       # kubelet
sudo ufw allow 2379:2380/tcp   # etcd
sudo ufw allow 80/tcp          # HTTP
sudo ufw allow 443/tcp         # HTTPS
sudo ufw allow 8472/udp        # Flannel VXLAN (노드 간 Pod 통신 - 반드시 필요)
sudo ufw allow 51820/udp       # Flannel WireGuard (암호화 모드 사용 시)
sudo ufw enable
```

```zsh
# Worker 노드 (VM2, VM3)
sudo ufw allow 10250/tcp       # kubelet
sudo ufw allow 30000:32767/tcp # NodePort
sudo ufw allow 80/tcp          # HTTP
sudo ufw allow 443/tcp         # HTTPS
sudo ufw allow 8472/udp        # Flannel VXLAN (노드 간 Pod 통신 - 반드시 필요)
sudo ufw allow 51820/udp       # Flannel WireGuard (암호화 모드 사용 시)
sudo ufw enable
```

```zsh
# 방화벽 적용 후 노드 간 Pod 통신 확인 방법
# 서로 다른 노드에 Pod를 하나씩 띄우고 통신 테스트
# ⚠️ busybox:latest는 pull 실패가 간혹 발생하므로 버전 고정 사용
kubectl run test-1 --image=busybox:1.36 --overrides='{"spec":{"nodeName":"worker-1"}}' -- sleep 3600
kubectl run test-2 --image=busybox:1.36 --overrides='{"spec":{"nodeName":"worker-2"}}' -- sleep 3600

# Pod가 Running 상태가 될 때까지 대기
kubectl get pod test-1 test-2 -w

# test-2의 Pod IP 확인
kubectl get pod test-2 -o wide

# test-1에서 test-2로 ping 테스트
kubectl exec test-1 -- ping <test-2-pod-ip> -c 3

# 테스트 후 정리
kubectl delete pod test-1 test-2
```

<br />
<br />
<br />

3. VM1에서 Master 1 설치

```zsh
# Traefik 유지, TLS SAN 추가 (IP는 VM IP에 맞게 변경 필요)
# MetalLB를 사용하므로 servicelb 비활성화, traefik은 유지
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=servicelb --tls-san=192.168.0.101" sh -

sudo systemctl status k3s
kubectl get nodes

# 워커 조인용 토큰 확인
sudo cat /var/lib/rancher/k3s/server/node-token
```

```zsh
# SAN 이란?
SAN = Subject Alternative Name

TLS 인증서에 이 인증서로 접속해도 되는 주소 목록이 들어 있음.

그 목록에 IP나 도메인을 넣어 두어야, 
그 주소로 접속했을 때 브라우저/클라이언트가 인증서를 유효하다 인정함.
```

<br />

```zsh
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# ⚠️ kubeconfig 파일 권한을 반드시 600으로 설정
# 권한이 너무 열려 있으면 kubectl이 경고를 내거나 동작을 거부함
chmod 600 ~/.kube/config

# ⚠️ k3s.yaml의 server 주소는 기본적으로 127.0.0.1로 되어 있음
# 마스터 노드 내부에서는 동작하지만, 외부 PC에서 kubectl로 원격 접근 시 이 파일을 그대로 쓰면 접속 불가
# 외부 접근이 필요한 경우 server 주소를 실제 마스터 IP로 교체
sed -i "s/127.0.0.1/192.168.0.101/g" ~/.kube/config

# 정상 동작 확인
kubectl get nodes
```

```zsh
# taint 설정 (toleration이 없는 Pod(대부분의 앱) 는 마스터에 안 올라가고 워커에만 올라감)
kubectl taint nodes master-1 node-role.kubernetes.io/control-plane:NoSchedule --overwrite
```

<br />
<br />
<br />

4. VM2, VM3에서 K3s Worker 조인

`VM2, VM3에서 실행`

```zsh
# ⚠️ export 방식은 SSH 세션이 끊기면 환경변수가 휘발되어 설치 실패 가능
# 한 줄로 인라인 전달하는 방식이 더 안전함
curl -sfL https://get.k3s.io | K3S_URL="https://192.168.0.101:6443" K3S_TOKEN="<토큰값>" sh -
sudo systemctl status k3s-agent
```

<br />

`VM1에서 조인 확인`

```zsh
# 노드 확인
kubectl get nodes
```

| NAME     | STATUS | ROLES                | AGE | VERSION  |
|----------|--------|----------------------|-----|----------|
| master-1 | Ready  | control-plane,master | 5m  | v1.30.x  |
| worker-1 | Ready  | <none>               | 2m  | v1.30.x  |
| worker-2 | Ready  | <none>               | 1m  | v1.30.x  |

<br />
<br />
<br />

5. MetalLB 설치 (LoadBalancer 구현)

```zsh
# ⚠️ MetalLB IP 범위는 공유기 DHCP 자동 할당 범위 밖으로 설정할 것
# 공유기 관리자 페이지에서 DHCP 범위를 확인하고, 해당 범위와 겹치지 않는 IP 대역을 사용해야 함
# DHCP 범위가 192.168.0.100~200 이라면 192.168.0.240~250 은 안전하지만 환경마다 다름

# ⚠️ 버전은 최신 stable로 변경: https://github.com/metallb/metallb/releases
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml

# Pod들이 Ready 상태가 될 때까지 대기
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# ⚠️ metallb-system에는 controller(1개)와 speaker(노드 수만큼 DaemonSet) 두 종류가 뜸
# speaker가 모두 Running이어야 L2 광고가 동작하고 IP가 실제로 할당됨
# wait는 일부만 통과해도 완료되므로, 반드시 아래로 전체 Pod 상태를 직접 확인
kubectl get pods -n metallb-system
# 출력 예시:
# controller-xxx   Running  ← 1개
# speaker-xxx      Running  ← 노드 수만큼 (master-1, worker-1, worker-2 각 노드당 1개)
```

<br />

`metallb-config.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  # ⚠️ 공유기 DHCP 범위와 겹치지 않는 IP 대역으로 설정할 것
  - 192.168.0.240-192.168.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

```zsh
kubectl apply -f metallb-config.yaml

# MetalLB 상태 확인
kubectl get ipaddresspool -n metallb-system
```

<br />
<br />
<br />

6. Ingress Controller 설정 (K3s 기본 Traefik)

```zsh
# ⚠️ patch 전에 Traefik Pod가 정상 Running 상태인지 먼저 확인
# Traefik이 CrashLoopBackOff 상태면 patch를 해도 EXTERNAL-IP가 할당되지 않음
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
# STATUS가 Running이어야 다음 단계 진행

# Traefik 서비스를 LoadBalancer 타입으로 변경
# MetalLB가 IPAddressPool 범위 내 IP를 자동으로 할당함
kubectl patch svc traefik -n kube-system \
  --type=merge \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

```zsh
# ⚠️ patch 후 MetalLB가 IP를 할당할 때까지 수 초 걸릴 수 있음
# EXTERNAL-IP가 바로 안 나타나면 아래 명령으로 할당될 때까지 대기
kubectl get svc -n kube-system traefik -w
# EXTERNAL-IP 컬럼에 192.168.0.240~250 범위의 IP가 표시되면 정상 (Ctrl+C로 종료)
kubectl get svc -n kube-system traefik

# ⚠️ EXTERNAL-IP가 계속 <pending> 이면 MetalLB 설정 또는 IP 대역 충돌 확인
# kubectl get pods -n metallb-system 으로 speaker 전체가 Running인지 확인
```

<br />
<br />
<br />

7. cert-manager 설치

```zsh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml

# CRD 등록까지 완전히 완료될 때까지 대기 (wait 직후 apply 시 CRD 미등록 오류 발생 가능)
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=90s

# ⚠️ wait 완료 후에도 CRD API 등록까지 수 초 더 걸릴 수 있음
# "no matches for kind ClusterIssuer" 오류 발생 시 아래 sleep 후 재시도
sleep 10
```

<br />

`letsencrypt-issuer.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    # 본인 이메일로 반드시 수정. Let's Encrypt가 인증서 만료·계정 관련 연락에 사용
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```

```zsh
kubectl apply -f letsencrypt-issuer.yaml
kubectl get clusterissuer
```

```zsh
# ⚠️ Let's Encrypt 발급 실패 시 주의사항
# 같은 도메인에 대해 1시간 내 5회 이상 실패하면 일시적으로 차단됨 (재시도 쿨다운 발생)
# 반드시 아래 명령으로 실패 원인을 먼저 확인하고 수정 후 재시도할 것

kubectl describe certificate -A
kubectl describe certificaterequest -A
kubectl logs -n cert-manager -l app=cert-manager --tail=100

# ⚠️ HTTP01 챌린지 전제 조건
# cert-manager가 HTTP01 챌린지를 수행하려면 Let's Encrypt 서버가 외부에서 도메인의 80포트로 접근 가능해야 함.
# 공유기 포트포워딩(12번 항목)이 먼저 설정되어 있어야 인증서 발급이 성공함.
# 일부 ISP (KT, SKB 등 일부 요금제)는 80/443 포트 자체를 차단하는 경우가 있음.
# → 해당 환경에서는 DNS01 챌린지 방식으로 전환 필요: https://cert-manager.io/docs/configuration/acme/dns01/
```

<br />
<br />
<br />

8. App Deployment 및 Service 설정

```zsh
# ⚠️ Service는 selector로 Pod를 찾습니다.
# spring-svc → app: spring 라벨이 있는 Pod
# next-svc   → app: next 라벨이 있는 Pod
# Deployment가 먼저 배포되어 있어야 Endpoint가 연결됩니다.
# Endpoint 확인: kubectl get endpoints spring-svc
# Endpoint가 <none> 이면 Deployment가 없거나 라벨이 일치하지 않는 것
```

<br />

`spring-deployment.yaml (예시)`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring  # spring-svc selector와 반드시 일치
  template:
    metadata:
      labels:
        app: spring
    spec:
      containers:
      - name: spring
        image: <your-spring-image>:<tag>  # 실제 이미지로 변경
        imagePullPolicy: Always  # 항상 최신 이미지 pull (로컬/프라이빗 레지스트리 사용 시 특히 중요)
        ports:
        - containerPort: 8080
```

<br />

`next-deployment.yaml (예시)`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: next
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: next  # next-svc selector와 반드시 일치
  template:
    metadata:
      labels:
        app: next
    spec:
      containers:
      - name: next
        image: <your-next-image>:<tag>  # 실제 이미지로 변경
        imagePullPolicy: Always  # 항상 최신 이미지 pull (로컬/프라이빗 레지스트리 사용 시 특히 중요)
        ports:
        - containerPort: 3000
```

<br />

`spring-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-svc
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: spring
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
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
    app: next
  ports:
  - name: http
    port: 3000
    targetPort: 3000
    protocol: TCP
```

```zsh
kubectl apply -f spring-deployment.yaml
kubectl apply -f spring-svc.yaml
kubectl get svc spring-svc
kubectl get endpoints spring-svc  # Pod와 연결 확인

kubectl apply -f next-deployment.yaml
kubectl apply -f next-svc.yaml
kubectl get svc next-svc
kubectl get endpoints next-svc  # Pod와 연결 확인
```

<br />
<br />
<br />

9. IngressClass (Traefik, 필요 시)

`ingressclass-traefik.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: traefik
spec:
  controller: traefik.io/ingress-controller
```

<br />

```zsh
# K3s 설치 시 traefik IngressClass가 자동 생성됨
# 아래로 먼저 확인하고, 목록에 traefik 이 없을 때만 apply 실행
kubectl get ingressclass
# NAME      CONTROLLER                      PARAMETERS   AGE
# traefik   traefik.io/ingress-controller   <none>       5m  ← 이 항목이 있으면 생략 가능

# traefik 항목이 없을 때만 실행
kubectl apply -f ingressclass-traefik.yaml
```

<br />
<br />
<br />

10. Ingress 라우팅 설정

```zsh
# ⚠️ Ingress apply 전 ClusterIssuer가 Ready 상태인지 반드시 확인
# READY: True 가 아닌 상태에서 apply 하면 인증서 발급이 시작되지 않음
kubectl get clusterissuer letsencrypt-prod
```

<br />

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    # A 레코드를 두 개 (또는 서브도메인 두 개)
    # 두 도메인을 하나의 secretName으로 묶으면 SAN 2개짜리 인증서가 발급됨
    - app.example.com
    - api.example.com
    secretName: example-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: next-svc
            port:
              number: 3000
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spring-svc
            port:
              number: 8080
```

<br />

`Ingress – 경로로 나누기 (같은 도메인)`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: spring-svc
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: next-svc
            port:
              number: 3000

# Traefik은 경로 길이 기준으로 자동 우선순위를 매기므로 순서에 덜 민감함.
# 다만 가독성을 위해 구체적인 경로(/api)를 먼저, 범용 경로(/)를 나중에 작성하는 것을 권장.
```

```zsh
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl get svc

# 인증서 발급 진행 확인 (Ingress apply 후 자동 시작, 최대 2-3분 소요)
# READY: False → 발급 진행 중 / READY: True → 발급 완료
kubectl get certificate -A -w
# Ctrl+C 로 종료

# 발급이 오래 걸리거나 실패 시 상세 확인
kubectl describe certificate example-tls -n default
```

<br />
<br />
<br />

11. 도메인 DNS 설정

```
도메인 구입처 DNS에서 A 레코드 추가. 
전파는 수분~최대 24시간 소요될 수 있음.
```

<br />

| 구분       | 설명           |
|-----------|---------------|
| 도메인     | healthapp.shop |
| 레코드     | A              |
| 호스트     | @              |
| 값(IP주소) | 공인IP          |
| TTL      | 3600 (1시간)    |

```zsh
nslookup healthapp.shop
dig healthapp.shop
```

<br />
<br />
<br />

12. 공유기 포트포워딩 설정

```
⚠️ cert-manager의 HTTP01 챌린지는 외부 → 80포트 → VIP → Traefik 흐름으로 동작함.
   포트포워딩이 설정되지 않으면 인증서 발급이 실패하므로 반드시 먼저 설정할 것.

⚠️ ISP(KT, SKB 등 일부 요금제)에서 80/443 포트를 차단하는 경우가 있음.
   이 경우 HTTP01 챌린지가 동작하지 않으므로 cert-manager DNS01 챌린지 방식으로 전환 필요.
   참고: https://cert-manager.io/docs/configuration/acme/dns01/
```

| 외부 포트    | 내부 IP        | 내부 포트   | 프로토콜   | 설명   |
|-----------|---------------|-----------|----------|-------|
| 80        | 192.168.0.240 | 80        | TCP      | HTTP  |
| 443       | 192.168.0.240 | 443       | TCP      | HTTPS |

<br />
<br />
<br />

13. 인증서 자동 갱신

```
cert-manager는 인증서 만료 30일 전에 자동으로 갱신을 시도함.
Let's Encrypt 인증서 유효기간은 90일이므로, 실제로는 60일째에 갱신이 시작됨.
별도 설정 없이 cert-manager가 설치되어 있으면 자동으로 동작함.
```

<br />

```zsh
# 인증서 현재 상태 및 만료일 확인
kubectl get certificate -A

# 상세 확인 (갱신 시도 여부, 마지막 갱신 시간 등)
kubectl describe certificate <certificate-name> -n default
```

```zsh
# 정상 상태 예시
# READY: True  → 인증서가 유효한 상태
# READY: False → 발급 중이거나 갱신 실패 상태

NAME          READY   SECRET        AGE
example-tls   True    example-tls   10d
```

<br />

```zsh
# 갱신이 제대로 동작하는지 확인하는 방법
# cert-manager가 갱신을 시도할 때 CertificateRequest 리소스가 생성됨
kubectl get certificaterequest -A

# 갱신 관련 이벤트 로그 확인
kubectl describe certificate example-tls -n default | grep -A 10 Events
```

<br />

```zsh
# ⚠️ 자동 갱신 실패 조건
# - 포트포워딩이 해제된 경우 (공유기 재시작, IP 변경 등)
# - ISP 공인 IP가 변경되어 DNS A 레코드가 틀어진 경우
# - cert-manager Pod가 죽어 있는 경우
# - Let's Encrypt 서버가 80포트로 접근 불가한 경우

# cert-manager Pod 상태 확인
kubectl get pods -n cert-manager

# 갱신 실패 시 수동으로 갱신 강제 트리거
# (Secret을 삭제하면 cert-manager가 즉시 재발급 시도)
kubectl delete secret example-tls -n default
# Secret 삭제 후 cert-manager가 자동으로 새 인증서 발급 시작
# kubectl get certificate -A -w 로 상태 모니터링
```

<br />

```zsh
# 갱신 주기 커스텀이 필요한 경우 (기본값 변경)
# Certificate 리소스에 renewBefore 필드로 갱신 시작 시점 조정 가능
```

<br />

`certificate-custom.yaml (갱신 시점 커스텀 예시)`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
  namespace: default
spec:
  secretName: example-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - app.example.com
  - api.example.com
  renewBefore: 720h  # 만료 30일 전 갱신 (기본값). 더 일찍 갱신하려면 값을 높임 (예: 1440h = 60일 전)
```

```zsh
# ⚠️ 위 Certificate 리소스는 Ingress에 cert-manager 어노테이션이 있으면 자동 생성됨.
# 갱신 시점만 바꾸고 싶은 경우에만 위 YAML로 수동 관리.
# 일반적으로는 Ingress 어노테이션 방식(10번 항목)으로도 자동 갱신이 정상 동작함.
```

<br />
<br />
<br />

14. 전체 시스템 확인

```zsh
kubectl get all -A
kubectl get ingress -A
kubectl get svc -n kube-system traefik
kubectl get certificate -A
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=100
kubectl logs -n metallb-system -l app=metallb --tail=50
```

<br />
<br />
<br />

15. 트러블슈팅

`인증서 발급 실패 시`

```zsh
# 인증서 상태 확인
kubectl describe certificate -A
kubectl describe certificaterequest -A

# cert-manager 로그 확인
kubectl logs -n cert-manager -l app=cert-manager --tail=100

# ⚠️ Let's Encrypt는 같은 도메인으로 1시간 내 5회 발급 실패 시 일시 차단됨
# 원인을 반드시 먼저 파악하고 수정한 뒤 재시도할 것
```

<br />

`Service Endpoint가 연결 안 될 때 (502 Bad Gateway)`

```zsh
# Endpoint 확인 - <none> 이면 Deployment가 없거나 라벨 불일치
kubectl get endpoints spring-svc
kubectl get endpoints next-svc

# Pod 상태 확인
kubectl get pods -n default
kubectl describe pod <pod-name>
```

<br />

`MetalLB EXTERNAL-IP가 pending 일 때`

```zsh
# MetalLB Pod 상태 확인 (controller + speaker 전체 Running이어야 함)
kubectl get pods -n metallb-system

# IPAddressPool 확인
kubectl get ipaddresspool -n metallb-system

# MetalLB 로그 확인
kubectl logs -n metallb-system -l app=metallb --tail=100

# Traefik EXTERNAL-IP 확인 (Pending 이면 MetalLB 미할당)
kubectl get svc -n kube-system traefik

# ⚠️ IP 범위가 공유기 DHCP 범위와 겹치면 충돌 발생
# 공유기 DHCP 범위를 확인하고 metallb-config.yaml의 주소 범위를 겹치지 않게 수정 후 재적용
kubectl apply -f metallb-config.yaml
```

<br />

`Pod 간 통신이 안 될 때 (다른 노드에 배치된 Pod끼리 통신 실패)`

```zsh
# VM끼리 ping은 되는데 Pod 간 통신이 안 되면 Flannel VXLAN 포트 문제일 가능성이 높음

# 1) 각 노드에서 UDP 8472 방화벽 오픈 여부 확인
sudo ufw status | grep 8472

# 2) 오픈이 안 되어 있으면 모든 노드에서 실행
sudo ufw allow 8472/udp
sudo ufw reload

# 3) Flannel Pod 상태 확인
kubectl get pods -n kube-system -l app=flannel

# 4) 노드 간 Pod 통신 테스트
kubectl run test-1 --image=busybox:1.36 --overrides='{"spec":{"nodeName":"worker-1"}}' -- sleep 3600
kubectl run test-2 --image=busybox:1.36 --overrides='{"spec":{"nodeName":"worker-2"}}' -- sleep 3600
kubectl get pod test-1 test-2 -o wide  # 각 Pod IP 확인
kubectl exec test-1 -- ping <test-2-pod-ip> -c 3
kubectl delete pod test-1 test-2

# ⚠️ Pod IP 대역(기본 10.42.x.x)이 VM 네트워크와 겹치는 경우에도 통신 문제 발생 가능
# K3s 설치 시 --cluster-cidr 옵션으로 대역 변경 가능
```

<br />

`전체 흐름 점검 순서`

```
공인IP 확인 → 포트포워딩 확인 → DNS 전파 확인 (nslookup/dig)
→ MetalLB IP 확인 (kubectl get svc -n kube-system traefik) → 인증서 확인 (kubectl get certificate)
→ Endpoint 확인 (kubectl get endpoints) → 앱 Pod 확인 (kubectl get pods)
→ Pod 간 통신 확인 (kubectl exec ping 테스트)
```

<br />
<br />
<br />

* TIP

```
* 클러스터 내부에서 Service 이름으로 연결

백엔드에서 디비 서버를 붙일때는
서비스 이름으로 지정해놓으면 됨.

ex) PostgreSQL → jdbc:postgresql://database-service:5432/dbname

┌───────────────────────────────────────┐
│            Service 타입 계층            │
│                                       │
│  ┌─────────────────────────────────┐  │
│  │         LoadBalancer            │  │
│  │  ┌───────────────────────────┐  │  │
│  │  │        NodePort           │  │  │
│  │  │  ┌─────────────────────┐  │  │  │
│  │  │  │     ClusterIP       │  │  │  │
│  │  │  │     (기본 포함)       │  │  │  │
│  │  │  └─────────────────────┘  │  │  │
│  │  └───────────────────────────┘  │  │
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
           * NodePort는 ClusterIP를 포함

내부 접속: database-service:5432 또는 10.43.123.45:5432 (ClusterIP)
외부 접속: 192.168.0.30:30432 (NodePort)
```

<br />
<br />

```
* 아이피 고정

각 VM의 IP 고정은 공유기 DHCP 설정에서
현재 사용 중인 MAC 주소에 IP를 예약(고정 할당)하는 방식이 권장됨.
VM에서 별도 네트워크 설정을 하지 않아도 되므로 관리가 편함.
```

<br />
<br />

```
* Ingress에 rate-limit 추가 고려 (Traefik 방식)

Traefik은 Middleware 리소스로 rate-limit을 설정함.
아래 YAML을 파일로 저장 후 kubectl apply 로 적용.
```

`middleware.yaml`

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 10   # 초당 평균 허용 요청 수
    burst: 50     # 순간 최대 허용 요청 수
```

```zsh
kubectl apply -f middleware.yaml

# Ingress annotation에 추가 (적용할 Ingress의 annotations 하위에 추가)
# traefik.ingress.kubernetes.io/router.middlewares: default-rate-limit@kubernetescrd
# 형식: <namespace>-<middleware-name>@kubernetescrd
```

<br />
<br />

```
* Minimal (최소 설치) -> Ubuntu Server (표준 설치)

sudo unminimize 명령어를 실행하면,
Ubuntu Minimal Install (최소 설치) 상태에서 표준 Ubuntu Server 환경과 동일한 수준의 패키지 셋이 설치된다.
```
