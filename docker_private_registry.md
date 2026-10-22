# Private Registry를 사용하는 반자동 배포 (용량 최적화 및 롤백 기능)

<br />
<br />

* 왜 Private Registry를 사용할까?

---

```
외부 Registry(Docker Hub, ghcr.io 등) 사용이 제한되거나 유료 플랜이 부담스러운 환경에서,
Master Node에 직접 Private Registry를 띄워 Worker Node들이 이미지를 내부 네트워크로 pull 할 수 있게 한다.

* 핵심 아키텍처 및 용량 최적화 포인트
 - 마스터 노드 자원 활용 (빌드 및 배포 중앙화)
   마스터 노드는 Taint 설정으로 인해 일반 서비스 파드가 뜨지 않아 평소에 유휴 자원이 많다.

   따라서 마스터 노드의 RAM(8GB~10GB)과 디스크(60GB+)를 넉넉히 할당한 뒤,
   마스터 노드에서 Docker 빌드, Private Registry 운영, DB 마이그레이션, kubectl 배포 명령을 중앙 집중식으로 처리한다.
   (이 방식을 쓰면 배포 빌드 시 실제 서비스가 돌아가는 Worker 1, Worker 2에 CPU/RAM 부하를 주지 않는다.)


 - 데이터 영구 보존 및 GC 작동 보장
   Registry 컨테이너의 내부 저장소를 호스트 폴더(-v)와 마운트하여
   재시작 시에도 이미지가 날아가지 않고 가비지 컬렉션(GC)이 실제 디스크 용량을 확보하도록 구성한다.


 - 2개 태그 로테이션(:latest, :rollback)
   태그가 무한정 쌓이는 것을 막기 위해 오직 2개의 태그만 교차 사용한다.

   이를 통해 별도의 무거운 .tar 백업 파일 없이도 즉각적인 롤백이 가능하며,
   언제나 직전 버전과 현재 버전 딱 2벌의 용량만 유지하고 나머지는 가비지 컬렉션으로 완전 삭제한다.
```

<br />
<br />
<br />
<br />

1. Master Node에 Private Registry 띄우기

```
RAM을 넉넉히(8GB~10GB) 할당한 "Master Node (예: 192.168.0.101)"에 Docker를 설치하고 경량 Registry 컨테이너를 실행한다.

Worker 노드들이 접근할 수 있도록 Master Node의 방화벽에서 5000번 포트를 반드시 허용해야 한다.

* 중요
  컨테이너가 재시작되어도 이미지가 날아가지 않고 가비지 컬렉션이 동작하도록
  반드시 데이터 저장 경로(/home/codej625/registry-data)도 함께 볼륨 마운트(-v)한다.
```

<br />

```zsh
# Master Node 방화벽에서 5000번 포트 허용 (사설망 내부 Worker 노드들의 이미지 Pull 허용)
sudo ufw allow from 192.168.0.0/24 to any port 5000 proto tcp
sudo ufw reload

# 설정 파일 및 데이터 저장용 디렉토리 생성
mkdir -p /home/codej625/yaml
mkdir -p /home/codej625/registry-data
```

<br />

`registry-config.yml`

```yaml
# 경로 -> /home/codej625/yaml/registry-config.yml

version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
```

<br />

```zsh
# 설정 파일 생성
cat <<EOF > /home/codej625/yaml/registry-config.yml
version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
EOF

# Registry 컨테이너 실행 (설정 파일 및 데이터 폴더 동시 마운트)
docker run -d \
  --name registry \
  --restart always \
  -p 5000:5000 \
  -v /home/codej625/yaml/registry-config.yml:/etc/docker/registry/config.yml \
  -v /home/codej625/registry-data:/var/lib/registry \
  registry:2
```

<br />
<br />
<br />

2. Docker insecure-registry 설정 (Master Node)

```
Private Registry는 HTTPS가 아닌 HTTP를 사용하므로,
빌드 및 Push를 수행할 Master Node의 Docker에 해당 주소를 insecure-registry로 허용해야 한다.

설정 후 Docker를 재시작해야 적용된다.
```

<br />

`/etc/docker/daemon.json`

```json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
```

```zsh
# 위의 192.168.0.101은 Registry가 떠 있는 Master Node의 고정 IP이다.
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
EOF

# 도커 재시작
sudo systemctl restart docker
```

<br />
<br />
<br />

3. k3s registries.yaml 설정 (Master Node 및 모든 Worker Node 공통)

```
k3s(containerd)가 HTTP Registry에서 이미지를 pull할 수 있도록
Master Node와 모든 Worker Node(Worker 1, Worker 2)에 registries.yaml을 작성한다.

파일이 없으면 직접 생성하고,
작성 후 반드시 재시작해야 적용된다.
```

<br />

`/etc/rancher/k3s/registries.yaml (Master Node, Worker Node 동일)`

```yaml
mirrors:
  "192.168.0.101:5000": # 아이피는 Registry가 설치된 Master Node의 IP를 사용
    endpoint:
      - "http://192.168.0.101:5000"
```

```zsh
# 디렉토리가 없으면 먼저 생성 후 파일 작성 (모든 노드에서 실행)
sudo mkdir -p /etc/rancher/k3s/
cat <<EOF | sudo tee /etc/rancher/k3s/registries.yaml
mirrors:
  "192.168.0.101:5000":
    endpoint:
      - "http://192.168.0.101:5000"
EOF

# Master Node에서 재시작
sudo systemctl restart k3s

# Worker Node(Worker 1, Worker 2)에서 재시작
sudo systemctl restart k3s-agent
```

<br />
<br />
<br />

4. Deployment yaml 준비 (Master Node)

```
기존 이미지 주소를 Master Node의 Private Registry 주소(192.168.0.101:5000)로 맞추고,
imagePullPolicy를 Always로 설정하여 배포 시 항상 최신 이미지를 가져오게 한다.

/home/codej625/yaml 경로에 next-deployment.yaml 과 nest-deployment.yaml 파일을 준비해 두고,
nest-secret.yaml 만 먼저 kubectl apply 해두면, 이후 ./deploy.sh 실행 시 이미지를 빌드/푸시한 뒤
Deployment까지 자동으로 클러스터에 적용(apply) 및 롤링 업데이트해 준다.
```

<br />

`next-deployment.yaml 및 nest-deployment.yaml 내 이미지 경로 확인`

```yaml
# /home/codej625/yaml/next-deployment.yaml 예시
containers:
  - name: next
    image: 192.168.0.101:5000/next:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always

# /home/codej625/yaml/nest-deployment.yaml 예시
containers:
  - name: nest
    image: 192.168.0.101:5000/nest:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always
```

<br />
<br />
<br />

5. GitHub SSH 키 설정 (Master Node)

```
배포 스크립트에서 git pull 시 매번 ID/PW를 입력하지 않도록
Master Node에 SSH 키를 생성하고 GitHub에 등록한다.
```

<br />

```zsh
# SSH 키 생성
ssh-keygen -t ed25519 -C "your-email@gmail.com" # 실제 사용하는 이메일 등록

# 공개키 확인 후 복사
cat ~/.ssh/id_ed25519.pub
```

```
# 깃 허브 접속
GitHub -> Settings -> SSH and GPG keys -> New SSH key -> 붙여넣기
```

```zsh
# remote URL을 SSH로 변경
cd /your/project/path
git remote set-url origin git@github.com:codej625/레포이름.git
```

<br />
<br />
<br />

6. 배포 스크립트 작성 (용량 극한 최적화 + DB 자동 마이그레이션 + 완벽한 롤백 보장)

