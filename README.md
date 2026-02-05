
# Hybrid Cloud DR PoC: Site-to-Site VPN 기반 MySQL Replication

## 목차

- [1. 개요](#1-개요)
- [2. 아키텍처 구성도](#2-아키텍처-구성도)
    - [2.1 아키텍처 구성 요소별 역할 분석](#21-아키텍처-구성-요소별-역할-분석)
- [3. 환경 정보](#3-환경-정보)
    - [시스템 환경](#시스템-환경)
    - [네트워크 구성](#네트워크-구성)
- [4. Site-to-Site VPN 구성](#4-site-to-site-vpn-구성)
    - [4.1 VPN 구성 개요](#41-vpn-구성-개요)
    - [4.2 VPN 상태 확인](#42-vpn-상태-확인)
- [5. MySQL Replication 구성](#5-mysql-replication-구성)
    - [5.1 Replication 역할 정의](#51-replication-역할-정의)
    - [5.2 Replication 상태 확인 및 데이터 생성](#52-replication-상태-확인-및-데이터-생성)
- [6. DR 테스트 시나리오](#6-dr-테스트-시나리오)
    - [6.1 테스트 목적](#61-테스트-목적)
    - [6.2 VPN 장애 발생 및 상태 확인](#62-vpn-장애-발생-및-상태-확인)
    - [6.3 장애 중 데이터 생성](#63-장애-중-데이터-생성)
    - [6.4 VPN 복구 절차](#64-vpn-복구-절차)
    - [6.5 AWS VPN 재시작](#65-aws-vpn-재시작)
    - [6.6 복구 확인](#66-복구-확인)
    - [6.7 복구 후 데이터 동기화 확인](#67-복구-후-데이터-동기화-확인)
- [7. 트러블슈팅 기록](#7-트러블슈팅-기록)
- [8. 결과 요약](#8-결과-요약)

---

## 1. 개요

본 문서는 AWS 환경과 로컬 On-Prem 환경을 **Site-to-Site VPN(StrongSwan, IPsec / IKEv2)**으로 연결하고, AWS에 위치한 **MySQL Primary(Source)**와 On-Prem 환경의 **MySQL Replica** 간 **비동기 Replication**을 구성하여 네트워크 장애 발생 시 **DR(Disaster Recovery) 관점에서 데이터 복구 가능 여부를 검증**한 PoC 프로젝트입니다.

**주요 목적:**

- VPN 장애 시 Replication 동작 상태 확인
- VPN 복구 후 Replication 자동 재연결 여부 검증
- 장애 중 발생한 데이터의 정합성 유지 여부 확인

---

## 2. 아키텍처 구성도

<img src="DR_PoC/dr-arc.png" width="700" alt="아키텍처 구성도">


### 2.1 아키텍처 구성 요소별 역할 분석

**AWS 영역**

- **EC2 (MySQL Primary):** 서비스 기준 데이터가 저장되는 Source DB입니다.
- **EC2 (StrongSwan VPN Endpoint):** On-Prem 환경과의 Site-to-Site VPN 종단점 역할을 수행합니다.
- **VPC CIDR:** `10.20.0.0/16`

**On-Prem 영역 (VMware Fusion)**

- **Ubuntu VM (MySQL Replica):** AWS MySQL Primary로부터 데이터를 복제하여 저장합니다.
- **Ubuntu VM (StrongSwan VPN Endpoint):** AWS와의 VPN 연결을 담당하는 로컬 종단점입니다.
- **Local CIDR:** `192.168.64.0/24`

**데이터 흐름**

- **VPN 터널:** `10.20.0.0/16` ↔ `192.168.64.0/24`
- **MySQL Replication:** AWS → On-Prem (Async 방식, TCP 3306 포트 사용)

---

## 3. 환경 정보

### 시스템 환경

- **Cloud:** AWS EC2
- **On-Prem:** VMware Fusion (Local VM)
- **OS:** Ubuntu
- **VPN:** StrongSwan (IPsec / IKEv2)
- **DB:** MySQL

### 네트워크 구성

| **구분**                | **IP / CIDR**   | **설명**                |
| --------------------- | --------------- | --------------------- |
| **AWS VPC**           | 10.20.0.0/16    | AWS 내부 네트워크 대역        |
| **On-Prem**           | 192.168.64.0/24 | 로컬 VM 네트워크 대역         |
| **AWS Public IP**     | 3.36.xxx.xxx    | AWS 측 VPN Endpoint IP |
| **On-Prem Public IP** | 112.xxx.xxx.xxx | 로컬 측 VPN Endpoint IP  |

---

## 4. Site-to-Site VPN 구성

### 4.1 VPN 구성 개요

AWS ↔ On-Prem 간 직접 통신을 위해 StrongSwan 기반 IPsec Site-to-Site VPN을 구성하였습니다.

- **사용 포트:** UDP 500 (IKE), UDP 4500 (NAT-T)

### 4.2 VPN 상태 확인

```bash
$ sudo ipsec statusall
```

**정상 연결 시 확인되는 상태:**

- **IKE_SA:** ESTABLISHED
- **CHILD_SA:** INSTALLED
- **통신 대역:** 10.20.0.0/16 === 192.168.64.0/24

<img src="DR_PoC/conn.png" width="700" alt="연결 상태">

---

## 5. MySQL Replication 구성

### 5.1 Replication 역할 정의

- **AWS:** MySQL Primary (Source)
- **On-Prem:** MySQL Replica
- **복제 방식:** 비동기(Async) 방식

### 5.2 Replication 상태 확인 및 데이터 생성


```sql
- 복제 상태 확인
SHOW REPLICA STATUS\G
```
**정상 연결 시 확인되는 상태:**

- **Replica_IO_Running: Yes**
- **Replica_SQL_Running: Yes**
- **Seconds_Behind_Source: 정상 범위 확인**

**테스트 데이터 생성:**
```sql
CREATE DATABASE testdb;
USE testdb;

CREATE TABLE ping (
  id INT AUTO_INCREMENT PRIMARY KEY,
  msg VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO ping(msg) VALUES ('replication_test_1');
```
---

## 6. DR 테스트 시나리오

### 6.1 테스트 목적

- VPN 장애 발생 시 Replication 상태 변화 확인
- VPN 복구 후 데이터 정합성 및 복구 여부 검증

### 6.2 VPN 장애 발생 및 상태 확인


```bash
$ sudo ipsec down onprem-aws
```

**Replica 상태 예시:**

- **Replica_IO_Running:** Connecting
- **Replica_SQL_Running:** Yes
- **Seconds_Behind_Source:** NULL
- *네트워크 단절로 인해 Source에 접근할 수 없는 정상적인 장애 상태임.*

### 6.3 장애 중 데이터 생성

```sql
# AWS Primary에서 실행

INSERT INTO ping(msg) VALUES('replication_test_while_vpn_down');
```
### 6.4 VPN 복구 절차

테스트 결과, 한쪽만 세션을 올리는 것보다 양쪽의 세션을 완전히 정리 후 재연결하는 것이 가장 안정적이었습니다.

```bash
# 1. On-Prem VPN 재시작
$ sudo ipsec down onprem-aws
$ sudo ipsec up onprem-aws
```
### 6.5 AWS VPN 재시작
```bash
$ sudo ipsec down onprem-aws
$ sudo ipsec up onprem-aws
```
### 6.6 복구 확인
```bash
$ sudo ipsec statusall
```
### 6.7 복구 후 데이터 동기화 확인

```sql
SELECT * FROM testdb.ping;
```
- 장애 이전 데이터 확인 완료
- 장애 중 생성된 데이터 확인 완료
- 복구 이후 데이터 확인 완료
- **모든 데이터가 정상적으로 Replica에 반영됨을 확인하였습니다.**


<img src="DR_PoC/test-ping.png" width="700" alt="db 상태">

---

## 7. 트러블슈팅 기록

| **현상** | **원인** | **해결 방법** |
| --- | --- | --- |
| **peer not responding** | 한쪽 VPN 세션만 정리된 상태에서 재연결 시도 | 양쪽 VPN을 완전히 down 후 다시 up 진행 |
| **Connecting 상태 유지** | VPN 정상화 이전 Replication 프로세스 고착 | `STOP REPLICA;` 실행 후 `START REPLICA;`로 재시작 |

---

## 8. 결과 요약

- **VPN 연결:** AWS ↔ On-Prem Site-to-Site VPN 연결 검증 완료
- **Replication:** MySQL Replication 정상 동작 및 비동기 복제 확인
- **장애 대응:** VPN 단절 시 IO Thread 중단 및 복구 후 자동 재연결(Catch-up) 검증
- **데이터 정합성:** 장애 상황에서 발생한 데이터 누락 없이 정상 복구됨을 확인
