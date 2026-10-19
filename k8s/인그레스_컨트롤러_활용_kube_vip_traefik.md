# 인그레스 컨트롤러 활용하기

<br />
<br />

* 쿠버네티스 환경에서 Ingress Controller를 사용한 외부 트래픽 처리

---

```
Ingress 리소스의 주된 목적은 Ingress Controller를 통해, 
외부 트래픽을 클러스터 내부의 특정 Service 리소스와 연결하는 것이다.

Hyper-V VM을 사용해서 K3s 클러스터 구조를 만들고 MetalLB, ServiceLB 를 사용하지 않는 버전 예시이다.
(kube-vip + Traefik 사용, 하나의 클러스터에 여러 개의 인스턴스 사용, 외부 별도 DB VM 및 정적 파일 스토리지 VM 연동)

[핵심 트래픽 흐름 요약]
외부 요청
-> 공유기 포트포워딩(80, 443 -> 192.168.0.240)
-> kube-vip가 선정한 리더 워커 노드의 랜카드(보조 IP 192.168.0.240)로 유입
-> 노드의 kube-proxy(iptables)가 모든 워커 노드의 Traefik(Ingress Controller) 파드로 1차 분산(로드밸런싱)
-> Traefik이 IngressRoute 규칙을 보고 kube-proxy를 통해 각 노드의 앱 파드(next, nest)로 2차 분산(로드밸런싱)
```

<br />
<br />
<br />
<br />

0. Windows 11 Pro 호스트 및 Hyper-V VM 생성 시 필수 주의사항

```
* 베어메탈 서버(Windows 11 Pro)와 Hyper-V VM을 설정할 때 아래 항목을 반드시 먼저 적용해야 운영 중 클러스터가 뻗지 않음.

1) Windows 11 호스트 전원 관리 및 자동 재부팅 방지
   제어판 -> 전원 옵션 -> [고성능] 또는 [최고 성능] 선택, 절전 모드 진입 시간을 [해당 없음(Never)]으로 설정.
   Windows 업데이트 후 자동 재부팅으로 인해 전체 VM이 내려가지 않도록 활성 시간 설정 및 업데이트 일시 중지/관리 정책 적용.

2) 동적 메모리(Dynamic Memory) 절대 금지 -> 반드시 [고정 메모리] 할당
   Linux는 남는 램을 디스크 캐시(buff/cache)로 사용하는데, Hyper-V와 K3s(kubelet),
   Postgres 간의 메모리 계산이 꼬여 메모리가 충분한데도 파드가 강제 종료(OOMKilled)되거나 VM이 멈춤

3) 동적 확장 디스크(동적 VHDX) 지양 -> 반드시 [고정 크기 VHDX] 권장
   특히 DB VM과 Master(etcd) VM은 디스크가 실시간으로 늘어날 때 발생하는 I/O 지연(Latency)에 매우 민감함
   (단, 정적 파일 보관용 VM5와 개발 서버인 VM6는 디스크 공간 절약을 위해 동적 확장 VHDX를 사용해도 무방함)

4) 가상 스위치 및 MAC 주소 스푸핑(MAC Address Spoofing) 허용 (가장 중요)
   네트워크는 반드시 외부 가상 스위치(External Virtual Switch)로 생성하여 공유기 대역 IP를 직접 받게 설정.
   kube-vip가 ARP 모드로 가상 IP(240)를 노드의 랜카드에 바인딩하고 신호를 브로드캐스트할 때,
   Hyper-V 보안 기능이 이를 차단하지 않도록 각 K3s VM의 설정 -> [네트워크 어댑터] -> [고급 기능]에서
   MAC 주소 스푸핑 활성화(Enable MAC address spoofing)를 반드시 체크

5) 보안 부팅(Secure Boot) 해제 또는 템플릿 변경 (2세대 VM 생성 시)
   Ubuntu ISO 부팅이 막히지 않도록 [보안 부팅 사용]을 체크 해제하거나 [Microsoft UEFI 인증 기관]으로 변경

6) 스냅샷(검사점 / Checkpoints) 운영 중 비활성화 권장
   검사점이 계속 쌓이면 디스크 I/O 성능이 저하되므로 초기 세팅 완료 후 운영 중에는 자동 검사점을 끄는 것을 권장
```

<br />
<br />
<br />

1. 생성할 VM 사양 (2697v2 12코어 24스레드 / 64GB RAM 기준)

| VM  | 역할                      | CPU     | RAM  | DISK (권장 VHDX)    | VM IP 예시 (고정)   | 비고                                        |
|:---:|--------------------------|---------|------|--------------------|-------------------|--------------------------------------------|
| Host| Windows 11 Pro           | -       | -    | -                  | -                 | 호스트 OS 및 Hyper-V 여유분                    |
| VM1 | Master 1                 | 4 vCPU  | 8GB  | 60GB+ (고정)        | 192.168.0.101     | 컨트롤 플레인 + Docker 빌드 및 Private Registry |
| VM2 | Worker 1                 | 6 vCPU  | 12GB | 80GB+ (고정)        | 192.168.0.102     | Traefik, Next, Nest 구동 + NFS 클라이언트 마운트 |
| VM3 | Worker 2                 | 6 vCPU  | 12GB | 80GB+ (고정)        | 192.168.0.103     | Traefik, Next, Nest 구동 + NFS 클라이언트 마운트 |
| VM4 | DB (PostgreSQL)          | 4 vCPU  | 12GB | 100GB+ (고정)       | 192.168.0.104     | 단독 PostgreSQL 서버                         |  
| VM5 | Storage (NFS 파일 서버)    | 2 vCPU  | 2GB  | 100GB+ (고정/동적)   | 192.168.0.105     | 정적 파일(/uploads) 전용 단독 NFS 스토리지 서버    |
| VM6 | Dev Server (개발 서버)     | 2 vCPU  | 8GB  | 60GB+ (동적 가능)    | 192.168.0.106     | 독립된 개발 및 테스트용 인스턴스                   |