```
* 스크립트 핵심 동작 원리
1. 태그 무한 증식 방지 및 롤백 보장 (:latest 와 :rollback 태그 로테이션)
   - 새로 빌드하기 전, 레지스트리에 있는 현재 정상 이미지(:latest)를 가져와 직전 백업 태그(:rollback)로 덮어씌워 푸시한다.
   - 최초 배포(1회차) 시에는 기존 이미지가 없으므로 이를 자동 감지하여 롤백 백업 단계를 안전하게 건너뛴다.
   - 이렇게 하면 별도의 무거운 .tar 백업 파일을 만들지 않아도, 배포 실패 시 즉시 :rollback 태그로 완벽한 롤백이 가능하다.
   - 동시에 2단계 이전의 구버전 이미지는 태그를 잃어버린 상태(Untagged)가 된다.

2. 무중단 DB 마이그레이션 자동 실행
   - 백엔드(NestJS) 이미지가 빌드된 직후, 쿠버네티스의 nest-secret에서 DATABASE_URL을 읽어와
     새로 빌드된 이미지의 drizzle 마이그레이션(npm run db:migrate:prod)을 배포 전에 먼저 적용한다.

3. 극한의 디스크 용량 확보
   - 배포 완료 후 Master Node의 Docker 이미지와 빌드 캐시를 전량 삭제(prune -a -f)한다.
   - 마지막으로 Registry 가비지 컬렉션(--delete-untagged)을 실행하여 태그를 잃은 구버전 이미지를 물리 디스크에서 완전히 지운다.
   - 결과적으로 레지스트리에는 언제나 현재 버전(:latest)과 직전 버전(:rollback) 딱 2벌의 용량만 유지된다.
```

<br />

`/home/codej625/deploy.sh`

```zsh
#!/usr/bin/env bash

set -euo pipefail

REGISTRY="192.168.0.101:5000"
NEXT_PUBLIC_API_URL="https://<사용하는 도메인>/api"
GIT_ROOT="/<프로젝트 경로>"
FRONTEND_PATH="$GIT_ROOT/front-end"
BACKEND_PATH="$GIT_ROOT/back-end"
YAML_DIR="/home/codej625/yaml"

NEXT_LATEST="$REGISTRY/next:latest"
NEXT_ROLLBACK="$REGISTRY/next:rollback"

NEST_LATEST="$REGISTRY/nest:latest"
NEST_ROLLBACK="$REGISTRY/nest:rollback"

HAS_NEXT_ROLLBACK=false
HAS_NEST_ROLLBACK=false

echo "=== 1. Registry 컨테이너 상태 확인 ==="
if [ ! "$(docker ps -q -f name=^registry$)" ]; then
    if [ "$(docker ps -aq -f status=exited -f name=^registry$)" ]; then
        docker start registry
    else
        docker run -d --name registry --restart always -p 5000:5000 \
          -v "$YAML_DIR/registry-config.yml":/etc/docker/registry/config.yml \
          -v /home/codej625/registry-data:/var/lib/registry \
          registry:2
    fi
fi

echo "=== 2. 기존 정상 이미지를 :rollback 태그로 백업 (태그 로테이션) ==="
# 기존 latest 이미지가 레지스트리에 있다면 pull 받아서 rollback 태그로 덮어써서 push 해둠
if docker pull "$NEXT_LATEST" >/dev/null 2>&1; then
    docker tag "$NEXT_LATEST" "$NEXT_ROLLBACK"
    docker push "$NEXT_ROLLBACK"
    HAS_NEXT_ROLLBACK=true
else
    echo "[안내] 기존 next:latest 이미지가 없습니다 (최초 배포)."
fi

if docker pull "$NEST_LATEST" >/dev/null 2>&1; then
    docker tag "$NEST_LATEST" "$NEST_ROLLBACK"
    docker push "$NEST_ROLLBACK"
    HAS_NEST_ROLLBACK=true
else
    echo "[안내] 기존 nest:latest 이미지가 없습니다 (최초 배포)."
fi

echo "=== 3. Git Pull ==="
cd "$GIT_ROOT" || exit 1
git pull

echo "=== 4. Docker Build & Push (순차 빌드) ==="
docker build \
  --build-arg NEXT_PUBLIC_API_URL="$NEXT_PUBLIC_API_URL" \
  -t "$NEXT_LATEST" \
  "$FRONTEND_PATH"
docker push "$NEXT_LATEST"

docker build -t "$NEST_LATEST" "$BACKEND_PATH"
docker push "$NEST_LATEST"

echo "=== 5. DB 마이그레이션 적용 (Drizzle) ==="
# k8s의 nest-secret에서 DATABASE_URL과 DATABASE_SSL을 읽어와 파드 배포 전에 DB 스키마를 최신화함
DB_URL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_URL}' | base64 -d)
DB_SSL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_SSL}' | base64 -d 2>/dev/null || echo "false")

docker run --rm \
  -e DATABASE_URL="$DB_URL" \
  -e DATABASE_SSL="${DB_SSL:-false}" \
  "$NEST_LATEST" \
  npm run db:migrate:prod

echo "=== 6. Kubernetes 배포 (NestJS 백엔드 먼저 배포) ==="
kubectl apply -f "$YAML_DIR/nest-deployment.yaml"
kubectl set image deployment/nest nest="$NEST_LATEST"
kubectl rollout restart deployment/nest

if kubectl rollout status deployment/nest --timeout=360s; then
  echo "[성공] nest 배포 완료."
else
  echo "[실패] nest 배포 실패. 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  exit 1
fi

echo "=== 7. Kubernetes 배포 (Next.js 프론트엔드 배포) ==="
kubectl apply -f "$YAML_DIR/next-deployment.yaml"
kubectl set image deployment/next next="$NEXT_LATEST"
kubectl rollout restart deployment/next

if kubectl rollout status deployment/next --timeout=360s; then
  echo "[성공] next 배포 완료."
else
  echo "[실패] next 배포 실패. 전체 서비스를 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    kubectl set image deployment/next next="$NEXT_ROLLBACK"
    kubectl rollout status deployment/next --timeout=180s || true

    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  exit 1
fi

echo "=== 8. Master Node 로컬 Docker 이미지 및 빌드 캐시 전체 삭제 (용량 확보) ==="
docker system prune -a -f
docker builder prune -a -f

echo "=== 9. Registry 가비지 컬렉션 실행 (태그 잃은 구버전 영구 삭제) ==="
docker exec registry bin/registry garbage-collect \
  --delete-untagged \
  /etc/docker/registry/config.yml
docker restart registry

echo "=== 10. 배포 및 용량 최적화 완료 ==="
```

```zsh
# 실행 권한 부여
chmod +x /home/codej625/deploy.sh
```

<br />
<br />
<br />

7. 배포 및 확인

```
스크립트를 실행하고, Pod 상태와 이미지 태그를 확인하여
배포가 정상적으로 이루어졌는지 검증한다.
```

<br />

```zsh
# 1) 배포 실행 (Master Node의 deploy.sh가 있는 디렉터리에서)
./deploy.sh

# 2) Pod 상태 및 노드 확인 (Worker 1, Worker 2에 정상적으로 떴는지 확인)
kubectl get pods -o wide

# 3) 이미지 태그 및 새로 pull 받은 이미지 해시(Image ID) 확인
kubectl describe pod <pod-name> | grep -E "Image:|Image ID:"

# 4) 실제 확보된 디스크 여유 공간 확인
df -h
```

<br />
<br />
<br />

8. 주의사항 (kubectl set image 및 Worker 노드 용량 관리)

```
kubectl set image 명령어는 yaml 파일을 참조하지 않고
k3s 내부 상태를 직접 변경한다.

따라서 아래 두 값이 반드시 일치해야 한다.
```

<br />

| 항목 | 확인 방법 |
| :--- | :--- |
| **컨테이너 이름** (`next`, `nest`) | deployment yaml의 `containers.name` 값과 일치해야 함 |
| **Deployment 이름** (`next`, `nest`) | `kubectl get deployment` NAME 컬럼 값과 일치해야 함 |
| **Worker Node 재시작** | `sudo systemctl restart k3s-agent` 사용 |
| **Master Node 재시작** | `sudo systemctl restart k3s` 사용 (k3s-agent 아님) |
| **Worker Node의 미사용 이미지 청소** | 실제 파드가 뜨는 Worker 1, Worker 2에도 구버전 이미지가 캐시로 남으므로, 주기적으로 워커 노드에서 `sudo k3s crictl rmi --prune` 명령을 실행하거나 크론탭(crontab)에 주 1회 등록해 두네! 마크다운 코드 블록이 중간에 닫히지 않도록 바깥쪽을 `~~~~markdown`으로 감싸서 **우측 상단 복사 버튼 한 번으로 문서 1 전체를 통째로 복사**하실 수 있게 준비했습니다.

(문서 2는 직전에 올려주신 내용이 이미 완벽하므로 그대로 두시면 됩니다!)

### 📄 문서 1 전체 복사본 (`Private Registry를 사용하는 반자동 배포`)

~~~~markdown
# Private Registry를 사용하는 반자동 배포 (용량 최적화 및 롤백 기능)

