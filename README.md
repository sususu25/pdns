```
## PDNS (PowerDNS Authoritative) - 사용 방법

### 포함 파일
- docker-compose.yaml : PDNS 컨테이너 실행
- pdns.conf : PDNS 설정(외부 MySQL/MariaDB 접속 정보)
- pdns_dump.sql : PowerDNS MySQL 스키마 예시

### 0) 사전 작업 (Rocky 8.10)
sudo dnf install -y dnf-utils
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker

### 1) DB에 스키마 적용(테이블 생성)
- DB 서버 접속정보(DB_HOST/PORT/USER/PASS/DB_NAME)를 준비한 뒤, 아래 실행
sudo dnf -y install mariadb
mysql -h <DB_HOST> -P <DB_PORT> -u <DB_USER> -p <DB_NAME> < pdns_dump.sql

### 2) pdns.conf에 DB 접속정보 입력
- pdns.conf에서 아래 항목을 실제 값으로 수정
gmysql-host=<DB_HOST>
gmysql-port=<DB_PORT>
gmysql-user=<DB_USER>
gmysql-password=<DB_PASS>
gmysql-dbname=<DB_NAME>

### 3) PDNS 실행
- 도커 컴포즈가 있는 폴더에서 실행
docker compose up -d
docker ps

### 4) 존/레코드 등록 예시
- 권한 DNS라서 “존(zone)”을 만들어야 질의에 NOERROR로 응답함
docker exec -it pdns-server pdnsutil create-zone corp.internal ns1.corp.internal
docker exec -it pdns-server pdnsutil add-record corp.internal www A 3600 10.0.0.20

### 5) 확인
- PDNS 서버에서 로컬 확인
dig @127.0.0.1 +norecurse www.corp.internal A
- 외부에서 확인(방화벽/보안그룹 53/tcp, 53/udp 허용 필요)
dig @<PDNS_SERVER_IP> +norecurse www.corp.internal A

### 6) DNSSEC 설정
- DNSSEC 활성화
docker exec -it pdns-server pdnsutil secure-zone corp.internal

- 생성 키 확인
docker exec -it pdns-server pdnsutil list-keys corp.internal
docker exec -it pdns-server pdnsutil show-zone corp.internal

- SOA 시리얼 자동 증가 설정
docker exec -it pdns-server pdnsutil set-meta corp.internal SOA-EDIT INCREMENT-WEEKS

- NSEC3 활성화
docker exec -it pdns-server pdnsutil set-nsec3 corp.internal '1 0 1 ab'

- 존 정리(rectify)
docker exec -it pdns-server pdnsutil rectify-zone corp.internal

### 7) 확인
dig @127.0.0.1 -p 53 www.test.internal A +dnssec
dig @127.0.0.1 -p 53 test.internal DNSKEY +dnssec
dig @127.0.0.1 -p 53 nonexistent.test.internal A +dnssec

### 종료
docker compose down


## REFUSED 발생 시(존/레코드 등록 후에도 응답이 거절될 때)
- 간헐적으로 PDNS 컨테이너가 초기 구동 시 설정/백엔드 반영이 불완전해 `REFUSED`가 발생할 수 있습니다.
- 아래 명령으로 컨테이너를 **재생성(설정 재로딩)** 한 뒤 다시 질의하세요.

docker compose -f docker-compose.yaml up -d --force-recreate
```