```
CPU 합계 -> 24 vCPU 할당 (4 + 6 + 6 + 4 + 2 + 2 = 24 스레드, 스토리지 및 개발 서버는 상시 부하가 낮으므로 CPU 경합 없이 쾌적하게 구동됨)

RAM 합계 -> VM에 총 54GB 고정 할당 (8 + 12 + 12 + 12 + 2 + 8 = 54GB) + Windows 11 호스트에 최대 10GB 확보 = 총 64GB

VIP IP -> 공유기 DHCP 할당 범위 밖의 "192.168.0.240" 이라고 가정
          공유기 관리자 페이지에서 DHCP 할당 범위를 192.168.0.2 ~ 192.168.0.200 으로 줄여두면,
          혹시 모를 201~254 대역 충돌 없이 안전하게 고정 IP 및 VIP로 사용할 수 있음
```

<br />
<br />
<br />

2. 각 VM 사전 설정 (Master, Worker, DB, Storage)

`0) 스왑(Swap) 메모리 영구 비활성화 및 커널 네트워크 설정 (각 K3s VM에서 필수)`

```zsh
# 쿠버네티스(kubelet)는 컨테이너의 정확한 메모리 사용량 추적과 성능 보장을 위해
# 스왑 메모리가 켜져 있으면 스케줄링이 꼬이거나 성능이 급격히 저하됨. (DB VM도 끄거나 최소화 권장)

# 현재 세션에서 즉시 스왑 끄기
sudo swapoff -a

# 재부팅 후에도 스왑이 켜지지 않도록 /etc/fstab 에서 swap 항목 주석 처리
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 스왑이 0B 인지 확인
free -h
```

```zsh
# 컨테이너 브릿지 네트워크 및 포워딩, 대량 트래픽 처리를 위한 필수 커널 모듈 및 sysctl 설정 (Master, Worker 공통)
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
fs.inotify.max_user_watches         = 524288
fs.inotify.max_user_instances       = 512
EOF

sudo sysctl --system
```

<br />

`1) NTP 시간 동기화 (모든 VM에서)`

```zsh
# 쿠버네티스는 노드 간 시간이 크게 어긋나면 TLS 인증 오류, etcd 이상, 토큰 검증 실패 등
# 원인을 찾기 어려운 문제가 발생
# 특히 Hyper-V VM은 호스트 절전이나 스냅샷 복구 시 시간이 틀어지기 쉬우므로 반드시 확인할 것

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

`2) 호스트명 설정 (모든 VM에서)`

```zsh
# 모든 노드의 호스트네임은 절대 중복되면 안 됨 (중복 시 워커 조인 충돌 발생)
# VM1 에서 실행
sudo hostnamectl set-hostname master-1

# VM2 에서 실행
sudo hostnamectl set-hostname worker-1

# VM3 에서 실행
sudo hostnamectl set-hostname worker-2

# VM4 에서 실행 (DB 서버)
sudo hostnamectl set-hostname db-server

# VM5 에서 실행 (정적 파일 스토리지 서버)
sudo hostnamectl set-hostname storage-server

# VM6 에서 실행 (개발 서버)
sudo hostnamectl set-hostname dev-server

# 호스트명 변경은 현재 세션에 즉시 반영되지 않음
# 변경 후 SSH 재접속 또는 아래 명령으로 새 쉘 실행
exec bash
```

<br />

`3) /etc/hosts에 등록 (모든 VM에서, IP는 예시)`

```zsh
sudo tee -a /etc/hosts << EOF
192.168.0.101 master-1
192.168.0.102 worker-1
192.168.0.103 worker-2
192.168.0.104 db-server
192.168.0.105 storage-server
192.168.0.106 dev-server
EOF
```

<br />

`4) 방화벽 설정 (Master, Worker, DB, Storage)`

```zsh
# Pod 간 통신 (CNI - Flannel VXLAN) 및 Metrics 주의사항
# K3s는 기본 CNI로 Flannel을 사용하며, 노드 간 Pod 트래픽은 VXLAN 터널링으로 전달됨.
# Pod 네트워크는 VM 네트워크와 별개의 오버레이 네트워크이므로,
# VM끼리 ping이 되더라도 UDP 8472 포트가 막혀 있으면 Pod 간 통신이 안 됨.
# 또한 10250(kubelet) 포트가 막히면 HPA(오토스케일링)가 CPU/메모리 측정값을 가져오지 못함.
```

```zsh
# Master 노드 (VM1)
sudo ufw allow 22/tcp          # SSH
sudo ufw allow 6443/tcp        # K3s API Server
sudo ufw allow 10250/tcp       # kubelet (Metrics Server 및 로그 조회 필수)
sudo ufw allow 2379:2380/tcp   # etcd
sudo ufw allow 80/tcp          # HTTP
sudo ufw allow 443/tcp         # HTTPS
sudo ufw allow 8472/udp        # Flannel VXLAN (노드 간 Pod 통신 - 반드시 필요)
sudo ufw allow 51820/udp       # Flannel WireGuard (암호화 모드 사용 시)
sudo ufw allow from 192.168.0.0/24 to any port 5000 proto tcp # Master 노드의 Private Registry 접근 허용
sudo ufw allow from 10.42.0.0/16 to any
sudo ufw allow from 10.43.0.0/16 to any
sudo ufw enable
```

```zsh
# Worker 노드 (VM2, VM3)
sudo ufw allow 22/tcp          # SSH
sudo ufw allow 10250/tcp       # kubelet (HPA 동작 및 kubectl exec/logs 필수)
sudo ufw allow 30000:32767/tcp # NodePort
sudo ufw allow 80/tcp          # HTTP
sudo ufw allow 443/tcp         # HTTPS
sudo ufw allow 8472/udp        # Flannel VXLAN (노드 간 Pod 통신 - 반드시 필요)
sudo ufw allow 51820/udp       # Flannel WireGuard (암호화 모드 사용 시)
sudo ufw allow from 10.42.0.0/16 to any
sudo ufw allow from 10.43.0.0/16 to any
sudo ufw enable
```

```zsh
# DB 노드 (VM4 - PostgreSQL 단독 서버)
# 외부 전체에 5432를 열지 않고 내부 사설망(192.168.0.0/24)에서의 접근만 허용하여 보안 강화
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.0.0/24 to any port 5432 proto tcp
sudo ufw enable
```

```zsh
# Storage 노드 (VM5 - 정적 파일 전용 단독 NFS 서버)
# 내부 사설망(192.168.0.0/24)에서의 NFS(2049) 접근만 허용하여 보안 강화
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.0.0/24 to any port 2049 proto tcp
sudo ufw enable
```

<br />

`5) 사설 Docker Registry (HTTP) 허용 설정 (Master, Worker 공통)`

```zsh
# Deployment에서 Master 노드(192.168.0.101:5000)에 띄운 사설 레지스트리의 이미지를 pull 받을 때,
# K3s(containerd)는 기본적으로 HTTPS만 허용하므로 HTTP 통신을 허용하도록 미리 등록해야 ImagePullBackOff 오류가 나지 않음.

sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "192.168.0.101:5000":
    endpoint:
      - "http://192.168.0.101:5000"
EOF

# 참고: K3s 설치 전에 미리 만들어두면 설치 시 자동 반영되며,
# 만약 K3s 설치 후에 수정했다면 마스터는 sudo systemctl restart k3s, 워커는 sudo systemctl restart k3s-agent 실행 필요.
```

<br />

`6) DB VM (VM4 - 4 vCPU / 12GB RAM) PostgreSQL 필수 튜닝 및 자동 백업 설정`

```zsh
# PostgreSQL 설치 후 기본 설정은 메모리를 128MB만 쓰도록 되어 있어 할당된 12GB RAM을 전혀 활용하지 못함
# 반드시 아래 설정을 postgresql.conf 및 pg_hba.conf 에 적용할 것.

# 1. 외부(워커 노드 및 개발 서버) 접속 허용 설정
# /etc/postgresql/<버전>/main/postgresql.conf 파일 수정:
# listen_addresses = '*'

# /etc/postgresql/<버전>/main/pg_hba.conf 파일 하단에 추가:
# host    all             all             192.168.0.0/24          scram-sha-256

# 2. 4 vCPU / 12GB RAM 맞춤 성능 튜닝 (/etc/postgresql/<버전>/main/postgresql.conf)
# shared_buffers = 3GB                  # 전체 RAM(12GB)의 25%
# effective_cache_size = 9GB            # 전체 RAM(12GB)의 75%
# maintenance_work_mem = 768MB
# work_mem = 16MB
# max_connections = 200                 # Nest 파드가 최대 4개로 늘어날 때의 커넥션 풀 여유분 확보
# random_page_cost = 1.1                # SSD 환경 최적화

# 설정 변경 후 재시작
# sudo systemctl restart postgresql
```

```zsh
# 3. DB 자동 백업(pg_dump) 크론탭 설정 (DB VM인 VM4에서 실행)
# 단일 베어메탈 서버 특성상 디스크 장애에 대비해 매일 새벽 자동 백업 및 7일 이상 된 백업본 자동 삭제 설정 권장
sudo mkdir -p /var/backups/postgres
sudo chown postgres:postgres /var/backups/postgres

# postgres 유저의 crontab에 매일 새벽 3시 DB 덤프 백업 등록 (pro_one 은 실제 DB 이름으로 변경)
sudo crontab -u postgres -l 2>/dev/null | { cat; echo "0 3 * * * pg_dump -Fc pro_one > /var/backups/postgres/pro_one_\$(date +\%F).dump && find /var/backups/postgres/ -type f -mtime +7 -delete"; } | sudo crontab -u postgres -
```

<br />

`7) Storage VM (VM5 - 2 vCPU / 2GB RAM) 단독 NFS 서버 구축 및 워커 노드 마운트`

```
* 왜 정적 파일(/uploads) 전용 Storage VM(NFS)을 사용하는가?
  - 워커 노드가 2대(Worker 1, Worker 2)이므로, 각 워커 노드의 로컬 디스크(/uploads)에 파일을 그냥 저장하면
    Worker 1의 파드에 업로드한 이미지나 PDF를 Worker 2의 파드에서는 찾지 못해 404 에러가 발생함.
  - 따라서 독립된 Storage VM(192.168.0.105)에 원본 폴더(/srv/nfs/uploads)를 만들고 NFS로 공유한 뒤,
    Worker 1과 Worker 2가 각자의 /uploads 경로에 이를 마운트하면 코드 수정 없이 모든 파드가 동일한 파일들을 실시간으로 읽고 쓸 수 있음.
```

```zsh
# [1단계: Storage VM (VM5 - 192.168.0.105) 에서 실행]
# NFS 서버 패키지 설치 및 원본 디렉토리 생성
sudo apt-get update && sudo apt-get install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/uploads
sudo chown nobody:nogroup /srv/nfs/uploads
sudo chmod 777 /srv/nfs/uploads