<br />
<br />

* 왜 Private Registry를 사용할까?

---

```
외부 Registry(Docker Hub, ghcr.io 등) 사용이 제한되거나 유료 플랜이 부담스러운 환경에서,
Master Node에 직접 Private Registry를 띄워 Worker Node들이 이미지를 내부 네트워크로 pull 할 수 있게 한다.

* 핵심 아키텍처 및 용량 최적화 포인트
 - 마스터 노드 자원 활용 (빌드 및 배포 중앙화)
   마스터 노드는 Taint 설정으로 인해 일반 서비스 파드가 뜨지 않아 평소에 유휴 자원이 많다.

   따라서 마스터 노드의 RAM(8GB~10GB)과 디스크(60GB+)를 넉넉히 할당한 뒤,
   마스터 노드에서 Docker 빌드, Private Registry 운영, DB 마이그레이션, kubectl 배포 명령을 중앙 집중식으로 처리한다.
   (이 방식을 쓰면 배포 빌드 시 실제 서비스가 돌아가는 Worker 1, Worker 2에 CPU/RAM 부하를 주지 않는다.)


 - 데이터 영구 보존 및 GC 작동 보장
   Registry 컨테이너의 내부 저장소를 호스트 폴더(-v)와 마운트하여
   재시작 시에도 이미지가 날아가지 않고 가비지 컬렉션(GC)이 실제 디스크 용량을 확보하도록 구성한다.


 - 2개 태그 로테이션(:latest, :rollback)
   태그가 무한정 쌓이는 것을 막기 위해 오직 2개의 태그만 교차 사용한다.

   이를 통해 별도의 무거운 .tar 백업 파일 없이도 즉각적인 롤백이 가능하며,
   언제나 직전 버전과 현재 버전 딱 2벌의 용량만 유지하고 나머지는 가비지 컬렉션으로 완전 삭제한다.
```

<br />
<br />
<br />
<br />

1. Master Node에 Private Registry 띄우기

```
RAM을 넉넉히(8GB~10GB) 할당한 "Master Node (예: 192.168.0.101)"에 Docker를 설치하고 경량 Registry 컨테이너를 실행한다.

Worker 노드들이 접근할 수 있도록 Master Node의 방화벽에서 5000번 포트를 반드시 허용해야 한다.

* 중요
  컨테이너가 재시작되어도 이미지가 날아가지 않고 가비지 컬렉션이 동작하도록
  반드시 데이터 저장 경로(/home/codej625/registry-data)도 함께 볼륨 마운트(-v)한다.
```

<br />

```zsh
# Master Node 방화벽에서 5000번 포트 허용 (사설망 내부 Worker 노드들의 이미지 Pull 허용)
sudo ufw allow from 192.168.0.0/24 to any port 5000 proto tcp
sudo ufw reload

# 설정 파일 및 데이터 저장용 디렉토리 생성
mkdir -p /home/codej625/yaml
mkdir -p /home/codej625/registry-data
```

<br />

`registry-config.yml`

```yaml
# 경로 -> /home/codej625/yaml/registry-config.yml

version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
```

<br />

```zsh
# 설정 파일 생성
cat <<EOF > /home/codej625/yaml/registry-config.yml
version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
EOF

# Registry 컨테이너 실행 (설정 파일 및 데이터 폴더 동시 마운트)
docker run -d \
  --name registry \
  --restart always \
  -p 5000:5000 \
  -v /home/codej625/yaml/registry-config.yml:/etc/docker/registry/config.yml \
  -v /home/codej625/registry-data:/var/lib/registry \
  registry:2
```

<br />
<br />
<br />

2. Docker insecure-registry 설정 (Master Node)

```
Private Registry는 HTTPS가 아닌 HTTP를 사용하므로,
빌드 및 Push를 수행할 Master Node의 Docker에 해당 주소를 insecure-registry로 허용해야 한다.

설정 후 Docker를 재시작해야 적용된다.
```

<br />

`/etc/docker/daemon.json`

```json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
```

```zsh
# 위의 192.168.0.101은 Registry가 떠 있는 Master Node의 고정 IP이다.
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
EOF

# 도커 재시작
sudo systemctl restart docker
```

<br />
<br />
<br />

3. k3s registries.yaml 설정 (Master Node 및 모든 Worker Node 공통)

```
k3s(containerd)가 HTTP Registry에서 이미지를 pull할 수 있도록
Master Node와 모든 Worker Node(Worker 1, Worker 2)에 registries.yaml을 작성한다.

파일이 없으면 직접 생성하고,
작성 후 반드시 재시작해야 적용된다.
```

<br />

`/etc/rancher/k3s/registries.yaml (Master Node, Worker Node 동일)`

```yaml
mirrors:
  "192.168.0.101:5000": # 아이피는 Registry가 설치된 Master Node의 IP를 사용
    endpoint:
      - "http://192.168.0.101:5000"
```

```zsh
# 디렉토리가 없으면 먼저 생성 후 파일 작성 (모든 노드에서 실행)
sudo mkdir -p /etc/rancher/k3s/
cat <<EOF | sudo tee /etc/rancher/k3s/registries.yaml
mirrors:
  "192.168.0.101:5000":
    endpoint:
      - "http://192.168.0.101:5000"
EOF

# Master Node에서 재시작
sudo systemctl restart k3s

# Worker Node(Worker 1, Worker 2)에서 재시작
sudo systemctl restart k3s-agent
```

<br />
<br />
<br />

4. Deployment yaml 준비 (Master Node)

```
기존 이미지 주소를 Master Node의 Private Registry 주소(192.168.0.101:5000)로 맞추고,
imagePullPolicy를 Always로 설정하여 배포 시 항상 최신 이미지를 가져오게 한다.

/home/codej625/yaml 경로에 next-deployment.yaml 과 nest-deployment.yaml 파일을 준비해 두고,
nest-secret.yaml 만 먼저 kubectl apply 해두면, 이후 ./deploy.sh 실행 시 이미지를 빌드/푸시한 뒤
Deployment까지 자동으로 클러스터에 적용(apply) 및 롤링 업데이트해 준다.
```

<br />

`next-deployment.yaml 및 nest-deployment.yaml 내 이미지 경로 확인`

```yaml
# /home/codej625/yaml/next-deployment.yaml 예시
containers:
  - name: next
    image: 192.168.0.101:5000/next:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always

# /home/codej625/yaml/nest-deployment.yaml 예시
containers:
  - name: nest
    image: 192.168.0.101:5000/nest:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always
```

<br />
<br />
<br />

5. GitHub SSH 키 설정 (Master Node)

```
배포 스크립트에서 git pull 시 매번 ID/PW를 입력하지 않도록
Master Node에 SSH 키를 생성하고 GitHub에 등록한다.
```

<br />

```zsh
# SSH 키 생성
ssh-keygen -t ed25519 -C "your-email@gmail.com" # 실제 사용하는 이메일 등록

# 공개키 확인 후 복사
cat ~/.ssh/id_ed25519.pub
```

```
# 깃 허브 접속
GitHub -> Settings -> SSH and GPG keys -> New SSH key -> 붙여넣기
```

```zsh
# remote URL을 SSH로 변경
cd /your/project/path
git remote set-url origin git@github.com:codej625/레포이름.git
```

<br />
<br />
<br />

6. 배포 스크립트 작성 (용량 극한 최적화 + DB 자동 마이그레이션 + 완벽한 롤백 보장)

```
* 스크립트 핵심 동작 원리
1. 태그 무한 증식 방지 및 롤백 보장 (:latest 와 :rollback 태그 로테이션)
   - 새로 빌드하기 전, 레지스트리에 있는 현재 정상 이미지(:latest)를 가져와 직전 백업 태그(:rollback)로 덮어씌워 푸시한다.
   - 최초 배포(1회차) 시에는 기존 이미지가 없으므로 이를 자동 감지하여 롤백 백업 단계를 안전하게 건너뛴다.
   - 이렇게 하면 별도의 무거운 .tar 백업 파일을 만들지 않아도, 배포 실패 시 즉시 :rollback 태그로 완벽한 롤백이 가능하다.
   - 동시에 2단계 이전의 구버전 이미지는 태그를 잃어버린 상태(Untagged)가 된다.

2. 무중단 DB 마이그레이션 자동 실행
   - 백엔드(NestJS) 이미지가 빌드된 직후, 쿠버네티스의 nest-secret에서 DATABASE_URL을 읽어와
     새로 빌드된 이미지의 drizzle 마이그레이션(npm run db:migrate:prod)을 배포 전에 먼저 적용한다.

3. 극한의 디스크 용량 확보
   - 배포 완료 후 Master Node의 Docker 이미지와 빌드 캐시를 전량 삭제(prune -a -f)한다.
   - 마지막으로 Registry 가비지 컬렉션(--delete-untagged)을 실행하여 태그를 잃은 구버전 이미지를 물리 디스크에서 완전히 지운다.
   - 결과적으로 레지스트리에는 언제나 현재 버전(:latest)과 직전 버전(:rollback) 딱 2벌의 용량만 유지된다.
```

