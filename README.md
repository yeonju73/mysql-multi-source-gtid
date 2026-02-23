# MySQL Multi-Source Replication (MSR) 실습

> [MySQL 8.4 Docs — 19.1.5 MySQL Multi-Source Replication](https://dev.mysql.com/doc/refman/8.4/en/replication-multi-source.html)

---

## 개념 정리

### MSR이란?

하나의 **Replica가 여러 Source로부터 동시에 데이터를 복제**받는 구조다.

기존 복제가 `Source 1 → Replica 1` 의 1:1 구조라면, MSR은 여러 Source의 데이터를 **하나의 Replica로 통합**할 수 있다.

```
Source A ──┐
           ├──▶ Replica
Source B ──┘
```

### 핵심 개념: Replication Channel

Replica는 각 Source마다 독립적인 **Channel(채널)** 을 생성한다.  
채널이 다르면 복제 연결도 완전히 독립적으로 동작한다.

```sql
-- Source A 채널 등록
CHANGE REPLICATION SOURCE TO ... FOR CHANNEL 'source-a';

-- Source B 채널 등록
CHANGE REPLICATION SOURCE TO ... FOR CHANNEL 'source-b';
```

### 실습 구성

| 역할 | 설명 |
|---|---|
| **Source A** | 첫 번째 데이터 원본 서버 |
| **Source B** | 두 번째 데이터 원본 서버 |
| **Replica** | Source A, B 양쪽으로부터 데이터를 복제받는 서버 |

Source A와 Source B 각각에 INSERT를 수행하면, Replica에서 두 데이터가 모두 조회되는 것을 확인한다.

```
Source A  →  INSERT 'hello from A'  ──┐
                                      ├──▶  Replica에서 두 데이터 모두 확인
Source B  →  INSERT 'hello from B'  ──┘
```

---

## 인프라 구성

### 마스터 서버 설정 (Source 1 & 2)

두 마스터 서버는 각각 고유한 `server-id`를 가진다.

```ini
[mysqld]
log-bin = mysql-bin
gtid_mode = ON
enforce_gtid_consistency = ON
server-id = 1  # Source 2는 2로 설정
```

복제용 계정 생성:

```sql
CREATE USER 'yeonju'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'yeonju'@'%';
```

### 레플리카 서버 설정

```ini
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
server-id = 3
```

---

## 복제 채널 연결

레플리카에서 각 마스터를 채널로 연결한다.  
`SOURCE_AUTO_POSITION=1` 을 사용하면 바이너리 로그 포지션을 수동으로 지정할 필요 없이 GTID 기반으로 자동 위치를 잡는다.

```sql
-- Source 1 채널 등록
CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='[Master1_IP]', 
    SOURCE_USER='username', 
    SOURCE_PASSWORD='password', 
    SOURCE_AUTO_POSITION=1 FOR CHANNEL 'source1_channel';

-- Source 2 채널 등록
CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='[Master2_IP]', 
    SOURCE_USER='username', 
    SOURCE_PASSWORD='password', 
    SOURCE_AUTO_POSITION=1 FOR CHANNEL 'source2_channel';

-- 복제 시작
START REPLICA FOR CHANNEL 'source1_channel';
START REPLICA FOR CHANNEL 'source2_channel';
```

---

## 검증

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

### 복제 상태 모니터링

```sql
-- 채널별 복제 상태 확인 (IO/SQL Running: Yes 이면 정상)
SHOW REPLICA STATUS\G

-- 복제된 GTID 집합 확인
SELECT @@GLOBAL.GTID_EXECUTED;
```

