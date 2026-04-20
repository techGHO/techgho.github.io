----
layout: single
author_profile: true
title: "WebLogic 14cR1(14.1.1) 완전 자동화 설치 및 최적화 가이드 (Oracle Linux / RHEL 8 기준)"
date: 2026-04-20 13:00:00 +0900
categories:
  - Middleware
tags:
  - WebLogic
  - Oracle Linux
  - RHEL
  - WAS
  - Troubleshooting
toc: true
toc_sticky: true
header:
  overlay_image: https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80
  overlay_filter: 0.5
---

## 1. 개요
본 포스팅에서는 엔터프라이즈 환경의 표준인 Oracle Linux 8 (RHEL 8 호환)에서 Oracle WebLogic Server 14c를 Silent Mode로 설치하는 과정을 다룹니다. GUI(X-Window) 환경이 지원되지 않는 폐쇄망 및 클라우드(OCI) 인스턴스 환경에서 스크립트 기반으로 인프라를 프로비저닝할 때 필수적인 절차입니다.

## 2. OS 사전 구성 및 필수 패키지 (Prerequisites)
RHEL 8 / Oracle Linux 8 환경에서는 과거 버전과 달리 누락된 기본 라이브러리들이 있으므로 설치 전 반드시 확인해야 합니다.

### 2.1. 필수 라이브러리 및 JDK 설치
WebLogic 14c 설치 시 GUI 인스톨러나 Silent 설치 모두 특정 라이브러리(`libnsl`)를 요구합니다. 이를 누락할 경우 설치 단계에서 JVM 에러가 발생합니다.

```bash
# 필수 의존성 패키지 및 OpenJDK 11 설치
sudo dnf update -y
sudo dnf install -y java-11-openjdk-devel libnsl libaio tar unzip

# Java 버전 확인
java -version
```

### 2.2. OS 계정 및 그룹 생성
보안 및 권한 분리를 위해 WAS 구동 전용 계정을 생성합니다.

```bash
sudo groupadd oinstall
sudo useradd -g oinstall -m oracle
sudo passwd oracle
```

### 2.3. 디렉터리 생성 및 권한 부여
오라클 제품군 설치 표준 권장 경로인 `/u01`을 기준으로 구성합니다.

```bash
# 엔진 설치 디렉터리 생성
sudo mkdir -p /u01/app/oracle/middleware
sudo mkdir -p /u01/app/oraInventory
sudo chown -R oracle:oinstall /u01

# oracle 계정으로 전환
su - oracle
```

### 2.4. 환경 변수 등록 (`~/.bash_profile`)
WebLogic 구동을 위해 필요한 JAVA_HOME을 등록합니다.

```bash
# ~/.bash_profile 하단에 추가
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
export PATH=$JAVA_HOME/bin:$PATH

# 적용
source ~/.bash_profile
```

## 3. WebLogic 14c Silent 설치
응답 파일(Response File)과 인벤토리 포인터 파일을 작성하여 백그라운드 설치를 진행합니다. 해당 파일들은 `oracle` 계정으로 작성합니다.

### 3.1. oraInst.loc 생성 (인벤토리 포인터)
오라클 인벤토리 위치를 지정합니다.

```bash
cat <<EOF > /u01/app/oraInst.loc
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
EOF
```

### 3.2. wls.rsp 생성 (응답 파일)
엔진이 설치될 경로와 설치 타입을 정의합니다.

```text
cat <<EOF > /u01/app/wls.rsp
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/u01/app/oracle/middleware/wlserver
INSTALL_TYPE=WebLogic Server
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
EOF
```

### 3.3. 설치 명령어 실행
오라클 홈페이지에서 다운로드 받은 Generic Installer jar 파일 위치에서 아래 명령을 실행합니다.

```bash
# 백그라운드 설치 실행
java -jar fmw_14.1.1.0.0_wls_lite_generic.jar -silent \
-responseFile /u01/app/wls.rsp \
-invPtrLoc /u01/app/oraInst.loc
```

설치가 성공적으로 완료되면 쉘 화면에 `The installation of Oracle Fusion Middleware 14.1.1.0.0 WebLogic Server and Coherence completed successfully.` 메시지가 출력됩니다.

## 4. 💡 실무 트러블슈팅 팁 (Troubleshooting)

### Q. 설치나 도메인 기동 시 속도가 비정상적으로 느린 경우?
Linux 환경에서 Java 기반의 WebLogic 기동 시 보안 난수 생성기(`/dev/random`)의 엔트로피 부족으로 인해 무한 대기(Hang) 현상이 발생할 수 있습니다. 

**해결 방안:** `java.security` 파일 또는 JVM 기동 옵션에 Non-blocking 난수 생성기인 `/dev/urandom`을 사용하도록 지정합니다.

```bash
# JVM 옵션 추가 예시
-Djava.security.egd=file:/dev/./urandom
```

이는 OCI의 Always Free 인스턴스와 같이 자원이 제한적인 클라우드 환경에서 기동 속도를 획기적으로 단축시키는 필수 튜닝 포인트입니다. 고단가 기술 키워드 중 하나이기도 합니다.

## 5. 다음 단계
기본 엔진(바이너리) 설치가 무사히 완료되었습니다. 다음 포스트에서는 실제 서비스를 올리기 위한 WebLogic Domain 구성 및 Admin Server 기동 방법에 대해 알아보겠습니다.