<br />

`/home/codej625/deploy.sh`

```zsh
#!/usr/bin/env bash

set -euo pipefail

REGISTRY="192.168.0.101:5000"
NEXT_PUBLIC_API_URL="https://<사용하는 도메인>/api"
GIT_ROOT="/<프로젝트 경로>"
FRONTEND_PATH="$GIT_ROOT/front-end"
BACKEND_PATH="$GIT_ROOT/back-end"
YAML_DIR="/home/codej625/yaml"

NEXT_LATEST="$REGISTRY/next:latest"
NEXT_ROLLBACK="$REGISTRY/next:rollback"

NEST_LATEST="$REGISTRY/nest:latest"
NEST_ROLLBACK="$REGISTRY/nest:rollback"

HAS_NEXT_ROLLBACK=false
HAS_NEST_ROLLBACK=false

echo "=== 1. Registry 컨테이너 상태 확인 ==="
if [ ! "$(docker ps -q -f name=^registry$)" ]; then
    if [ "$(docker ps -aq -f status=exited -f name=^registry$)" ]; then
        docker start registry
    else
        docker run -d --name registry --restart always -p 5000:5000 \
          -v "$YAML_DIR/registry-config.yml":/etc/docker/registry/config.yml \
          -v /home/codej625/registry-data:/var/lib/registry \
          registry:2
    fi
fi

echo "=== 2. 기존 정상 이미지를 :rollback 태그로 백업 (태그 로테이션) ==="
# 기존 latest 이미지가 레지스트리에 있다면 pull 받아서 rollback 태그로 덮어써서 push 해둠
if docker pull "$NEXT_LATEST" >/dev/null 2>&1; then
    docker tag "$NEXT_LATEST" "$NEXT_ROLLBACK"
    docker push "$NEXT_ROLLBACK"
    HAS_NEXT_ROLLBACK=true
else
    echo "[안내] 기존 next:latest 이미지가 없습니다 (최초 배포)."
fi

if docker pull "$NEST_LATEST" >/dev/null 2>&1; then
    docker tag "$NEST_LATEST" "$NEST_ROLLBACK"
    docker push "$NEST_ROLLBACK"
    HAS_NEST_ROLLBACK=true
else
    echo "[안내] 기존 nest:latest 이미지가 없습니다 (최초 배포)."
fi

echo "=== 3. Git Pull ==="
cd "$GIT_ROOT" || exit 1
git pull

echo "=== 4. Docker Build & Push (순차 빌드) ==="
docker build \
  --build-arg NEXT_PUBLIC_API_URL="$NEXT_PUBLIC_API_URL" \
  -t "$NEXT_LATEST" \
  "$FRONTEND_PATH"
docker push "$NEXT_LATEST"

docker build -t "$NEST_LATEST" "$BACKEND_PATH"
docker push "$NEST_LATEST"

echo "=== 5. DB 마이그레이션 적용 (Drizzle) ==="
# k8s의 nest-secret에서 DATABASE_URL과 DATABASE_SSL을 읽어와 파드 배포 전에 DB 스키마를 최신화함
DB_URL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_URL}' | base64 -d)
DB_SSL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_SSL}' | base64 -d 2>/dev/null || echo "false")

docker run --rm \
  -e DATABASE_URL="$DB_URL" \
  -e DATABASE_SSL="${DB_SSL:-false}" \
  "$NEST_LATEST" \
  npm run db:migrate:prod

echo "=== 6. Kubernetes 배포 (NestJS 백엔드 먼저 배포) ==="
kubectl apply -f "$YAML_DIR/nest-deployment.yaml"
kubectl set image deployment/nest nest="$NEST_LATEST"
kubectl rollout restart deployment/nest

if kubectl rollout status deployment/nest --timeout=360s; then
  echo "[성공] nest 배포 완료."
else
  echo "[실패] nest 배포 실패. 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  exit 1
fi

echo "=== 7. Kubernetes 배포 (Next.js 프론트엔드 배포) ==="
kubectl apply -f "$YAML_DIR/next-deployment.yaml"
kubectl set image deployment/next next="$NEXT_LATEST"
kubectl rollout restart deployment/next

if kubectl rollout status deployment/next --timeout=360s; then
  echo "[성공] next 배포 완료."
else
  echo "[실패] next 배포 실패. 전체 서비스를 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    kubectl set image deployment/next next="$NEXT_ROLLBACK"
    kubectl rollout status deployment/next --timeout=180s || true

    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  exit 1
fi

echo "=== 8. Master Node 로컬 Docker 이미지 및 빌드 캐시 전체 삭제 (용량 확보) ==="
docker system prune -a -f
docker builder prune -a -f

echo "=== 9. Registry 가비지 컬렉션 실행 (태그 잃은 구버전 영구 삭제) ==="
docker exec registry bin/registry garbage-collect \
  --delete-untagged \
  /etc/docker/registry/config.yml
docker restart registry

echo "=== 10. 배포 및 용량 최적화 완료 ==="
```

```zsh
# 실행 권한 부여
chmod +x /home/codej625/deploy.sh
```

<br />
<br />
<br />

7. 배포 및 확인

```
스크립트를 실행하고, Pod 상태와 이미지 태그를 확인하여
배포가 정상적으로 이루어졌는지 검증한다.
```

<br />

```zsh
# 1) 배포 실행 (Master Node의 deploy.sh가 있는 디렉터리에서)
./deploy.sh

# 2) Pod 상태 및 노드 확인 (Worker 1, Worker 2에 정상적으로 떴는지 확인)
kubectl get pods -o wide

# 3) 이미지 태그 및 새로 pull 받은 이미지 해시(Image ID) 확인
kubectl describe pod <pod-name> | grep -E "Image:|Image ID:"

# 4) 실제 확보된 디스크 여유 공간 확인
df -h
```

<br />
<br />
<br />

8. 주의사항 (kubectl set image 및 Worker 노드 용량 관리)

```
kubectl set image 명령어는 yaml 파일을 참조하지 않고
k3s 내부 상태를 직접 변경한다.

따라서 아래 두 값이 반드시 일치해야 한다.
```

<br />

