# 🚀 MySQL Multi-Source Replication with GTID

분산된 여러 MySQL 마스터 서버의 데이터를 하나의 레플리카(Replica) 서버로 통합하고, **GTID(Global Transaction ID)** 를 통해 장애 발생 시 스스로 복구되는 **Self-Healing(자기 치유)** 파이프라인을 구축한 프로젝트입니다.

## 🎯 Project Overview

* **Objective**: 여러 마스터 서버의 실시간 데이터 통합 및 고가용성 복제 환경 구축
* **Key Technology**: MySQL Multi-Source Replication, GTID, Docker
* **Main Feature**: 복제 중단 후 재개 시 별도의 포지션 설정 없이 자동으로 누락된 데이터를 보정하는 "Auto-healing" 구현

---

## 🛠️ Infrastructure Configuration

### 1. Master Servers (Source 1 & 2)

두 대의 마스터 서버는 각각 독립된 데이터를 관리하며, 고유한 `server-id`를 가집니다.

* **Common Settings (`my.cnf`)**
```ini
[mysqld]
log-bin = mysql-bin
gtid_mode = ON
enforce_gtid_consistency = ON
server-id = 1  # Master 2는 2로 설정

```


* **Replication Account**
```sql
CREATE USER 'yeonju'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'yeonju'@'%';

```



### 2. Replica Server

두 마스터의 데이터를 채널별로 받아들이는 통합 서버입니다.

* **Settings (`my.cnf`)**
```ini
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
server-id = 3

```



---

## 🔗 Replication Setup (Multi-Source Channels)

레플리카 서버에서 각 마스터를 독립된 채널로 연결합니다. **`SOURCE_AUTO_POSITION=1`**을 사용하여 바이너리 로그 포지션을 수동으로 맞출 필요가 없습니다.

```sql
-- Channel for Master 1
CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='[Master1_IP]', 
    SOURCE_USER='username', 
    SOURCE_PASSWORD='password', 
    SOURCE_AUTO_POSITION=1 FOR CHANNEL 'source1_channel';

-- Channel for Master 2
CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='[Master2_IP]', 
    SOURCE_USER='username', 
    SOURCE_PASSWORD='password', 
    SOURCE_AUTO_POSITION=1 FOR CHANNEL 'source2_channel';

-- Start Replication
START REPLICA FOR CHANNEL 'source1_channel';
START REPLICA FOR CHANNEL 'source2_channel';

```

---

## 🧪 Verification & Test Scenarios

### 1. Data Integration (데이터 통합)

각 마스터에 입력된 서로 다른 데이터가 레플리카에서 한꺼번에 조회되는지 확인합니다.

* **Master 1**: `INSERT INTO products VALUES ('apple');`
* **Master 2**: `INSERT INTO products VALUES ('source2-1');`
* **Replica**: `SELECT * FROM products;` 결과로 두 데이터가 모두 출력됨을 확인.

### 2. Auto-Healing Test (자기 치유)

1. 레플리카 복제 중단: `STOP REPLICA;`
2. 마스터 서버들에 추가 데이터 입력.
3. 레플리카 복제 재개: `START REPLICA;`
4. **관찰**: GTID 기반으로 중단되었던 시점 이후의 트랜잭션을 추적하여 자동으로 동기화 완료.

### 3. Monitoring

```sql
-- 채널별 복제 상태 확인 (IO/SQL Running: Yes)
SHOW REPLICA STATUS\G

-- 통합된 GTID 집합 확인
SELECT @@GLOBAL.GTID_EXECUTED;

```

---

## 📝 Lessons Learned

* **GTID의 강력함**: 바이너리 로그 파일명과 위치(`Pos`)를 직접 계산하지 않아도 되는 복제의 편리함을 체득함.
* **Multi-Source Isolation**: 특정 마스터 서버의 장애나 채널 오류가 다른 마스터와의 복제에 영향을 주지 않음을 확인.
* **Conflict Handling**: 중복 키 발생 시 복제가 중단되는 원리를 이해하고 설계 단계에서 PK 분리의 중요성을 깨달음.