# 워커 노드(192.168.0.102, 103) 및 개발 서버(192.168.0.106)에서 읽고 쓸 수 있도록 export 설정 등록
# no_root_squash 옵션을 넣어야 컨테이너 내부 프로세스가 권한 오류 없이 파일을 생성/수정할 수 있음
cat <<EOF | sudo tee -a /etc/exports
/srv/nfs/uploads 192.168.0.102(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/uploads 192.168.0.103(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/uploads 192.168.0.106(rw,sync,no_subtree_check,no_root_squash)
EOF

# 설정 반영 및 서비스 활성화
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server

# (선택) Storage VM 내부에서 매일 새벽 3시 30분 정적 파일 압축 백업 크론탭 등록
sudo mkdir -p /var/backups/uploads
sudo crontab -l 2>/dev/null | { cat; echo "30 3 * * * tar -czf /var/backups/uploads/uploads_\$(date +\%F).tar.gz -C /srv/nfs uploads && find /var/backups/uploads/ -type f -mtime +7 -delete"; } | sudo crontab -
```

```zsh
# [2단계: Worker 1 (VM2) 및 Worker 2 (VM3) 에서 각각 실행]
# NFS 클라이언트 패키지 설치 및 마운트할 /uploads 폴더 생성
sudo apt-get update && sudo apt-get install -y nfs-common
sudo mkdir -p /uploads

# 재부팅 시에도 Storage VM(192.168.0.105)의 폴더가 자동 마운트되도록 /etc/fstab 에 등록 후 즉시 마운트
echo "192.168.0.105:/srv/nfs/uploads /uploads nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a

# 마운트가 정상적으로 되었는지 확인 (192.168.0.105:/srv/nfs/uploads 항목이 보여야 함)
df -h | grep uploads
```

<br />
<br />
<br />

3. VM1에서 Master 1 설치

```zsh
# 디렉토리 생성
sudo mkdir -p /var/lib/rancher/k3s/server/manifests

# Traefik 설정 파일 생성
# - replicas: 2 로 설정하여 워커 노드에 각각 파드가 떠서 가용성 및 부하 분산 확보
# - externalTrafficPolicy: Cluster (기본값)를 유지하여 VIP가 어느 워커에 잡히든
#   kube-proxy가 워커1, 워커2의 Traefik 파드로 트래픽을 균등하게 로드밸런싱하도록 함
sudo tee /var/lib/rancher/k3s/server/manifests/traefik-config.yaml > /dev/null <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    deployment:
      replicas: 2
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values:
                      - traefik
              topologyKey: "kubernetes.io/hostname"
EOF
```

```zsh
# ServiceLB 비활성화 (kube-vip 사용), Traefik 유지, TLS SAN 추가, 시스템 리소스 예약 설정
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=servicelb --tls-san=192.168.0.101 --kubelet-arg=system-reserved=cpu=500m,memory=1Gi" sh -

# K3s 서비스 상태 확인
sudo systemctl status k3s

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

# kubeconfig 파일 권한을 반드시 600으로 설정
# 권한이 너무 열려 있으면 kubectl이 경고를 내거나 동작을 거부함
chmod 600 ~/.kube/config

# k3s.yaml의 server 주소는 기본적으로 127.0.0.1로 되어 있음
# 마스터 노드 내부에서는 동작하지만, 외부 PC에서 kubectl로 원격 접근 시 이 파일을 그대로 쓰면 접속 불가
# 외부 접근이 필요한 경우 server 주소를 실제 마스터 IP로 교체
sed -i "s/127.0.0.1/192.168.0.101/g" ~/.kube/config

# 영구 적용
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc

# 노드 및 Traefik 파드 개수 확인 (2개가 떠야 함)
kubectl get nodes
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik -o wide
```

```zsh
# taint 설정 (toleration이 없는 일반 앱 파드 및 kube-vip 파드가 마스터에 안 올라가고 워커에만 올라가도록 강제함)
# 마스터 노드의 API Server와 Docker 빌드 자원을 보호하기 위해 설정하며,
# 반드시 5번(kube-vip 설치) 진행 전에 실행되어 있어야 kube-vip가 마스터 노드를 피해 워커 노드에만 설치됨
kubectl taint nodes master-1 node-role.kubernetes.io/control-plane:NoSchedule --overwrite
```

<br />
<br />
<br />

4. VM2, VM3에서 K3s Worker 조인

`VM2, VM3에서 실행`

```zsh
# export 방식은 SSH 세션이 끊기면 환경변수가 휘발되어 설치 실패 가능
# 한 줄로 인라인 전달하는 방식이 더 안전함
# --kubelet-arg 옵션을 통해 워커 노드의 OS와 K3s 데몬이 쓸 최소 자원(1Gi RAM)을 예약하여 노드 전체 멈춤(NotReady)을 방지함
curl -sfL https://get.k3s.io | K3S_URL="https://192.168.0.101:6443" K3S_TOKEN="<토큰값>" INSTALL_K3S_EXEC="--kubelet-arg=system-reserved=cpu=500m,memory=1Gi --kubelet-arg=kube-reserved=cpu=500m,memory=1Gi" sh -
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

```zsh
# 마스터에 Taint를 걸었으므로 기존에 마스터에 떠 있던 파드들을 워커1, 워커2로 재배치하기 위해
# Traefik과 Metrics Server 등을 재시작하여 워커 노드로 이동시킴
kubectl rollout restart deployment traefik -n kube-system
kubectl rollout restart deployment metrics-server -n kube-system
kubectl rollout restart deployment coredns -n kube-system

# 파드들이 worker-1, worker-2에 잘 분산되어 떴는지 확인
kubectl get pods -n kube-system -o wide
```

<br />

`노드 간 Pod 통신 확인 (VM1에서)`

```zsh
# busybox:latest는 pull 실패가 간혹 발생하므로 버전 고정 사용
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

5. kube-vip 설치 (LoadBalancer용 가상 IP)

```
* 왜 VIP(192.168.0.240) 하나만 사용하고 IP Pool(MetalLB)은 쓰지 않는가?
  - 홈랩/단일 진입점 환경에서는 공유기 포트포워딩(80, 443)이 바라볼 단 하나의 고정 IP만 있으면 됨.
  - 클러스터 내부의 라우팅은 Ingress Controller(Traefik)가 도메인과 경로(/api 등)를 보고 각 Service로 분배하므로, 여러 개의 외부 IP를 찍어내는 무거운 IP Pool(MetalLB)은 불필요함.

* kube-vip의 진짜 역할과 로드밸런싱의 오해
  - kube-vip 자체는 트래픽을 분산하는 로드밸런서(LB)가 아님.
  - 워커 노드 중 한 곳(리더)의 랜카드에 192.168.0.240(보조 IP)을 달아주고, 그 노드가 죽으면 다른 워커 노드로 IP를 즉시 옮겨주는 '고가용성(HA) 문지기' 역할만 수행함.
  - 실제 트래픽 분산(Load Balancing)은 패킷이 노드에 도착한 직후, 리눅스 커널(kube-proxy / iptables)이 낚아채서 모든 워커 노드의 Traefik 파드들로 50:50 균등하게 뿌려줌.

* 왜 마스터 노드는 제외하고 워커 노드에만 설치하는가?
  - 현재 구성은 단일 마스터(1 Master)이므로 API 서버(6443) 이중화(--controlplane)가 필요 없음.
  - 명령어에서 --taint 옵션을 빼서 마스터 노드(Taint가 걸려 있음)에는 kube-vip가 스케줄링되지 않게 함.
  - 이렇게 하면 VIP가 무조건 워커 노드(Worker 1 또는 Worker 2)에만 잡히게 되어, 외부 트래픽이 마스터 노드를 거치지 않고 곧바로 워커 노드로 진입하므로 마스터 노드의 자원을 온전히 보호할 수 있음.
```

<br />

```zsh
# kube-vip 네트워크 환경 전제 조건
# 이 매뉴얼의 kube-vip 설정은 ARP 모드 기준임.
# ARP 모드는 같은 L2 네트워크(공유기 하위 사설망)에서만 동작함.
# 일부 클라우드 VM 또는 특정 네트워크 환경에서는 ARP broadcast가 차단되어 VIP가 동작하지 않을 수 있음.
# → 해당 환경에서는 BGP 모드 사용 필요: https://kube-vip.io/docs/installation/bgp/
```

<br />

```zsh
# 1) RBAC 먼저 적용
# 재설치 시 이미 리소스가 존재하면 충돌 오류가 날 수 있음
# 기존 RBAC 리소스가 있으면 먼저 삭제 후 재적용
# kubectl delete -f https://kube-vip.io/manifests/rbac.yaml 2>/dev/null; true
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml

# 2) K3s 내장 ctr 명령어로 kube-vip DaemonSet YAML 생성 및 적용 (별도 Docker 설치 불필요)
# --interface: 실제 네트워크 인터페이스명으로 변경 (ip a 또는 ip link 로 확인, 보통 eth0)
# --address: 공유기 DHCP 할당 범위 밖의 고정 IP로 설정 (예: 공유기 DHCP 범위를 2~200으로 줄이고 240 사용)
# 제외된 옵션 설명:
#   --controlplane 제외 : 단일 마스터이므로 API 서버용 VIP 불필요 (제외 시 --address 가 서비스 전용 VIP로 동작)
#   --taint 제외        : 마스터 노드의 Taint를 무시하는 옵션을 빼서, 오직 워커 노드에만 kube-vip 파드가 뜨도록 강제함
sudo k3s ctr images pull ghcr.io/kube-vip/kube-vip:v0.8.7
sudo k3s ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v0.8.7 vip /kube-vip manifest daemonset \
  --interface eth0 \
  --address 192.168.0.240 \
  --inCluster \
  --services \
  --arp \
  --leaderElection | kubectl apply -f -
```

```zsh
# 생성된 YAML의 Pod label을 확인 후 wait 실행
# 마스터를 제외했으므로 워커 노드 개수(2개)만큼만 kube-vip 파드가 Running 상태로 떠야 정상임
kubectl get pods -n kube-system -o wide | grep kube-vip

kubectl wait --namespace kube-system --for=condition=ready pod --selector=app.kubernetes.io/name=kube-vip-ds --timeout=90s

# kube-vip Pod가 워커 노드(worker-1, worker-2)에만 잘 떠 있는지 최종 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
```

<br />
<br />
<br />

6. Ingress Controller 설정 (K3s 기본 Traefik)

```zsh
# patch 전에 Traefik Pod 2개가 각 워커 노드에 정상 Running 상태인지 먼저 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik -o wide
# STATUS가 Running이어야 다음 단계 진행

# Traefik Service에 고정 VIP(192.168.0.240)를 할당하도록 패치
# kube-vip 파드가 이 설정을 감지하고 리더 워커 노드의 랜카드(eth0)에 192.168.0.240을 보조 IP로 바인딩함
kubectl patch svc traefik -n kube-system \
  --type=merge \
  -p '{
    "spec": {
      "type": "LoadBalancer",
      "loadBalancerIP": "192.168.0.240"
    },
    "metadata": {
      "annotations": {"kube-vip.io/requestedIP": "192.168.0.240"}
    }
  }'
```

```zsh
# patch 후 kube-vip가 감지하여 EXTERNAL-IP를 부여할 때까지 확인
kubectl get svc -n kube-system traefik -w
# EXTERNAL-IP 컬럼에 192.168.0.240이 표시되면 정상 (Ctrl+C로 종료)
kubectl get svc -n kube-system traefik

# (선택) 어느 워커 노드가 리더가 되어 랜카드에 192.168.0.240을 가져갔는지 확인하려면
# 각 워커 노드에서 ip a show eth0 실행 시 inet 192.168.0.240/32 가 보이는 노드가 현재 리더임
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

# wait 완료 후에도 CRD API 등록까지 수 초 더 걸릴 수 있음
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

<br />
<br />
<br />

8. 도메인 DNS 설정

```
도메인 구입처 DNS에서 A 레코드 추가. 
전파는 수분~최대 24시간 소요될 수 있음.

Ingress 적용 전에 DNS 전파가 완료되어 있어야 인증서 발급이 정상 동작함.

참고: 가정집/사무실 유동 공인 IP 환경인 경우, 공유기 재부팅 시 공인 IP가 바뀔 수 있으므로
도메인 업체(Cloudflare 등)의 DDNS 스크립트를 돌리거나 공유기 자체 DDNS 연동 기능을 활용하여
공인 IP 변경 시 A 레코드가 자동 갱신되도록 구성하는 것을 권장함.
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

9. 공유기 포트포워딩 및 DHCP 범위 설정

```
공유기 DHCP 할당 범위 조정 (필수)
   공유기가 다른 기기(스마트폰, PC 등)에 VIP 주소(192.168.0.240)나 VM 고정 IP를 자동 할당하면 IP 충돌이 발생함.
   공유기 관리자 페이지 -> 내부 네트워크 설정 -> DHCP 할당 범위를 192.168.0.2 ~ 192.168.0.200 으로 줄일 것!

cert-manager의 HTTP01 챌린지는 외부 → 80포트 → VIP(192.168.0.240) → Traefik 흐름으로 동작함.
   포트포워딩이 설정되지 않으면 인증서 발급이 실패하므로 Ingress 적용 전에 반드시 먼저 설정할 것.

ISP(KT, SKB 등 일부 요금제)에서 80/443 포트를 차단하는 경우가 있음.
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

10. App Deployment 및 Service 설정

```zsh
# Service는 selector로 Pod를 찾습니다.
# nest-svc → app: nest 라벨이 있는 Pod
# next-svc → app: next 라벨이 있는 Pod
# Deployment가 먼저 배포되어 있어야 Endpoint가 연결됩니다.
# Endpoint 확인: kubectl get endpoints nest-svc
# Endpoint가 <none> 이면 Deployment가 없거나 라벨이 일치하지 않는 것

# 주의 1: 무중단 배포와 고가용성을 위한 기본 파드 개수 (replicas: 2, minReplicas: 2)
# 프론트(next)와 백엔드(nest)가 각각 1개의 프로젝트더라도, 파드를 1개만 띄우면 배포 시나 노드 점검 시 서비스가 끊김.
# 따라서 기본 2개(워커1에 1개, 워커2에 1개)를 상시 띄워두어 무중단 배포(RollingUpdate)와 노드 장애 대응을 보장함.

# 주의 2: Pod 분산을 위한 podAntiAffinity 설정 시 주의사항!
# 워커 노드가 2개인 상태에서 requiredDuringScheduling(강제 분산)을 쓰면,
# HPA가 파드를 3개, 4개로 스케일아웃할 때 "파드 겹침 금지" 규칙 때문에 3번째 파드부터 Pending에 빠져 뜨지 못함!
# 따라서 반드시 preferredDuringScheduling(가급적 분산하되, 노드가 모자라면 한 노드에 여러 개 허용)을 사용해야 함.

# 주의 3: 롤링 업데이트 시 기존 트래픽 유실 방지 (preStop 훅 적용)
# 파드가 내려갈 때 kube-proxy의 엔드포인트 목록에서 제거되는 시간(수 초) 동안 이미 들어오던 요청이 끊기지 않도록
# lifecycle.preStop 에 sleep 5 를 주어 안전하게 무중단 종료(Graceful Shutdown)되도록 구성함.

# 주의 4: 정적 파일 공유 폴더(/uploads) 볼륨 마운트
# 2-7번 단계에서 각 워커 노드의 /uploads 에 마운트해 둔 Storage VM(192.168.0.105)의 NFS 폴더를
# 파드 내부의 /uploads 로 연결(hostPath)하여 어느 워커 노드에 뜬 파드든 동일한 이미지/PDF/텍스트 파일에 접근할 수 있게 함.
```

<br />

`nest-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nest
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nest
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # 배포중 Pod 1개 추가로 먼저 띄움
      maxUnavailable: 0 # 기존 Pod는 새 Pod 준비되기 전까지 안죽임
  template:
    metadata:
      labels:
        app: nest
    spec:
      terminationGracePeriodSeconds: 30
      # 동일 앱 replica를 가급적 서로 다른 노드에 분산하되,
      # HPA 스케일아웃(최대 4개)이나 롤링 업데이트 시 워커가 2대뿐이어도 Pending 없이 뜨도록 preferred 사용
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - nest
                topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nest
        image: 192.168.0.101:5000/nest:latest
        imagePullPolicy: Always
        command: ["node"]
        args: ["dist/main.js"]
        ports:
        - containerPort: 4000
        envFrom:
        - secretRef:
            name: nest-secret
        volumeMounts:
        - name: uploads-volume
          mountPath: /uploads
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        # 앱 초기화 완료 전까지 liveness/readiness 보호
        # 초기화 로직 없으면 제거하고 livenessProbe initialDelaySeconds 복구
        startupProbe:
          httpGet:
            path: /              # /health 엔드포인트 생기면 변경 권장
            port: 4000
          failureThreshold: 30   # 최대 30 * 10초 = 300초 대기
          periodSeconds: 10
        # 운영중 먹통되면 자동 재시작
        livenessProbe:
          httpGet:
            path: /              # /health 엔드포인트 생기면 변경 권장
            port: 4000
          initialDelaySeconds: 0 # startupProbe 가 대신함
          periodSeconds: 10
        # 배포 시 준비될때까지 트래픽 안받음
        readinessProbe:
          httpGet:
            path: /              # /health 엔드포인트 생기면 변경 권장
            port: 4000
          initialDelaySeconds: 0 # startupProbe 가 대신함
          periodSeconds: 5
      volumes:
      - name: uploads-volume
        hostPath:
          path: /uploads
          type: DirectoryOrCreate
```

<br />

`nest-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nest-svc
  namespace: default
  labels:
    app: nest
spec:
  type: ClusterIP
  ports:
  - port: 4000
    targetPort: 4000
    protocol: TCP
    name: http
  selector:
    app: nest
```

<br />

`nest-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nest-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nest
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  # 트래픽이 잠깐 빠졌다고 파드를 즉시 죽였다가 다시 띄우는 플래핑(Flapping) 현상 방지
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # 부하가 줄어도 5분간 추이를 지켜본 뒤 천천히 축소
```

<br />

`nest-pdb.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nest-pdb
  namespace: default
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nest
```

<br />

`nest-secret.yaml`

```yaml
# 프로덕션용 시크릿 예시.
# 1. 이 파일을 nest-secret.yaml 로 복사한 뒤 실제 값으로 채우세요.
# 2. nest-secret.yaml 은 .gitignore에 넣고 커밋하지 마세요.
# 3. 적용: kubectl apply -f nest-secret.yaml
#
# 중요: DB가 별도 Hyper-V VM(예: 192.168.0.104)에 있으므로,
# DATABASE_URL에는 외부 도메인(healthapp.shop)이 아니라 반드시 내부 사설 IP(192.168.0.104:5432)를 적어야
# 공유기를 거치지 않고 Hyper-V 내부 스위치를 통해 초고속으로 직접 통신함!
#
# 또는 한 번에 생성:
#   kubectl create secret generic nest-secret \
#     --from-literal=NODE_ENV=production \
#     --from-literal=PORT=4000 \
#     --from-literal=DATABASE_URL='postgresql://codej625:0625@192.168.0.104:5432/pro_one?schema=public' \
#     --from-literal=JWT_SECRET='운영용-강한-랜덤-문자열' \
#     --from-literal=CORS_ORIGIN='https://healthapp.shop' \
#     -n default
---
apiVersion: v1
kind: Secret
metadata:
  name: nest-secret
  namespace: default
type: Opaque
stringData:
  NODE_ENV: "production"
  PORT: "4000"
  # PostgreSQL 연결 문자열 (별도 DB VM의 사설 IP 사용 권장)
  DATABASE_URL: "postgresql://{id}:{password}@192.168.0.104:5432/{database_name}?schema=public"
  # 디비 연결 시 지연될 수 있을 때 사용 (커넥션 타임아웃 및 커넥션 풀 제한 옵션)
  # DATABASE_URL: "postgresql://{id}:{password}@192.168.0.104:5432/{database_name}?schema=public&connect_timeout=30&connection_limit=20"
  # JWT 서명용 시크릿 (운영에서 반드시 강한 랜덤 문자열로 변경)
  JWT_SECRET: "REPLACE_WITH_STRONG_RANDOM_SECRET"
  # CORS 허용 오리진 (프론트엔드 URL, 쉼표 여러 개 가능)
  CORS_ORIGIN: "https://healthapp.shop"
```

<br />

`next-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: next-config
  namespace: default
data:
  API_SERVER_URL: "https://healthapp.shop"
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
  replicas: 2
  selector:
    matchLabels:
      app: next
  # 배포 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 배포중 Pod 1개 추가로 먼저 띄움
      maxUnavailable: 0  # 기존 Pod는 새 Pod 준비되기 전까지 안죽임
  template:
    metadata:
      labels:
        app: next
    spec:
      terminationGracePeriodSeconds: 30
      # 동일 앱 replica를 가급적 서로 다른 노드에 분산하되,
      # HPA 스케일아웃(최대 4개) 시 워커가 2대뿐이어도 Pending 없이 뜨도록 preferred 사용
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - next
                topologyKey: "kubernetes.io/hostname"
      containers:
      - name: next
        envFrom:
          - configMapRef:
              name: next-config
        image: 192.168.0.101:5000/next:latest
        imagePullPolicy: Always
        command: ["node"]
        args: ["server.js"]
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: uploads-volume
          mountPath: /uploads
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1500m"  # SSR 렌더링 스파이크 대비 1코어 이상 허용
            memory: "768Mi"
        # 앱 초기화 완료 전까지 liveness/readiness 보호
        # 초기화 로직 없으면 제거하고 livenessProbe initialDelaySeconds 복구
        startupProbe:
          httpGet:
            path: /    # /health 엔드포인트 생기면 변경 권장
            port: 3000
          failureThreshold: 30  # 최대 30 * 10초 = 300초 대기
          periodSeconds: 10
        # 배포 시 준비될때까지 트래픽 안받음
        readinessProbe:
          httpGet:
            path: /    # /health 엔드포인트 생기면 변경 권장
            port: 3000
          initialDelaySeconds: 0  # startupProbe 가 대신함
          periodSeconds: 5
        # 운영중 먹통되면 자동 재시작
        livenessProbe:
          httpGet:
            path: /    # /health 엔드포인트 생기면 변경 권장
            port: 3000
          initialDelaySeconds: 0  # startupProbe 가 대신함
          periodSeconds: 10
      volumes:
      - name: uploads-volume
        hostPath:
          path: /uploads
          type: DirectoryOrCreate
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
    name: next # Deployment 이름과 반드시 일치
  minReplicas: 2
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # SSR 렌더링 특성상 순간 CPU 스파이크로 스케일아웃된 후 너무 빨리 줄어드는 현상 방지
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # 5분간 안정화 대기 후 축소
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

```zsh
kubectl apply -f nest-secret.yaml
kubectl apply -f nest-deployment.yaml
kubectl apply -f nest-svc.yaml
kubectl apply -f nest-pdb.yaml
kubectl apply -f nest-hpa.yaml
kubectl get svc nest-svc
kubectl get endpoints nest-svc # Pod와 연결 확인

kubectl apply -f next-configmap.yaml
kubectl apply -f next-deployment.yaml
kubectl apply -f next-svc.yaml
kubectl apply -f next-pdb.yaml
kubectl apply -f next-hpa.yaml
kubectl get svc next-svc
kubectl get endpoints next-svc # Pod와 연결 확인
```

<br />
<br />
<br />

11. IngressClass (IngressRoute 방식 사용 시 IngressClass 불필요)

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

# Ingress 리소스 방식(cert-manager HTTP01 solver 포함) 사용 시 필요
```

<br />
<br />
<br />

12. Ingress 라우팅 설정

```zsh
# Ingress apply 전 ClusterIssuer가 Ready 상태인지 반드시 확인
# READY: True 가 아닌 상태에서 apply 하면 인증서 발급이 시작되지 않음
kubectl get clusterissuer letsencrypt-prod
```

<br />

`ingressroute.yaml`

```yaml
# 경로로 나누기 - strip-api-prefix 미들웨어 포함 버전

# ----- HTTP (80) -----
# healthapp.shop → HTTPS로 리다이렉트 후 next-svc(프론트)로
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: main-ingressroute
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`healthapp.shop`)
    kind: Rule
    middlewares:
    - name: redirect-https
    services:
    - name: next-svc
      port: 3000
---
# ----- HTTPS (443) -----
# - /api/* → nest-svc(백엔드). strip-api-prefix로 /api 제거 후 전달 (예: /api/auth/login → /auth/login)
# - 그 외 → next-svc(프론트)
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: main-ingressroute-tls
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`healthapp.shop`) && PathPrefix(`/api`)
    kind: Rule
    middlewares:
    - name: strip-api-prefix
    - name: rate-limit
    services:
    - name: nest-svc
      port: 4000
  - match: Host(`healthapp.shop`)
    kind: Rule
    middlewares:
    - name: rate-limit
    services:
    - name: next-svc
      port: 3000
  tls:
    secretName: example-tls
```

<br />


```yaml
# 경로로 나누기 - 미들웨어 없이 /api 그대로 전달하는 버전
# (백엔드가 /api 접두사를 직접 처리하는 경우)

apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: main-ingressroute
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`healthapp.shop`)
    kind: Rule
    middlewares:
    - name: redirect-https
    services:
    - name: next-svc
      port: 3000
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: main-ingressroute-tls
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`healthapp.shop`) && PathPrefix(`/api`)
    kind: Rule
    services:
    - name: nest-svc
      port: 4000
  - match: Host(`healthapp.shop`)
    kind: Rule
    services:
    - name: next-svc
      port: 3000
  tls:
    secretName: example-tls
```

<br />

`redirect-https-middleware.yaml`

```yaml
# Ingress에 HTTPS 설정만 되어 있으면 HTTP 요청은 매칭되는 라우트가 없어 404 나옴

apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
    port: "443"
```

<br />

`strip-api-prefix-middleware.yaml`

```yaml
# Traefik 미들웨어: 요청 경로에서 /api 제거
# 프론트는 /api/auth/login 으로 보내고, 백엔드(nest)는 /auth/login 만 받도록 함.
# 적용: ingressroute.yaml 의 /api 라우트에서 이 미들웨어 참조.
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: strip-api-prefix
  namespace: default
spec:
  replacePathRegex:
    regex: "^/api/?(.*)"
    replacement: "/${1}"
```

<br />

`rate-limit-middleware.yaml`

```yaml
# Traefik 미들웨어: 과도한 요청 차단 (Rate Limit)
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 100   # 초당 평균 허용 요청 수
    burst: 300     # 순간 최대 허용 요청 수
```

<br />

`certificate.yaml`

```yaml
# IngressRoute 사용 시 인증서 직접 발급

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
  - healthapp.shop
```

<br />

```zsh
# IngressRoute는 cert-manager 어노테이션이 없으므로 Certificate 리소스로 직접 발급 필요
# 또한 IngressRoute가 참조하는 Middleware(redirect-https, strip-api-prefix, rate-limit)가
#    먼저 생성되어 있지 않으면 라우팅이 전체 실패하므로 반드시 Middleware를 먼저 apply 할 것!

kubectl apply -f certificate.yaml
kubectl get certificate -A -w # READY: True 될 때까지 대기

kubectl apply -f redirect-https-middleware.yaml
kubectl apply -f strip-api-prefix-middleware.yaml
kubectl apply -f rate-limit-middleware.yaml
kubectl apply -f ingressroute.yaml

# 확인
kubectl get ingressroute -n default
kubectl get middleware -n default
```

```zsh
# Let's Encrypt 발급 실패 시 주의사항
# 같은 도메인에 대해 1시간 내 5회 이상 실패하면 일시적으로 차단됨 (재시도 쿨다운 발생)
# 반드시 아래 명령으로 실패 원인을 먼저 확인하고 수정 후 재시도할 것

kubectl describe certificate -A
kubectl describe certificaterequest -A
kubectl logs -n cert-manager -l app=cert-manager --tail=100

# HTTP01 챌린지 전제 조건
# cert-manager가 HTTP01 챌린지를 수행하려면 Let's Encrypt 서버가 외부에서 도메인의 80포트로 접근 가능해야 함.
# 공유기 포트포워딩(9번 항목)이 먼저 설정되어 있어야 인증서 발급이 성공함.
# 일부 ISP (KT, SKB 등 일부 요금제)는 80/443 포트 자체를 차단하는 경우가 있음.
# → 해당 환경에서는 DNS01 챌린지 방식으로 전환 필요: https://cert-manager.io/docs/configuration/acme/dns01/
```

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
# 자동 갱신 실패 조건
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
  - healthapp.shop
  renewBefore: 720h  # 만료 30일 전 갱신 (예시값). 더 일찍 갱신하려면 값을 높임 (예: 1440h = 60일 전)
```

```zsh
# IngressRoute 방식에서는 cert-manager 어노테이션이 없으므로 Certificate 리소스로 직접 관리.
# 갱신 시점만 바꾸고 싶은 경우에만 위 YAML로 수동 관리.
# 일반적으로는 Certificate 리소스가 있으면 cert-manager가 자동 갱신함.
```

<br />
<br />
<br />

14. 전체 시스템 확인

```zsh
kubectl get all -A
kubectl get ingressroute -A
kubectl get svc -n kube-system traefik
kubectl get certificate -A
kubectl get hpa -A
kubectl top nodes
kubectl top pods -A
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=100
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=50
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

# Let's Encrypt는 같은 도메인으로 1시간 내 5회 발급 실패 시 일시 차단됨
# 원인을 반드시 먼저 파악하고 수정한 뒤 재시도할 것
```

<br />

`Service Endpoint가 연결 안 될 때 (502 Bad Gateway)`

```zsh
# Endpoint 확인 - <none> 이면 Deployment가 없거나 라벨 불일치
kubectl get endpoints nest-svc
kubectl get endpoints next-svc

# Pod 상태 확인
kubectl get pods -n default -o wide
kubectl describe pod <pod-name>
```

<br />

`kube-vip VIP가 안 잡힐 때 또는 외부 접속이 안 될 때`

```zsh
# 1) kube-vip 로그 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=100

# 2) Traefik EXTERNAL-IP 확인 (Pending 이면 VIP 미할당)
kubectl get svc -n kube-system traefik

# 3) 인터페이스명 확인 (eth0 이 아닌 경우 kube-vip 재생성 필요)
ip a
ip link

# 4) EXTERNAL-IP는 192.168.0.240으로 잡혔는데 공유기나 다른 PC에서 ping/접속이 안 될 때
# → Hyper-V VM 설정 -> [네트워크 어댑터] -> [고급 기능]에서
#   [MAC 주소 스푸핑 활성화]가 체크되어 있는지 반드시 확인!
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

# Pod IP 대역(기본 10.42.x.x)이 VM 네트워크와 겹치는 경우에도 통신 문제 발생 가능
# K3s 설치 시 --cluster-cidr 옵션으로 대역 변경 가능
```

<br />

`전체 흐름 점검 순서`

```
공인IP 확인 → 포트포워딩 확인 → DNS 전파 확인 (nslookup/dig)
→ VIP 확인 (kubectl get svc traefik 및 Hyper-V MAC 스푸핑 확인)
→ 인증서 확인 (kubectl get certificate) → 미들웨어 확인 (kubectl get middleware)
→ Endpoint 확인 (kubectl get endpoints) → 앱 Pod 확인 (kubectl get pods -o wide)
→ Pod 간 통신 확인 (kubectl exec ping 테스트)
```

<br />
<br />
<br />
<br />

* TIP

```
* 클러스터 내부 및 외부 DB / Storage 연결 방식

1) DB가 클러스터 내부 Pod로 떠 있을 때:
   Service 이름으로 지정해놓으면 됨.
   ex) PostgreSQL → postgresql://id:pw@database-service:5432/dbname

2) DB가 클러스터 외부 별도 Hyper-V VM(예: 192.168.0.104)에 떠 있을 때 (현재 아키텍처):
   공인 도메인(healthapp.shop)을 쓰지 말고, 사설 IP(192.168.0.104)를 직접 지정할 것!
   ex) PostgreSQL → postgresql://id:pw@192.168.0.104:5432/dbname

3) 정적 파일 스토리지(NFS)가 별도 Hyper-V VM(예: 192.168.0.105)에 떠 있을 때 (현재 아키텍처):
   각 워커 노드의 /uploads 경로에 NFS를 마운트하고, 파드에서는 hostPath(/uploads)로 연결하여
   어느 노드에 뜬 파드든 로컬 폴더처럼 투명하게 파일을 읽고 쓸 수 있음.

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

내부 접속 -> nest-svc:4000 또는 10.43.123.45:4000 (ClusterIP)
외부 접속 -> 192.168.0.240:80, 443 (kube-vip가 바인딩한 LoadBalancer IP)
```