| 항목 | 확인 방법 |
| :--- | :--- |
| **컨테이너 이름** (`next`, `nest`) | deployment yaml의 `containers.name` 값과 일치해야 함 |
| **Deployment 이름** (`next`, `nest`) | `kubectl get deployment` NAME 컬럼 값과 일치해야 함 |
| **Worker Node 재시작** | `sudo systemctl restart k3s-agent` 사용 |
| **Master Node 재시작** | `sudo systemctl restart k3s` 사용 (k3s-agent 아님) |
| **Worker Node의 미사용 이미지 청소** | 실제 파드가 뜨는 Worker 1, Worker 2에도 구버전 이미지가 캐시로 남으므로, 주기적으로 워커 노드에서 `sudo k3s crictl rmi --prune` 명령을 실행하거나 크론탭(crontab)에 주 1회 등록해 두면 워커 노드의 디스크도 영구적으로 깨끗하게 유지됨 |네, 내부 코드 블록(`` ``` ``) 때문에 마크다운 박스가 중간에 끊기지 않도록 **가장 바깥쪽을 물결 4개(`~~~~markdown`)로 감싸서 한 번에 전체 복사**하실 수 있게 출력해 드립니다!

우측 상단의 복사(Copy) 버튼을 눌러 **문서 1**에 그대로 덮어쓰시면 됩니다. (문서 2는 이미 완벽하게 수정되어 있으니 그대로 두시면 됩니다.)

~~~~markdown
# Private Registry를 사용하는 반자동 배포 (용량 최적화 및 롤백 기능)

<br />
<br />

* 왜 Private Registry를 사용할까?

---

```
외부 Registry(Docker Hub, ghcr.io 등) 사용이 제한되거나 유료 플랜이 부담스러운 환경에서,
Master Node에 직접 Private Registry를 띄워 Worker Node들이 이미지를 내부 네트워크로 pull 할 수 있게 한다.

* 핵심 아키텍처 및 용량 최적화 포인트
 - 마스터 노드 자원 활용 (빌드 및 배포 중앙화)
   마스터 노드는 Taint 설정으로 인해 일반 서비스 파드가 뜨지 않아 평소에 유휴 자원이 많다.

   따라서 마스터 노드의 RAM(8GB~10GB)과 디스크(60GB+)를 넉넉히 할당한 뒤,
   마스터 노드에서 Docker 빌드, Private Registry 운영, DB 마이그레이션, kubectl 배포 명령을 중앙 집중식으로 처리한다.
   (이 방식을 쓰면 배포 빌드 시 실제 서비스가 돌아가는 Worker 1, Worker 2에 CPU/RAM 부하를 주지 않는다.)


 - 데이터 영구 보존 및 GC 작동 보장
   Registry 컨테이너의 내부 저장소를 호스트 폴더(-v)와 마운트하여
   재시작 시에도 이미지가 날아가지 않고 가비지 컬렉션(GC)이 실제 디스크 용량을 확보하도록 구성한다.


 - 2개 태그 로테이션(:latest, :rollback)
   태그가 무한정 쌓이는 것을 막기 위해 오직 2개의 태그만 교차 사용한다.

   이를 통해 별도의 무거운 .tar 백업 파일 없이도 즉각적인 롤백이 가능하며,
   언제나 직전 버전과 현재 버전 딱 2벌의 용량만 유지하고 나머지는 가비지 컬렉션으로 완전 삭제한다.
```

<br />
<br />
<br />
<br />

1. Master Node에 Private Registry 띄우기

```
RAM을 넉넉히(8GB~10GB) 할당한 "Master Node (예: 192.168.0.101)"에 Docker를 설치하고 경량 Registry 컨테이너를 실행한다.

Worker 노드들이 접근할 수 있도록 Master Node의 방화벽에서 5000번 포트를 반드시 허용해야 한다.

* 중요
  컨테이너가 재시작되어도 이미지가 날아가지 않고 가비지 컬렉션이 동작하도록
  반드시 데이터 저장 경로(/home/codej625/registry-data)도 함께 볼륨 마운트(-v)한다.
```

<br />

```zsh
# Master Node 방화벽에서 5000번 포트 허용 (사설망 내부 Worker 노드들의 이미지 Pull 허용)
sudo ufw allow from 192.168.0.0/24 to any port 5000 proto tcp
sudo ufw reload

# 설정 파일 및 데이터 저장용 디렉토리 생성
mkdir -p /home/codej625/yaml
mkdir -p /home/codej625/registry-data
```

<br />

`registry-config.yml`

```yaml
# 경로 -> /home/codej625/yaml/registry-config.yml

version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
```

<br />

```zsh
# 설정 파일 생성
cat <<EOF > /home/codej625/yaml/registry-config.yml
version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
EOF

# Registry 컨테이너 실행 (설정 파일 및 데이터 폴더 동시 마운트)
docker run -d \
  --name registry \
  --restart always \
  -p 5000:5000 \
  -v /home/codej625/yaml/registry-config.yml:/etc/docker/registry/config.yml \
  -v /home/codej625/registry-data:/var/lib/registry \
  registry:2
```

<br />
<br />
<br />

2. Docker insecure-registry 설정 (Master Node)

```
Private Registry는 HTTPS가 아닌 HTTP를 사용하므로,
빌드 및 Push를 수행할 Master Node의 Docker에 해당 주소를 insecure-registry로 허용해야 한다.

설정 후 Docker를 재시작해야 적용된다.
```

<br />

`/etc/docker/daemon.json`

```json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
```

```zsh
# 위의 192.168.0.101은 Registry가 떠 있는 Master Node의 고정 IP이다.
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
EOF

# 도커 재시작
sudo systemctl restart docker
```

<br />
<br />
<br />

3. k3s registries.yaml 설정 (Master Node 및 모든 Worker Node 공통)

```
k3s(containerd)가 HTTP Registry에서 이미지를 pull할 수 있도록
Master Node와 모든 Worker Node(Worker 1, Worker 2)에 registries.yaml을 작성한다.

파일이 없으면 직접 생성하고,
작성 후 반드시 재시작해야 적용된다.
```

<br />

`/etc/rancher/k3s/registries.yaml (Master Node, Worker Node 동일)`

```yaml
mirrors:
  "192.168.0.101:5000": # 아이피는 Registry가 설치된 Master Node의 IP를 사용
    endpoint:
      - "http://192.168.0.101:5000"
```

```zsh
# 디렉토리가 없으면 먼저 생성 후 파일 작성 (모든 노드에서 실행)
sudo mkdir -p /etc/rancher/k3s/
cat <<EOF | sudo tee /etc/rancher/k3s/registries.yaml
mirrors:
  "192.168.0.101:5000":
    endpoint:
      - "http://192.168.0.101:5000"
EOF

# Master Node에서 재시작
sudo systemctl restart k3s

# Worker Node(Worker 1, Worker 2)에서 재시작
sudo systemctl restart k3s-agent
```

<br />
<br />
<br />

4. Deployment yaml 준비 (Master Node)

```
기존 이미지 주소를 Master Node의 Private Registry 주소(192.168.0.101:5000)로 맞추고,
imagePullPolicy를 Always로 설정하여 배포 시 항상 최신 이미지를 가져오게 한다.

/home/codej625/yaml 경로에 next-deployment.yaml 과 nest-deployment.yaml 파일을 준비해 두고,
nest-secret.yaml 만 먼저 kubectl apply 해두면, 이후 ./deploy.sh 실행 시 이미지를 빌드/푸시한 뒤
Deployment까지 자동으로 클러스터에 적용(apply) 및 롤링 업데이트해 준다.
```

<br />

`next-deployment.yaml 및 nest-deployment.yaml 내 이미지 경로 확인`

```yaml
# /home/codej625/yaml/next-deployment.yaml 예시
containers:
  - name: next
    image: 192.168.0.101:5000/next:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always

# /home/codej625/yaml/nest-deployment.yaml 예시
containers:
  - name: nest
    image: 192.168.0.101:5000/nest:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always
```

<br />
<br />
<br />

5. GitHub SSH 키 설정 (Master Node)

```
배포 스크립트에서 git pull 시 매번 ID/PW를 입력하지 않도록
Master Node에 SSH 키를 생성하고 GitHub에 등록한다.
```

<br />

```zsh
# SSH 키 생성
ssh-keygen -t ed25519 -C "your-email@gmail.com" # 실제 사용하는 이메일 등록

# 공개키 확인 후 복사
cat ~/.ssh/id_ed25519.pub
```

```
# 깃 허브 접속
GitHub -> Settings -> SSH and GPG keys -> New SSH key -> 붙여넣기
```

```zsh
# remote URL을 SSH로 변경
cd /your/project/path
git remote set-url origin git@github.com:codej625/레포이름.git
```

<br />
<br />
<br />

6. 배포 스크립트 작성 (용량 극한 최적화 + DB 자동 마이그레이션 + 완벽한 롤백 보장)

```
* 스크립트 핵심 동작 원리
1. 태그 무한 증식 방지 및 롤백 보장 (:latest 와 :rollback 태그 로테이션)
   - 새로 빌드하기 전, 레지스트리에 있는 현재 정상 이미지(:latest)를 가져와 직전 백업 태그(:rollback)로 덮어씌워 푸시한다.
   - 최초 배포(1회차) 시에는 기존 이미지가 없으므로 이를 자동 감지하여 롤백 백업 단계를 안전하게 건너뛴다.
   - 이렇게 하면 별도의 무거운 .tar 백업 파일을 만들지 않아도, 배포 실패 시 즉시 :rollback 태그로 완벽한 롤백이 가능하다.
   - 동시에 2단계 이전의 구버전 이미지는 태그를 잃어버린 상태(Untagged)가 된다.

2. 무중단 DB 마이그레이션 자동 실행
   - 백엔드(NestJS) 이미지가 빌드된 직후, 쿠버네티스의 nest-secret에서 DATABASE_URL을 읽어와
     새로 빌드된 이미지의 drizzle 마이그레이션(npm run db:migrate:prod)을 배포 전에 먼저 적용한다.

3. 극한의 디스크 용량 확보
   - 배포 완료 후 Master Node의 Docker 이미지와 빌드 캐시를 전량 삭제(prune -a -f)한다.
   - 마지막으로 Registry 가비지 컬렉션(--delete-untagged)을 실행하여 태그를 잃은 구버전 이미지를 물리 디스크에서 완전히 지운다.
   - 결과적으로 레지스트리에는 언제나 현재 버전(:latest)과 직전 버전(:rollback) 딱 2벌의 용량만 유지된다.
```

<br />

`/home/codej625/deploy.sh`

```zsh
#!/usr/bin/env bash

set -euo pipefail

REGISTRY="192.168.0.101:5000"
NEXT_PUBLIC_API_URL="https://<사용하는 도메인>/api"
GIT_ROOT="/<프로젝트 경로>"
FRONTEND_PATH="$GIT_ROOT/front-end"
BACKEND_PATH="$GIT_ROOT/back-end"
YAML_DIR="/home/codej625/yaml"

NEXT_LATEST="$REGISTRY/next:latest"
NEXT_ROLLBACK="$REGISTRY/next:rollback"

NEST_LATEST="$REGISTRY/nest:latest"
NEST_ROLLBACK="$REGISTRY/nest:rollback"

HAS_NEXT_ROLLBACK=false
HAS_NEST_ROLLBACK=false

echo "=== 1. Registry 컨테이너 상태 확인 ==="
if [ ! "$(docker ps -q -f name=^registry$)" ]; then
    if [ "$(docker ps -aq -f status=exited -f name=^registry$)" ]; then
        docker start registry
    else
        docker run -d --name registry --restart always -p 5000:5000 \
          -v "$YAML_DIR/registry-config.yml":/etc/docker/registry/config.yml \
          -v /home/codej625/registry-data:/var/lib/registry \
          registry:2
    fi
fi

echo "=== 2. 기존 정상 이미지를 :rollback 태그로 백업 (태그 로테이션) ==="
# 기존 latest 이미지가 레지스트리에 있다면 pull 받아서 rollback 태그로 덮어써서 push 해둠
if docker pull "$NEXT_LATEST" >/dev/null 2>&1; then
    docker tag "$NEXT_LATEST" "$NEXT_ROLLBACK"
    docker push "$NEXT_ROLLBACK"
    HAS_NEXT_ROLLBACK=true
else
    echo "[안내] 기존 next:latest 이미지가 없습니다 (최초 배포)."
fi

if docker pull "$NEST_LATEST" >/dev/null 2>&1; then
    docker tag "$NEST_LATEST" "$NEST_ROLLBACK"
    docker push "$NEST_ROLLBACK"
    HAS_NEST_ROLLBACK=true
else
    echo "[안내] 기존 nest:latest 이미지가 없습니다 (최초 배포)."
fi

echo "=== 3. Git Pull ==="
cd "$GIT_ROOT" || exit 1
git pull

echo "=== 4. Docker Build & Push (순차 빌드) ==="
docker build \
  --build-arg NEXT_PUBLIC_API_URL="$NEXT_PUBLIC_API_URL" \
  -t "$NEXT_LATEST" \
  "$FRONTEND_PATH"
docker push "$NEXT_LATEST"

docker build -t "$NEST_LATEST" "$BACKEND_PATH"
docker push "$NEST_LATEST"

echo "=== 5. DB 마이그레이션 적용 (Drizzle) ==="
# k8s의 nest-secret에서 DATABASE_URL과 DATABASE_SSL을 읽어와 파드 배포 전에 DB 스키마를 최신화함
DB_URL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_URL}' | base64 -d)
DB_SSL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_SSL}' | base64 -d 2>/dev/null || echo "false")

