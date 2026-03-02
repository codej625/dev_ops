# Private Registry를 사용하는 반자동 배포

<br />
<br />

* 왜 Private Registry를 사용할까?

---

```
외부 Registry(Docker Hub, ghcr.io 등) 사용이 불가한 환경에서,
Master Node에 직접 Private Registry를 띄워 Worker Node들이
이미지를 내부 네트워크로 pull 할 수 있게 한다.
```

<br />
<br />
<br />
<br />

1. Master Node에 Private Registry 띄우기

```
Docker가 설치된 "Master Node"에 경량 Registry 컨테이너를 실행한다.

5000번 포트로 외부에 노출하며, 항상 재시작되도록 설정한다.

이미지 저장소 서버 이름은 registry,
버전은 2를 사용한다.
```

<br />

```zsh
docker run -d \
  --name registry \ 
  --restart always \
  -p 5000:5000 \
  registry:2
```

<br />
<br />
<br />

2. Docker insecure-registry 설정 (Master Node)

```
Private Registry는 HTTPS가 아닌 HTTP를 사용하므로,
Docker에 해당 주소를 insecure-registry로 허용해야 한다.

설정 후 Docker를 재시작해야 적용된다.
```

<br />

`/etc/docker/daemon.json`

```json
{
  "insecure-registries": ["192.168.0.219:5000"]
}
```

```zsh
// 도커 재시작
sudo systemctl restart docker
```

<br />
<br />
<br />

3. k3s registries.yaml 설정

```
k3s가 HTTP Registry에서 이미지를 pull할 수 있도록
Master Node와 모든 Worker Node에 registries.yaml을 작성한다.

파일이 없으면 직접 생성하고, 작성 후 반드시 재시작해야 적용된다.
```
<br />

`/etc/rancher/k3s/registries.yaml (Master Node, Worker Node 동일)`

```yaml
mirrors:
  "192.168.0.219:5000": # 아이피는 마스터 노드의 IP를 사용
    endpoint:
      - "http://192.168.0.219:5000" # 아이피는 마스터 노드의 IP를 사용
```

```zsh
# 디렉토리가 없으면 먼저 생성
sudo mkdir -p /etc/rancher/k3s/

# Master Node 재시작
sudo systemctl restart k3s

# Worker Node 재시작
sudo systemctl restart k3s-agent
```

<br />
<br />
<br />

4. Deployment yaml 수정

```
기존 로컬 이미지 주소를 Private Registry 주소로 변경하고,
imagePullPolicy를 Always로 설정하여 항상 최신 이미지를 가져오게 한다.

수정 후 kubectl apply로 한 번 적용해두면 이후엔 스크립트로만 배포 가능하다.
```

<br />

`/home/codej625/yaml/next-deployment.yaml`

```yaml
containers:
  - name: next
    image: 192.168.0.219:5000/next:latest # registry 주소로 변경
    imagePullPolicy: Always
```

```zsh
// deployment 적용
kubectl apply -f /home/codej625/yaml/next-deployment.yaml
```

<br />
<br />
<br />

5. GitHub SSH 키 설정

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
// 깃 허브 접속
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

6. 배포 스크립트 작성

```
git pull -> docker build -> registry push -> kubectl 배포까지
모든 과정을 스크립트 하나로 자동화한다.

TAG를 timestamp로 생성하여 매 배포마다 새 이미지로 인식되게 한다.
```

<br />

`/home/codej625/deploy.sh`

```zsh
#!/bin/zsh

APP_NAME="next" # app 이름
REGISTRY="192.168.0.219:5000" # <마스터 노드 아이피>:<위에서 열어둔 포트>
TAG=$(date +%Y%m%d%H%M%S) # 태그를 타임스탬프로 사용
PROJECT_PATH="/home/codej625/프로젝트경로" # 실제 경로로 변경

echo "=== 1. Git Pull ==="
cd $PROJECT_PATH
git pull

echo "=== 2. Docker Build & Push ==="
docker build -t $REGISTRY/$APP_NAME:$TAG .
docker push $REGISTRY/$APP_NAME:$TAG

echo "=== 3. Kubernetes 배포 ==="
kubectl set image deployment/$APP_NAME $APP_NAME=$REGISTRY/$APP_NAME:$TAG
kubectl rollout status deployment/$APP_NAME

echo "=== 완료! ==="
```

```zsh
// 권한 획득
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
# 배포 실행
sh /home/codej625/deploy.sh

# Pod 상태 및 노드 확인
kubectl get pods -o wide

# 이미지 태그 확인
kubectl describe pod <pod-name> | grep Image
```

<br />
<br />
<br />

8. 주의사항 (kubectl set image)

```
kubectl set image 명령어는 yaml 파일을 참조하지 않고
k3s 내부 상태를 직접 변경한다.

따라서 아래 두 값이 반드시 일치해야 한다.
```

<br />

| 항목 | 확인 방법 |
| :--- | :--- |
| **컨테이너 이름** (`$APP_NAME`) | deployment yaml의 `containers.name` 값과 일치해야 함 |
| **Deployment 이름** (`$APP_NAME`) | `kubectl get deployment` NAME 컬럼 값과 일치해야 함 |
| **Worker Node 재시작** | `k3s-agent` 사용 / 오래 걸릴 수 있음, 다른 터미널에서 진행 가능 |
| **Master Node 재시작** | `k3s` 사용 (k3s-agent 아님) |
