# 도커 파일을 만들 때 쿨팁 모음

<br />
<br />

* Dockerfile에서 npm ci를 쓰는 이유?

---

```
Dockerfile에서 npm ci를 쓰는 이유는
항상 같은 버전으로 재현 가능한 빌드를 보장하기 위해서이다.

자세한 건 밑에 표에서 확인
```

<br />
<br />
<br />
<br />

1. npm install 대신 npm ci를 사용하는 이유

<br />

| 항목               | `npm install`                                                                                                 | `npm ci`                                                                                                    |
|--------------------|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| **사용 파일**      | `package.json`, `package-lock.json` (있을 경우)                                                               | `package-lock.json`                                                                                        |
| **설치 방식**      | `package.json`에 명시된 패키지를 설치하며, `package-lock.json`에 맞지 않으면 업데이트                           | `package-lock.json`에 명시된 정확한 버전의 패키지를 설치                                                   |
| **설치 대상**      | 새로운 패키지를 추가하거나 업데이트할 때 사용                                                                  | CI/CD 파이프라인에서 일관된 패키지 설치를 위해 사용                                                        |
| **속도**           | 상대적으로 느림                                                                                                | 상대적으로 빠름                                                                                            |
| **기존 `node_modules`** | 유지하며, 새로운 패키지만 추가                                                                              | 기존 `node_modules` 폴더를 삭제하고 재설치                                                                  |
| **파일 필요 여부** | `package-lock.json` 파일이 없어도 동작                                                                          | `package-lock.json` 파일이 반드시 필요                                                                     |
| **실패 조건**      | `package.json`이 `package-lock.json`와 일치하지 않아도 설치 가능                                                | `package.json`이 `package-lock.json`와 일치하지 않으면 실패                                                 |
| **사용 사례**      | 로컬 개발 환경에서 패키지를 추가하거나 관리할 때 사용                                                           | CI/CD 환경에서 일관된 설치를 보장하기 위해 사용                                                             |

<br />
<br />
<br />

2. Dockerfile 사용 예시

```docker
FROM node:18-alpine

WORKDIR /app

COPY package.json package-lock.json ./

# 여기!
RUN npm ci

COPY . .

RUN npm run build

CMD ["node", "server.js"]
```

<br />

| 환경 | 명령어 | 이유 |
| :--- | :--- | :--- |
| **로컬 개발** | `npm install` | 패키지 추가/변경할 때 |
| **Dockerfile** | `npm ci` | 빌드할 때, 재현 가능한 이미지 생성 |
| **GitHub Actions** | `npm ci` | CI/CD에서 일관된 테스트/빌드 보장 |