docker run --rm \
  -e DATABASE_URL="$DB_URL" \
  -e DATABASE_SSL="${DB_SSL:-false}" \
  "$NEST_LATEST" \
  npm run db:migrate:prod

echo "=== 6. Kubernetes 배포 (NestJS 백엔드 먼저 배포) ==="
kubectl apply -f "$YAML_DIR/nest-deployment.yaml"
kubectl set image deployment/nest nest="$NEST_LATEST"
kubectl rollout restart deployment/nest

if kubectl rollout status deployment/nest --timeout=360s; then
  echo "[성공] nest 배포 완료."
else
  echo "[실패] nest 배포 실패. 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  exit 1
fi

echo "=== 7. Kubernetes 배포 (Next.js 프론트엔드 배포) ==="
kubectl apply -f "$YAML_DIR/next-deployment.yaml"
kubectl set image deployment/next next="$NEXT_LATEST"
kubectl rollout restart deployment/next

if kubectl rollout status deployment/next --timeout=360s; then
  echo "[성공] next 배포 완료."
else
  echo "[실패] next 배포 실패. 전체 서비스를 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    kubectl set image deployment/next next="$NEXT_ROLLBACK"
    kubectl rollout status deployment/next --timeout=180s || true

    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  exit 1
fi

echo "=== 8. Master Node 로컬 Docker 이미지 및 빌드 캐시 전체 삭제 (용량 확보) ==="
docker system prune -a -f
docker builder prune -a -f

echo "=== 9. Registry 가비지 컬렉션 실행 (태그 잃은 구버전 영구 삭제) ==="
docker exec registry bin/registry garbage-collect \
  --delete-untagged \
  /etc/docker/registry/config.yml
docker restart registry

echo "=== 10. 배포 및 용량 최적화 완료 ==="
```

```zsh
# 실행 권한 부여
chmod +x /home/codej625/deploy.sh
```

<br />
<br />
<br />

7. 배포 및 확인

```
스크립트를 실행하고, Pod 상태와 이미지 태그를 확인하여
배포가 정상적으로 이루어졌는지 검증한다.
```

<br />

```zsh
# 1) 배포 실행 (Master Node의 deploy.sh가 있는 디렉터리에서)
./deploy.sh

# 2) Pod 상태 및 노드 확인 (Worker 1, Worker 2에 정상적으로 떴는지 확인)
kubectl get pods -o wide

# 3) 이미지 태그 및 새로 pull 받은 이미지 해시(Image ID) 확인
kubectl describe pod <pod-name> | grep -E "Image:|Image ID:"

# 4) 실제 확보된 디스크 여유 공간 확인
df -h
```

<br />
<br />
<br />

8. 주의사항 (kubectl set image 및 Worker 노드 용량 관리)

```
kubectl set image 명령어는 yaml 파일을 참조하지 않고
k3s 내부 상태를 직접 변경한다.

따라서 아래 두 값이 반드시 일치해야 한다.
```

<br />

| 항목 | 확인 방법 |
| :--- | :--- |
| **컨테이너 이름** (`next`, `nest`) | deployment yaml의 `containers.name` 값과 일치해야 함 |
| **Deployment 이름** (`next`, `nest`) | `kubectl get deployment` NAME 컬럼 값과 일치해야 함 |
| **Worker Node 재시작** | `sudo systemctl restart k3s-agent` 사용 |
| **Master Node 재시작** | `sudo systemctl restart k3s` 사용 (k3s-agent 아님) |
| **Worker Node의 미사용 이미지 청소** | 실제 파드가 뜨는 Worker 1, Worker 2에도 구버전 이미지가 캐시로 남으므로, 주기적으로 워커 노드에서 `sudo k3s crictl rmi --prune` 명령을 실행하거나 크론탭(crontab)에 주 1회 등록해 두면 워커 노드의 디스크도네! 위 답변에서 마크다운 코드 블록이 겹쳐서 잘려 보였네요. 우측 상단 복사 버튼 한 번으로 통째로 복사해서 바로 덮어쓰실 수 있도록 **문서 1 전체**를 깔끔하게 단일 코드 블록으로 출력해 드립니다!

(문서 2는 직전에 올려주신 버전이 이미 100% 완벽하므로 그대로 두시면 됩니다.)

````markdown
# Private Registry를 사용하는 반자동 배포 (용량 최적화 및 롤백 기능)

<br />
<br />

* 왜 Private Registry를 사용할까?

---

```
외부 Registry(Docker Hub, ghcr.io 등) 사용이 제한되거나 유료 플랜이 부담스러운 환경에서,
Master Node에 직접 Private Registry를 띄워 Worker Node들이 이미지를 내부 네트워크로 pull 할 수 있게 한다.

