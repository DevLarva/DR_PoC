# DR_PoC
# Site-to-Site VPN + MySQL Replication DR PoC (AWS ↔ On-Prem)

## 1. 프로젝트 개요

본 프로젝트는 AWS VPC 환경과 로컬 On-Prem 환경을  
**Site-to-Site VPN(StrongSwan, IPsec / IKEv2)** 으로 연결하고,

AWS에 위치한 **MySQL Primary(Source)** 와  
On-Prem에 위치한 **MySQL Replica** 간 **비동기 Replication** 을 구성한 뒤,

의도적으로 VPN 장애를 발생시켜 **DR(Disaster Recovery) 관점에서 데이터 복구 가능성**을 검증하는 PoC이다.

본 PoC의 목적은 고가용성(HA)이 아니라,

- 네트워크 단절 상황에서 Replication이 어떻게 동작하는지
- 장애 복구 후 데이터가 정상적으로 따라오는지
- 실제 운영 환경에서 발생할 수 있는 문제 포인트를 확인하는 데 있다.

---

## 2. 아키텍처 구성

### 구성 요소

### AWS

- EC2 (MySQL Primary)
- EC2 (StrongSwan VPN Endpoint)
- VPC CIDR: `10.20.0.0/16`
- EC2 Private IP (예시): `10.20.11.123`
- EC2 Public IP (예시): `3.36.94.130`

### On-Prem (Local VM / VMware Fusion)

- Ubuntu VM (MySQL Replica)
- Ubuntu VM (StrongSwan VPN Endpoint)
- Local CIDR: `192.168.64.0/24`
- VM Private IP (예시): `192.168.64.15`
- Public IP (예시): `112.146.120.193`

---

### 네트워크 연결 요약

- Site-to-Site VPN
  - `10.20.0.0/16` ↔ `192.168.64.0/24`
- MySQL Replication
  - AWS → On-Prem
  - Async Replication (TCP 3306)

---

## 3. 사전 준비 사항

### 필수 포트

#### IPsec VPN
- UDP 500 (IKE)
- UDP 4500 (NAT-T)

#### MySQL
- TCP 3306

---

## 4. VPN 구성 (StrongSwan)

### VPN 상태 확인

AWS / On-Prem 양쪽에서 다음 명령으로 VPN 상태를 확인한다.

```bash
sudo ipsec statusall
정상 연결 시 다음과 같은 상태가 확인된다.

IKE_SA: ESTABLISHED

CHILD_SA: INSTALLED

Traffic Selector
10.20.0.0/16 === 192.168.64.0/24

5. MySQL Replication 구성
역할
AWS: MySQL Primary (Source)

On-Prem: MySQL Replica

Replica 상태 확인:

SHOW REPLICA STATUS\G
정상 상태 기준:

Replica_IO_Running: Yes

Replica_SQL_Running: Yes

Seconds_Behind_Source 값이 정상 범위

6. 데이터 검증용 테스트 테이블
AWS(MySQL Primary)에서 테스트용 데이터 생성:

CREATE DATABASE IF NOT EXISTS testdb;
USE testdb;

CREATE TABLE IF NOT EXISTS ping (
  id INT AUTO_INCREMENT PRIMARY KEY,
  msg VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO ping(msg) VALUES ('replication_test_1');
Replica(On-Prem)에서 확인:

SELECT * FROM testdb.ping;
7. 장애 테스트 시나리오 (DR 관점)
테스트 목적
VPN 장애 발생 시 Replication 동작 확인

장애 복구 후 데이터 정합성 확인

네트워크 단절 상황에서도 데이터 유실 없이 복구 가능한지 검증

8. 장애 테스트: VPN Down → Recover
Step 1) VPN Down (AWS 기준)
sudo ipsec down onprem-aws
Replica 상태 확인:

SHOW REPLICA STATUS\G
일반적인 장애 상태 예시:

Replica_IO_Running: Connecting

Replica_SQL_Running: Yes

Seconds_Behind_Source: NULL

Last_IO_Error: Can't connect ... (110)

VPN 단절 상태에서는 Replica가 Source에 접근할 수 없으므로
IO Thread가 Connecting 상태로 유지되는 것이 정상 동작이다.

Step 2) 장애 중 데이터 입력
VPN이 끊긴 상태에서 AWS(MySQL Primary)에 데이터 추가:

INSERT INTO ping(msg) VALUES ('replication_test_while_vpn_down');
이 단계는 장애 중 발생한 데이터가 복구 후 반영되는지를 확인하기 위함이다.

Step 3) VPN 복구
테스트 과정에서 한쪽만 VPN을 올릴 경우
peer not responding 문제가 발생하는 경우가 있었으며,

양쪽 VPN을 완전히 정리한 후 다시 올리는 방식이 가장 안정적이었다.

권장 복구 순서
# AWS
sudo ipsec down onprem-aws

# On-Prem
sudo ipsec down onprem-aws
sudo ipsec up onprem-aws

# AWS
sudo ipsec up onprem-aws
복구 확인:

sudo ipsec statusall
ESTABLISHED

INSTALLED

Security Associations (1 up)

Step 4) 복구 후 데이터 동기화 확인
On-Prem(MySQL Replica):

SELECT * FROM testdb.ping;
확인 포인트:

장애 이전 데이터

장애 중 입력된 데이터

복구 이후 데이터

모두 정상적으로 조회되면 DR 시나리오 검증 성공이다.

9. 트러블슈팅 정리
이슈 1) peer not responding
증상: IKE_SA_INIT 재전송 반복

원인: 한쪽 세션만 정리된 상태에서 재연결 시도

해결: 양쪽 VPN 완전 종료 후 재연결

이슈 2) Replication이 Connecting 상태에서 복구되지 않음
VPN이 정상화되지 않은 상태에서 Replication만 확인한 경우 발생

VPN 정상화 후 필요 시 Replica 재시작

STOP REPLICA;
START REPLICA;
10. 결과 요약
Site-to-Site VPN 기반 AWS ↔ On-Prem 연결 검증 완료

MySQL Replication 정상 동작 확인

VPN 장애 시 Replication IO Thread 중단 확인

VPN 복구 후 자동 재연결 및 데이터 catch-up 확인

장애 중 생성된 데이터도 복구 후 정상 반영됨

본 PoC를 통해 네트워크 단절 상황에서도 데이터 복구가 가능한 DR 아키텍처임을 검증했다.


