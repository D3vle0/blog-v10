---
title: "Matrix Synapse 서버를 새로운 서버로 이전하기"
date: 2026-10-22
draft: false
categories: ["Server"]
tags: ["matrix", "synapse", "linux"]
cover:
  image: images/posts/transfer-matrix-to-new-server/1.webp
  caption: ""
ShowToc: true
TocOpen: true
---

## 주의 사항
- [이 글](/blog/linux-synapse)에서 진행한 대로 서버를 구축한 경우에 가장 적절한 방식이다.
- 새 서버의 IP 주소는 달라져도 괜찮지만 도메인은 그대로 유지해야 한다.

## Docker 구성 확인

```sh
cd <matrix docker compose 경로>
docker compose config --services
```

synapse 서비스명과 데이터베이스 서비스명을 확인한다. (컨테이너명과는 다르니 `docker-compose.yaml` 파일을 읽으며 검증해도 좋다.)

## PostgreSQL 데이터베이스 백업

synapse 설정 파일에서 database 항목을 확인하자.

```yaml
database:
  name: psycopg2
  args:
    user: <db 사용자명>
    password: "<db 비밀번호>"
    database: <db 이름>
    host: <db 호스트>
    port: <db 포트>
```

db 사용자명, db 이름, 호스트, 포트를 기록해 둔다.

```sh
docker compose exec <db 서비스명> \
  psql -U <db 사용자명> -d <db 이름> -c 'SELECT version();'
```

PostgreSQL 컨테이너에 접속할 수 있는지 확인한다. 버전 정보가 나오면 정상 작동하는 것이다. 인증 오류가 발생하면 POSTGRES_USER와 Synapse의 DB 사용자 설정이 일치하는지, 비밀번호 인증이 필요한지 확인한다.  

[Synapse의 공식 백업 가이드 에서는 `pg_dump`의 압축 사용자 지정 형식과 `e2e_one_time_keys_json` 테이블 데이터 제외를 권장한다.](https://element-hq.github.io/synapse/latest/usage/administration/backups.html) 오래된 백업을 복원하면 이미 사용한 일회용 암호화 키가 다시 발급되어 메시지 복호화 문제가 발생할 수 있기 때문이다. 다음 명령은 Synapse가 실행 중인 상태에서도 초기 백업을 만들 수 있다.

```sh
set -o pipefail
BACKUP="$HOME/synapse-$(date +%Y%m%d-%H%M%S).dump"
docker compose exec -T <db 서비스명> \
  pg_dump -U <db 사용자명> \
  -Fc \
  --exclude-table-data=e2e_one_time_keys_json \
  <db 이름> > "$BACKUP"

file "$BACKUP"
docker compose exec -T <db 서비스명> pg_restore --list < "$BACKUP" | head -n 30
sha256sum "$BACKUP" | tee "$BACKUP.sha256"
```

백업 파일과 체크섬 파일 모두 별도 저장소에 복사해 둔다.

## Synapse 미디어·설정·서명 키 백업

```sh
docker ps
docker inspect <Synapse 컨테이너명> -f '{{json .Mounts}}'
docker inspect <Synapse 컨테이너명> -f '{{.Config.Image}}'
```

Source 값이 실제 synapse 데이터가 저장되는 위치다. 이하 "데이터 위치" 라고 서술하겠다. 현재 서버에 설치된 Synapse 버전도 기록해 둔다.

```sh
docker compose stop <synapse 서비스명>
grep -E '^(server_name|signing_key_path|media_store_path|database):' <데이터 위치>/homeserver.yaml
sudo tar --numeric-owner -czpf \
  "$HOME/synapse-data.tar.gz" \
  -C <데이터 위치의 상위 경로> data
```

여기서 데이터 위치의 상위 경로란 만약 데이터 위치가 `/a/b/c/data`라면 `/a/b/c`이다.

```sh
ls -lh "$HOME/synapse-data.tar.gz"
tar -tzf "$HOME/synapse-data.tar.gz" | head -n 30
sha256sum "$HOME/synapse-data.tar.gz" | tee "$HOME/synapse-data.tar.gz.sha256"
```

## Nginx Proxy Manager 백업

SSL 인증서도 새 서버에서 동일하게 사용해야 문제가 생기지 않는다.

```sh
mkdir -p "$HOME/npm-backup"
chmod 700 "$HOME/npm-backup"
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
docker inspect <NPM 컨테이너명> -f '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

데이터 위치를 확인한다.

```sh
docker stop <NPM 컨테이너명>

sudo tar --numeric-owner -czpf \
  "$HOME/npm-backup/npm-data.tar.gz" \
  -C <NPM 데이터 위치의 상위 경로> data

sudo tar --numeric-owner -czpf \
  "$HOME/npm-backup/npm-letsencrypt.tar.gz" \
  -C <NPM 데이터 위치의 상위 경로> letsencrypt
```

## 새 서버에서 데이터 복원하기

새 서버의 Nginx Proxy Manager 데이터 저장 위치에 npm-data.tar.gz와 npm-letsencrypt.tar.gz 압축을 푼다. Synapse 컨테이너 배포를 위해 사용한 `docker-compose.yaml` 파일을 그대로 사용하되, 버전을 기존 서버와 동일하게 맞춘다. Synapse 데이터가 백업된 synapse-data.tar.gz를 데이터 저장되는 곳에 압축을 푼다.

## PostgreSQL 데이터베이스 복원

```sh
docker compose up -d <db 서비스명>
docker logs --tail=50 <db 컨테이너명>
docker exec <db 컨테이너명> pg_isready -U <db 사용자명> -d <db 이름>
docker exec -i <db 컨테이너명> \
  pg_restore -U <db 사용자명> \
  -d <db 이름> \
  --no-owner \
  --no-privileges \
  < <synapse dump 파일 경로>
docker exec <db 컨테이너명> \
  psql -U <db 사용자명> -d <db 이름> \
  -c '\dt'
```

이렇게 했을 때 테이블이 정상적으로 표시되면 DB 복원이 진행된 것이다.


```sh
docker compose up -d <synapse 서비스명>
```

만약 `psycopg2.OperationalError: connection to server at "XXXX" (XXX.XXX.XXX.XXX), port XXXX failed: FATAL:  password authentication failed for user "XXXX"` 와 같은 에러가 발생한다면 db의 비밀번호를 변경하고 컨테이너를 재시작하면 된다.

```sh
docker exec -it <db 컨테이너명> psql -U <db 사용자명> -d <db 이름>
```

PostgreSQL 쉘 내부에서

```sh
\password <db 사용자명>

\q
```

으로 새 비밀번호를 설정한 뒤 `homeserver.yaml` 파일의 database 항목에 새 비밀번호를 반영한다.