* 핵심 아키텍처 및 용량 최적화 포인트
 - 마스터 노드 자원 활용 (빌드 및 배포 중앙화)
   마스터 노드는 Taint 설정으로 인해 일반 서비스 파드가 뜨지 않아 평소에 유휴 자원이 많다.

   따라서 마스터 노드의 RAM(8GB~10GB)과 디스크(60GB+)를 넉넉히 할당한 뒤,
   마스터 노드에서 Docker 빌드, Private Registry 운영, DB 마이그레이션, kubectl 배포 명령을 중앙 집중식으로 처리한다.
   (이 방식을 쓰면 배포 빌드 시 실제 서비스가 돌아가는 Worker 1, Worker 2에 CPU/RAM 부하를 주지 않는다.)


 - 데이터 영구 보존 및 GC 작동 보장
   Registry 컨테이너의 내부 저장소를 호스트 폴더(-v)와 마운트하여
   재시작 시에도 이미지가 날아가지 않고 가비지 컬렉션(GC)이 실제 디스크 용량을 확보하도록 구성한다.


 - 2개 태그 로테이션(:latest, :rollback)
   태그가 무한정 쌓이는 것을 막기 위해 오직 2개의 태그만 교차 사용한다.

   이를 통해 별도의 무거운 .tar 백업 파일 없이도 즉각적인 롤백이 가능하며,
   언제나 직전 버전과 현재 버전 딱 2벌의 용량만 유지하고 나머지는 가비지 컬렉션으로 완전 삭제한다.
```

<br />
<br />
<br />
<br />

1. Master Node에 Private Registry 띄우기

```
RAM을 넉넉히(8GB~10GB) 할당한 "Master Node (예: 192.168.0.101)"에 Docker를 설치하고 경량 Registry 컨테이너를 실행한다.

Worker 노드들이 접근할 수 있도록 Master Node의 방화벽에서 5000번 포트를 반드시 허용해야 한다.

* 중요
  컨테이너가 재시작되어도 이미지가 날아가지 않고 가비지 컬렉션이 동작하도록
  반드시 데이터 저장 경로(/home/codej625/registry-data)도 함께 볼륨 마운트(-v)한다.
```

<br />

```zsh
# Master Node 방화벽에서 5000번 포트 허용 (사설망 내부 Worker 노드들의 이미지 Pull 허용)
sudo ufw allow from 192.168.0.0/24 to any port 5000 proto tcp
sudo ufw reload

# 설정 파일 및 데이터 저장용 디렉토리 생성
mkdir -p /home/codej625/yaml
mkdir -p /home/codej625/registry-data
```

<br />

`registry-config.yml`

```yaml
# 경로 -> /home/codej625/yaml/registry-config.yml

version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
```

<br />

```zsh
# 설정 파일 생성
cat <<EOF > /home/codej625/yaml/registry-config.yml
version: 0.1
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
EOF

# Registry 컨테이너 실행 (설정 파일 및 데이터 폴더 동시 마운트)
docker run -d \
  --name registry \
  --restart always \
  -p 5000:5000 \
  -v /home/codej625/yaml/registry-config.yml:/etc/docker/registry/config.yml \
  -v /home/codej625/registry-data:/var/lib/registry \
  registry:2
```

<br />
<br />
<br />

2. Docker insecure-registry 설정 (Master Node)

```
Private Registry는 HTTPS가 아닌 HTTP를 사용하므로,
빌드 및 Push를 수행할 Master Node의 Docker에 해당 주소를 insecure-registry로 허용해야 한다.

설정 후 Docker를 재시작해야 적용된다.
```

<br />

`/etc/docker/daemon.json`

```json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
```

```zsh
# 위의 192.168.0.101은 Registry가 떠 있는 Master Node의 고정 IP이다.
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.0.101:5000"]
}
EOF

# 도커 재시작
sudo systemctl restart docker
```

<br />
<br />
<br />

3. k3s registries.yaml 설정 (Master Node 및 모든 Worker Node 공통)

```
k3s(containerd)가 HTTP Registry에서 이미지를 pull할 수 있도록
Master Node와 모든 Worker Node(Worker 1, Worker 2)에 registries.yaml을 작성한다.

파일이 없으면 직접 생성하고,
작성 후 반드시 재시작해야 적용된다.
```

<br />

`/etc/rancher/k3s/registries.yaml (Master Node, Worker Node 동일)`

```yaml
mirrors:
  "192.168.0.101:5000": # 아이피는 Registry가 설치된 Master Node의 IP를 사용
    endpoint:
      - "http://192.168.0.101:5000"
```

```zsh
# 디렉토리가 없으면 먼저 생성 후 파일 작성 (모든 노드에서 실행)
sudo mkdir -p /etc/rancher/k3s/
cat <<EOF | sudo tee /etc/rancher/k3s/registries.yaml
mirrors:
  "192.168.0.101:5000":
    endpoint:
      - "http://192.168.0.101:5000"
EOF

# Master Node에서 재시작
sudo systemctl restart k3s

# Worker Node(Worker 1, Worker 2)에서 재시작
sudo systemctl restart k3s-agent
```

<br />
<br />
<br />

4. Deployment yaml 준비 (Master Node)

```
기존 이미지 주소를 Master Node의 Private Registry 주소(192.168.0.101:5000)로 맞추고,
imagePullPolicy를 Always로 설정하여 배포 시 항상 최신 이미지를 가져오게 한다.

/home/codej625/yaml 경로에 next-deployment.yaml 과 nest-deployment.yaml 파일을 준비해 두고,
nest-secret.yaml 만 먼저 kubectl apply 해두면, 이후 ./deploy.sh 실행 시 이미지를 빌드/푸시한 뒤
Deployment까지 자동으로 클러스터에 적용(apply) 및 롤링 업데이트해 준다.
```

<br />

`next-deployment.yaml 및 nest-deployment.yaml 내 이미지 경로 확인`

```yaml
# /home/codej625/yaml/next-deployment.yaml 예시
containers:
  - name: next
    image: 192.168.0.101:5000/next:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always

# /home/codej625/yaml/nest-deployment.yaml 예시
containers:
  - name: nest
    image: 192.168.0.101:5000/nest:latest # Master Node의 registry 주소로 설정
    imagePullPolicy: Always
```

<br />
<br />
<br />

5. GitHub SSH 키 설정 (Master Node)

```
배포 스크립트에서 git pull 시 매번 ID/PW를 입력하지 않도록
Master Node에 SSH 키를 생성하고 GitHub에 등록한다.
```

<br />

```zsh
# SSH 키 생성
ssh-keygen -t ed25519 -C "your-email@gmail.com" # 실제 사용하는 이메일 등록

# 공개키 확인 후 복사
cat ~/.ssh/id_ed25519.pub
```

```
# 깃 허브 접속
GitHub -> Settings -> SSH and GPG keys -> New SSH key -> 붙여넣기
```

```zsh
# remote URL을 SSH로 변경
cd /your/project/path
git remote set-url origin git@github.com:codej625/레포이름.git
```

<br />
<br />
<br />

6. 배포 스크립트 작성 (용량 극한 최적화 + DB 자동 마이그레이션 + 완벽한 롤백 보장)

```
* 스크립트 핵심 동작 원리
1. 태그 무한 증식 방지 및 롤백 보장 (:latest 와 :rollback 태그 로테이션)
   - 새로 빌드하기 전, 레지스트리에 있는 현재 정상 이미지(:latest)를 가져와 직전 백업 태그(:rollback)로 덮어씌워 푸시한다.
   - 최초 배포(1회차) 시에는 기존 이미지가 없으므로 이를 자동 감지하여 롤백 백업 단계를 안전하게 건너뛴다.
   - 이렇게 하면 별도의 무거운 .tar 백업 파일을 만들지 않아도, 배포 실패 시 즉시 :rollback 태그로 완벽한 롤백이 가능하다.
   - 동시에 2단계 이전의 구버전 이미지는 태그를 잃어버린 상태(Untagged)가 된다.

2. 무중단 DB 마이그레이션 자동 실행
   - 백엔드(NestJS) 이미지가 빌드된 직후, 쿠버네티스의 nest-secret에서 DATABASE_URL을 읽어와
     새로 빌드된 이미지의 drizzle 마이그레이션(npm run db:migrate:prod)을 배포 전에 먼저 적용한다.

