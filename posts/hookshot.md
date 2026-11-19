---
title: "Matrix Hookshot으로 Synapse 서버에 웹훅 연동하기 🪝"
date: 2026-10-28
draft: true
categories: ["Server"]
tags: ["matrix", "synapse", "linux"]
cover:
  image: images/posts/hookshot/1.webp
  caption: ""
ShowToc: true
TocOpen: true
---

## 네트워크 구성

Synapse, Hookshot, Nginx Proxy Manager 가 한 Docker 네트워크 안으로 묶여야 한다.

```sh
docker network create hookshot_net
mkdir -p <docker compose 파일 저장할 경로>
sudo chown -R "$USER":"$(id -gn)" <docker compose 파일 저장할 경로>
cd <docker compose 파일 저장할 경로>
```

이후 Synapse와 Nginx Proxy Manager의 `docker-compose.yml` 파일을 수정하고 컨테이너를 재시작하여 `hookshot_net` 네트워크를 연결한다.

```yml
services:
  synapse:
    # 기존 image, volumes, ports 등 유지
    networks:
      default: {}
      hookshot_net:
        aliases:
          - Synapse

networks:
  default:
    # 기존 네트워크 설정 유지
  hookshot_net:
    external: true
    name: hookshot_net
```

Synapse의 `docker-compose.yml` 은 위와 같이,

```yml
services:
  app:
    # 기존 image, ports, volumes 등 유지
    networks:
      default: {}
      hookshot_net: {}

networks:
  default:
    # 기존 네트워크 설정 유지
  hookshot_net:
    external: true
    name: hookshot_net
```

Nginx Proxy Manager의  `docker-compose.yml` 은 위와 같이 수정하면 된다. 서비스 명 또는 이외의 설정은 자신의 환경에 맞게 입력해주어야 한다.

`docker network inspect hookshot_net` 을 했을 때 Nginx Proxy Manager와 Synapse 컨테이너 정보가 나오면 성공적으로 네트워크가 연결된 것이다.

## hookshot의 docker-compose.yml 설정

```yml
services:
  matrix-hookshot:
    image: ghcr.io/matrix-org/matrix-hookshot:latest
    container_name: matrix-github-hookshot
    hostname: matrix-hookshot
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    environment:
      TZ: Asia/Seoul
    volumes:
      - ./data:/data:rw
    networks:
      default: {}
      hookshot_net:
        aliases:
          - matrix-hookshot

networks:
  hookshot_net:
    external: true
    name: hookshot_net
```

이 구성은 조금 전 생성한 외부 hookshot_net 네트워크를 사용한다. Compose 파일 자체는 별개지만, Synapse와 통신하기 위해 네트워크 하나를 공유하는 방식이다. 또한 포트 매핑을 의도적으로 생략했다. GitHub의 Webhook은 Nginx Proxy Manager를 통해 전달하고, Appservice 포트는 Docker 내부에서만 접근하게 한다.

## 보안 토큰과 암호화 키 생성

```sh
umask 077
openssl rand -hex 32 > as_token.txt
openssl rand -hex 32 > hs_token.txt
openssl rand -hex 32 > github_webhook_secret.txt
openssl genpkey -out passkey.pem -outform PEM -algorithm RSA -pkeyopt rsa_keygen_bits:4096
chmod 600 as_token.txt hs_token.txt github_webhook_secret.txt passkey.pem
```

## GitHub App 생성

