# MySQL Multi-Source Replication (MSR) 실습

> [MySQL 8.4 Docs — 19.1.5 MySQL Multi-Source Replication](https://dev.mysql.com/doc/refman/8.4/en/replication-multi-source.html)

---

## 개념 정리

### 1. MSR이란?
<img width="807" height="282" alt="image" src="https://github.com/user-attachments/assets/93c46351-2f17-4725-b821-cf488309608d" />

하나의 **Replica가 여러 Source로부터 동시에 데이터를 복제**받는 구조

기존 복제가 `Source 1 → Replica 1` 의 1:1 구조라면, MSR은 여러 Source의 데이터를 **하나의 Replica로 통합** 가능

### 2. Replication Channel

Replica는 각 Source마다 독립적인 **Channel(채널)** 생성
채널이 다르면 복제 연결도 완전히 독립적으로 동작

```sql
-- Source A 채널 등록
CHANGE REPLICATION SOURCE TO ... FOR CHANNEL 'source-a';

-- Source B 채널 등록
CHANGE REPLICATION SOURCE TO ... FOR CHANNEL 'source-b';
```

### 실습 구성

| 역할 | 설명 |
|---|---|
| **mysql-source-1** | 첫 번째 데이터 원본 서버 |
| **mysql-source-2** | 두 번째 데이터 원본 서버 |
| **mysql-replica** | Source 1, 2 양쪽으로부터 데이터를 복제받는 서버 |

실습 목표: Source 1와 Source 2 각각에 INSERT를 수행하면, Replica에서 두 데이터가 모두 조회되는 것 확인

```
Source 1  →  INSERT 'hello from 1'  ──┐
                                      ├──▶  Replica에서 두 데이터 모두 확인
Source 2  →  INSERT 'hello from 2'  ──┘
```

---

## 실습 과정

### 1. 환경 구축 및 네트워크 설정

이 프로젝트는 Docker 컨테이너 간의 원활한 이름 기반 통신을 위해 **사용자 정의 브리지 네트워크**를 생성하고 각 프로세스 연결

#### [Step 1] Docker 네트워크 생성

기본 `bridge` 네트워크는 컨테이너 이름으로 서로를 찾을 수 없으므로, 별도의 네트워크 생성

```bash
docker network create replication-net

```

#### [Step 2] 마스터 및 레플리카 컨테이너 실행

각 컨테이너를 생성한 네트워크에 할당하여 실행

```bash
# Master 1 실행
docker run -d --name mysql-source-1 --network replication-net \
  -e MYSQL_ROOT_PASSWORD=password mysql:8.0

# Master 2 실행
docker run -d --name mysql-source-2 --network replication-net \
  -e MYSQL_ROOT_PASSWORD=password mysql:8.0

# Replica 실행
docker run -d --name mysql-replica --network replication-net \
  -e MYSQL_ROOT_PASSWORD=password mysql:8.0

```

#### [Step 3] 통신 확인

mysql-replica 컨테이너에서 각 마스터 서버로의 접근 가능 여부 확인

```bash
# mysql-replica 내부에서 mysql-source-2로 접속 테스트
docker exec -it mysql-replica mysql -h mysql-source-2 -u yeonju -p

```

---

## 2. 마스터 서버 설정 (Source 1 & 2)

두 마스터 서버는 각각 고유한 `server-id`를 가짐

```ini
[mysqld]
log-bin = mysql-bin
gtid_mode = ON
enforce_gtid_consistency = ON
server-id = 1  # Source 2는 2로 설정

```

복제용 계정 생성:

```sql
CREATE USER 'username'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'username'@'%';
FLUSH PRIVILEGES;

```

---

## 3. 레플리카 서버 설정

```ini
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
server-id = 3

```

---

## 4. 복제 채널 연결

레플리카에서 각 마스터를 채널로 연결

`SOURCE_AUTO_POSITION=1` 을 사용하면 바이너리 로그 포지션을 수동으로 지정할 필요 없이 GTID 기반으로 자동 위치를 잡음

```sql
-- Source 1 채널 등록
CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='mysql-source-1', 
    SOURCE_USER='yeonju', 
    SOURCE_PASSWORD='password', 
    SOURCE_AUTO_POSITION=1 FOR CHANNEL 'source1_channel';

-- Source 2 채널 등록
CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='mysql-source-2', 
    SOURCE_USER='yeonju', 
    SOURCE_PASSWORD='password', 
    SOURCE_AUTO_POSITION=1 FOR CHANNEL 'source2_channel';

-- 복제 시작
START REPLICA FOR CHANNEL 'source1_channel';
START REPLICA FOR CHANNEL 'source2_channel';

```

---

## 실습 결과

### 데이터 통합 확인

각 Source에 INSERT 후 Replica에서 조회한다.

```sql
-- Source 1에서 실행
INSERT INTO products VALUES ('apple');

-- Source 2에서 실행
INSERT INTO products VALUES ('source2-1');

-- Replica에서 확인
SELECT * FROM products;
-- → 두 데이터 모두 출력됨
```
<img width="615" height="317" alt="image" src="https://github.com/user-attachments/assets/d903ed14-10f5-474c-b778-8468094c76a0" />

### 복제 상태 모니터링

```sql
-- 채널별 복제 상태 확인 (IO/SQL Running: Yes 이면 정상)
SHOW REPLICA STATUS\G

```
<img width="800" height="297" alt="image" src="https://github.com/user-attachments/assets/a1e95fb1-cec3-4fef-bc10-1a3d5307f4a6" />
<img width="800" height="302" alt="image" src="https://github.com/user-attachments/assets/4ad108ca-8e62-4465-b583-fbfc462e15c7" />