3. 극한의 디스크 용량 확보
   - 배포 완료 후 Master Node의 Docker 이미지와 빌드 캐시를 전량 삭제(prune -a -f)한다.
   - 마지막으로 Registry 가비지 컬렉션(--delete-untagged)을 실행하여 태그를 잃은 구버전 이미지를 물리 디스크에서 완전히 지운다.
   - 결과적으로 레지스트리에는 언제나 현재 버전(:latest)과 직전 버전(:rollback) 딱 2벌의 용량만 유지된다.
```

<br />

`/home/codej625/deploy.sh`

```zsh
#!/usr/bin/env bash

set -euo pipefail

REGISTRY="192.168.0.101:5000"
NEXT_PUBLIC_API_URL="https://<사용하는 도메인>/api"
GIT_ROOT="/<프로젝트 경로>"
FRONTEND_PATH="$GIT_ROOT/front-end"
BACKEND_PATH="$GIT_ROOT/back-end"
YAML_DIR="/home/codej625/yaml"

NEXT_LATEST="$REGISTRY/next:latest"
NEXT_ROLLBACK="$REGISTRY/next:rollback"

NEST_LATEST="$REGISTRY/nest:latest"
NEST_ROLLBACK="$REGISTRY/nest:rollback"

HAS_NEXT_ROLLBACK=false
HAS_NEST_ROLLBACK=false

echo "=== 1. Registry 컨테이너 상태 확인 ==="
if [ ! "$(docker ps -q -f name=^registry$)" ]; then
    if [ "$(docker ps -aq -f status=exited -f name=^registry$)" ]; then
        docker start registry
    else
        docker run -d --name registry --restart always -p 5000:5000 \
          -v "$YAML_DIR/registry-config.yml":/etc/docker/registry/config.yml \
          -v /home/codej625/registry-data:/var/lib/registry \
          registry:2
    fi
fi

echo "=== 2. 기존 정상 이미지를 :rollback 태그로 백업 (태그 로테이션) ==="
# 기존 latest 이미지가 레지스트리에 있다면 pull 받아서 rollback 태그로 덮어써서 push 해둠
if docker pull "$NEXT_LATEST" >/dev/null 2>&1; then
    docker tag "$NEXT_LATEST" "$NEXT_ROLLBACK"
    docker push "$NEXT_ROLLBACK"
    HAS_NEXT_ROLLBACK=true
else
    echo "[안내] 기존 next:latest 이미지가 없습니다 (최초 배포)."
fi

if docker pull "$NEST_LATEST" >/dev/null 2>&1; then
    docker tag "$NEST_LATEST" "$NEST_ROLLBACK"
    docker push "$NEST_ROLLBACK"
    HAS_NEST_ROLLBACK=true
else
    echo "[안내] 기존 nest:latest 이미지가 없습니다 (최초 배포)."
fi

echo "=== 3. Git Pull ==="
cd "$GIT_ROOT" || exit 1
git pull

echo "=== 4. Docker Build & Push (순차 빌드) ==="
docker build \
  --build-arg NEXT_PUBLIC_API_URL="$NEXT_PUBLIC_API_URL" \
  -t "$NEXT_LATEST" \
  "$FRONTEND_PATH"
docker push "$NEXT_LATEST"

docker build -t "$NEST_LATEST" "$BACKEND_PATH"
docker push "$NEST_LATEST"

echo "=== 5. DB 마이그레이션 적용 (Drizzle) ==="
# k8s의 nest-secret에서 DATABASE_URL과 DATABASE_SSL을 읽어와 파드 배포 전에 DB 스키마를 최신화함
DB_URL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_URL}' | base64 -d)
DB_SSL=$(kubectl get secret nest-secret -n default -o jsonpath='{.data.DATABASE_SSL}' | base64 -d 2>/dev/null || echo "false")

docker run --rm \
  -e DATABASE_URL="$DB_URL" \
  -e DATABASE_SSL="${DB_SSL:-false}" \
  "$NEST_LATEST" \
  npm run db:migrate:prod

echo "=== 6. Kubernetes 배포 (NestJS 백엔드 먼저 배포) ==="
kubectl apply -f "$YAML_DIR/nest-deployment.yaml"
kubectl set image deployment/nest nest="$NEST_LATEST"
kubectl rollout restart deployment/nest

if kubectl rollout status deployment/nest --timeout=360s; then
  echo "[성공] nest 배포 완료."
else
  echo "[실패] nest 배포 실패. 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  exit 1
fi

echo "=== 7. Kubernetes 배포 (Next.js 프론트엔드 배포) ==="
kubectl apply -f "$YAML_DIR/next-deployment.yaml"
kubectl set image deployment/next next="$NEXT_LATEST"
kubectl rollout restart deployment/next

if kubectl rollout status deployment/next --timeout=360s; then
  echo "[성공] next 배포 완료."
else
  echo "[실패] next 배포 실패. 전체 서비스를 직전 이미지(:rollback)로 즉시 복구합니다..."
  if [ "$HAS_NEXT_ROLLBACK" = true ]; then
    kubectl set image deployment/next next="$NEXT_ROLLBACK"
    kubectl rollout status deployment/next --timeout=180s || true

    docker pull "$NEXT_ROLLBACK"
    docker tag "$NEXT_ROLLBACK" "$NEXT_LATEST"
    docker push "$NEXT_LATEST"
  fi
  if [ "$HAS_NEST_ROLLBACK" = true ]; then
    kubectl set image deployment/nest nest="$NEST_ROLLBACK"
    kubectl rollout status deployment/nest --timeout=180s || true

    docker pull "$NEST_ROLLBACK"
    docker tag "$NEST_ROLLBACK" "$NEST_LATEST"
    docker push "$NEST_LATEST"
  fi
  exit 1
fi

echo "=== 8. Master Node 로컬 Docker 이미지 및 빌드 캐시 전체 삭제 (용량 확보) ==="
docker system prune -a -f
docker builder prune -a -f

echo "=== 9. Registry 가비지 컬렉션 실행 (태그 잃은 구버전 영구 삭제) ==="
docker exec registry bin/registry garbage-collect \
  --delete-untagged \
  /etc/docker/registry/config.yml
docker restart registry

echo "=== 10. 배포 및 용량 최적화 완료 ==="
```

```zsh
# 실행 권한 부여
chmod +x /home/codej625/deploy.sh
```

<br />
<br />
<br />

7. 배포 및 확인

```
스크립트를 실행하고, Pod 상태와 이미지 태그를 확인하여
배포가 정상적으로 이루어졌는지 검증한다.
```

<br />

```zsh
# 1) 배포 실행 (Master Node의 deploy.sh가 있는 디렉터리에서)
./deploy.sh

# 2) Pod 상태 및 노드 확인 (Worker 1, Worker 2에 정상적으로 떴는지 확인)
kubectl get pods -o wide

# 3) 이미지 태그 및 새로 pull 받은 이미지 해시(Image ID) 확인
kubectl describe pod <pod-name> | grep -E "Image:|Image ID:"

# 4) 실제 확보된 디스크 여유 공간 확인
df -h
```

<br />
<br />
<br />

8. 주의사항 (kubectl set image 및 Worker 노드 용량 관리)

```
kubectl set image 명령어는 yaml 파일을 참조하지 않고
k3s 내부 상태를 직접 변경한다.

따라서 아래 두 값이 반드시 일치해야 한다.
```

<br />

| 항목 | 확인 방법 |
| :--- | :--- |
| **컨테이너 이름** (`next`, `nest`) | deployment yaml의 `containers.name` 값과 일치해야 함 |
| **Deployment 이름** (`next`, `nest`) | `kubectl get deployment` NAME 컬럼 값과 일치해야 함 |
| **Worker Node 재시작** | `sudo systemctl restart k3s-agent` 사용 |
| **Master Node 재시작** | `sudo systemctl restart k3s` 사용 (k3s-agent 아님) |
| **Worker Node의 미사용 이미지 청소** | 실제 파드가 뜨는 Worker 1, Worker 2에도 구버전 이미지가 캐시로 남으므로, 주기적으로 워커 노드에서 `sudo k3s crictl rmi --prune` 명령을 실행하거나 크론탭(crontab)에 주 1회 등록해 두면 워커 노드의 디스크도 영구적으로 깨끗하게 유지됨 |