[이곳에서 새 앱을 생성하고](https://github.com/settings/apps) 다음과 같이 입력한다.

| 항목 | 값 |
| --- | --- |
| GitHub App name | `GitHub Matrix Hookshot` |
| Homepage URL | `https://<Synapse 홈서버 도메인>` |
| Redirect URI | `https://hookshot.<Synapse 홈서버 도메인>/oauth` |
| Webhook Active | 활성화 |
| Webhook URL | `https://hookshot.<Synapse 홈서버 도메인>/github/webhook` |
| Webhook secret | `github_webhook_secret.txt` 내용 |

다음과 같이 Repository Permissions에서 권한을 설정 한다.

| 권한 | 값 |
| --- | --- |
| Actions | Read-only |
| Contents | Read-only |
| Discussions | Read & write |
| Issues | Read & write |
| Metadata | Read-only |
| Projects | Read & write |
| Pull requests | Read & write |

`Subscribe to events` 에서 알림을 원하는 이벤트를 선택한다. 필자는 Commit comment, Create, Discussion, Discussion Comment, Delete, Fork, Gollum, Issue comment, Issues, Public, Pull Requests, Pull request review, Pull request review comment, Pull request review thread, Push, Release, Repository, Star, Watch, Workflow run, Sub issues를 선택했다.  
Where can this GitHub App be installed? 에서 `Any Account`를 선택한다.  

App ID, Client ID 확인하고 Credentials 에서 Client secret 과 private key를 생성한다.

## Hookshot 설정 파일 만들기

```sh
mkdir -p data
chmod 700 data
```

조금 전 생성한 `passkey.pem` 와 GitHub 에서 다운로드 받은 private key 파일을 data 폴더로 이동시킨다.

```sh
chmod 600 data/*.pem
```

`data/config.yml` 을 다음과 같이 생성한다.

```yml
bridge:
  domain: "<Synapse 홈서버 도메인>"
  url: "http://synapse:8008"
  mediaUrl: "https://<Synapse 홈서버 도메인>"
  port: 9993
  bindAddress: "0.0.0.0"

passFile: "/data/passkey.pem"

logging:
  level: info
  colorize: false
  json: false
  timestampFormat: "HH:mm:ss:SSS"

listeners:
  - port: 9000
    bindAddress: "0.0.0.0"
    resources:
      - webhooks

github:
  auth:
    id: <App ID>
    privateKeyFile: "/data/<GitHub 에서 다운로드 받은 private key 파일명>"
  webhook:
    secret: <github_webhook_secret.txt 의 값>
  oauth:
    client_id: <Client ID>
    client_secret: <Client secret>
    redirect_uri: "https://<Synapse 홈서버 도메인>/oauth"
  defaultOptions:
    showIssueRoomLink: true

permissions:
  - actor: "<Synapse 홈서버 도메인>"
    services:
      - service: "*"
        level: admin
```

`data/registration.yml`을 다음과 같이 작성한다.

```yml
id: matrix-hookshot-github

as_token: <as_token.txt 내용>
hs_token: <hs_token.txt 내용>

namespaces:
  rooms: []
  users:
    - regex: "@_github_.*:example\\.com"
      exclusive: true
    - regex: "@_gitlab_.*:example\\.com"
      exclusive: true
    - regex: "@_jira_.*:example\\.com"
      exclusive: true
    - regex: "@_webhooks_.*:example\\.com"
      exclusive: true
  aliases:
    - regex: "#github_.+:example\\.com"
      exclusive: true

sender_localpart: hookshot
url: "http://matrix-github-hookshot:9993"
rate_limited: false
receive_ephemeral: true
```

`regex` 항목의 `example\\.com` 부분은 본인의 Synapse 홈서버 도메인으로 변경해야 한다. 정규표현식이므로 마침표(`.`) 앞에 역슬래시(`\\`)를 붙여 이스케이프 처리하면 된다. (예: `matrix.yourdomain.com` → `matrix\\.yourdomain\\.com`)

## Synapse에 Hookshot 연결하기

`data/registration.yml` 파일을 synapse의 docker-compose 파일이 있는 위치로 복사한다.

```sh
cp <hookshot compose 경로>/data/registration.yml <synapse compose 경로>/data/hookshot-registration.yml
```

Synapse의 `data/homeserver.yaml` 파일에서 `app_service_config_files` 항목을 수정한다. 만약 다른 Appservice 등록 파일이 이미 있다면 기존 항목을 유지하고 새 경로만 추가하면 된다. 같은 키를 중복해서 선언하면 안 된다.

```yaml
app_service_config_files:
  - /data/hookshot-registration.yml
```

## 리버스 프록시 설정

- 도메인: `hookshot.<Synapse 홈서버 도메인>`
- scheme: http
- Forward Hostname/IP: matrix-github-hookshot
- Forward Port: 9000
- Block Common Exploits, Force SSL, HTTP/2 Support 활성화

위와 같이 설정 후 Synapse, Nginx Proxy Manager 컨테이너 재시작, hookshot 컨테이너를 시작한다. 