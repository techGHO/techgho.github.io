---
layout: single
author_profile: true
title: "WebLogic 14c Domain 생성 및 관리서버(Admin Server) 기동 완전 가이드"
date: 2026-06-12 10:00:00 +0900
categories:
  - Middleware
tags:
  - WebLogic
  - WLST
  - Domain
  - AdminServer
  - Oracle Linux
  - WAS
  - Troubleshooting
toc: true
toc_sticky: true
header:
  overlay_image: https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80
  overlay_filter: 0.5
  show_overlay_excerpt: false
excerpt: "WebLogic 14c 바이너리 설치 이후, WLST(WebLogic Scripting Tool)를 활용하여 Domain을 자동으로 생성하고 관리서버를 기동하는 전 과정을 실무 관점에서 상세히 다룹니다."
---

## 1. 개요

[지난 포스트](https://techgho.github.io/middleware/weblogic-14c-silent-install-oracle-linux/)에서 Oracle Linux 8 환경에 WebLogic 14c 바이너리를 Silent 방식으로 설치했습니다. 바이너리 설치만으로는 실제 애플리케이션을 배포할 수 없습니다. 서비스를 운영하기 위해서는 반드시 **Domain**을 생성하고 **Admin Server**를 기동해야 합니다.

Domain은 WebLogic이 관리하는 논리적 단위로, Admin Server, Managed Server, 클러스터, 데이터소스, 보안 설정 등을 포함합니다. 본 포스트에서는 GUI 없이 **WLST(WebLogic Scripting Tool)**의 오프라인 스크립트 방식으로 Domain을 생성하고, 운영 환경에 맞는 JVM 튜닝과 systemd 서비스 등록까지 한 번에 구성하는 방법을 다룹니다.

---

## 2. 사전 준비 및 환경 확인

이전 포스트의 설치가 완료된 상태를 기준으로 합니다. `oracle` 계정으로 전환하여 작업합니다.

```bash
su - oracle
```

### 2.1. 환경 변수 재확인

WLST를 포함한 WebLogic 도구들은 `MW_HOME` 및 `WL_HOME` 기준으로 동작합니다.

```bash
# ~/.bash_profile에 아래 내용이 없으면 추가
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
export MW_HOME=/u01/app/oracle/middleware/wlserver
export WL_HOME=$MW_HOME
export PATH=$JAVA_HOME/bin:$MW_HOME/server/bin:$PATH

source ~/.bash_profile
```

### 2.2. WLST 동작 확인

WLST는 WebLogic 도메인 구성을 코드로 자동화할 수 있는 Jython 기반 CLI 도구입니다.

```bash
# WLST 실행 확인
$MW_HOME/common/bin/wlst.sh

# 정상이면 아래와 같은 프롬프트 출력
# Initializing WebLogic Scripting Tool (WLST) ...
# wls:/offline>
```

`exit()` 명령 또는 `Ctrl+D`로 종료합니다.

---

## 3. WLST 스크립트로 Domain 자동 생성

GUI 없이 Domain을 생성하는 가장 실무적인 방법입니다. 아래 스크립트를 `/u01/app/scripts/create_domain.py`로 저장합니다.

### 3.1. Domain 생성 스크립트 작성

```python
# /u01/app/scripts/create_domain.py
# WebLogic 14c Domain 자동 생성 WLST 스크립트

import os

# --- 설정 값 ---
domain_name   = 'base_domain'
domain_home   = '/u01/app/oracle/domains/' + domain_name
admin_name    = 'AdminServer'
admin_port    = 7001
admin_user    = 'weblogic'
admin_passwd  = 'Welcome1#'   # 운영 환경에서는 반드시 강력한 패스워드로 변경
template_path = os.environ['MW_HOME'] + '/common/templates/wls/wls.jar'

print("==> Domain 생성 시작: " + domain_home)

# 기본 WLS 템플릿 읽기
readTemplate(template_path)

# Domain 기본 설정
cd('Servers/AdminServer')
set('ListenAddress', '')
set('ListenPort',    admin_port)

# 관리자 계정 설정
cd('/')
cd('Security/base_domain/User/weblogic')
cmo.setPassword(admin_passwd)

# 도메인 저장
setOption('OverwriteDomain', 'true')
writeDomain(domain_home)
closeTemplate()

print("==> Domain 생성 완료: " + domain_home)
exit()
```

### 3.2. 스크립트 실행

```bash
mkdir -p /u01/app/scripts
mkdir -p /u01/app/oracle/domains

# WLST로 도메인 생성 스크립트 실행
$MW_HOME/common/bin/wlst.sh /u01/app/scripts/create_domain.py
```

실행이 성공하면 `/u01/app/oracle/domains/base_domain` 디렉터리 구조가 생성됩니다.

```bash
ls -l /u01/app/oracle/domains/base_domain/
# 주요 디렉터리
# bin/        - 기동/정지 스크립트
# config/     - Domain 설정 XML (config.xml)
# security/   - 보안 정책 파일
# servers/    - 서버별 로그·임시 파일
```

---

## 4. Admin Server 기동 전 JVM 튜닝

Admin Server를 기동하기 전에 JVM 옵션을 사전에 지정해야 합니다. WebLogic은 기동 스크립트에서 `USER_MEM_ARGS` 환경 변수를 우선 적용합니다.

### 4.1. setUserOverrides.sh 생성 (권장 방식)

기동 스크립트 직접 수정은 업그레이드 시 유실될 수 있어, 별도 오버라이드 파일을 사용합니다.

```bash
cat <<'EOF' > /u01/app/oracle/domains/base_domain/bin/setUserOverrides.sh
#!/bin/bash
# WebLogic JVM 튜닝 - setUserOverrides.sh
# startWebLogic.sh가 이 파일을 자동으로 source 합니다.

# Heap 설정 (운영 환경에 맞게 조정)
export USER_MEM_ARGS="-Xms512m -Xmx1024m -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m"

# G1GC 사용 (JDK 11 권장 GC)
export USER_MEM_ARGS="${USER_MEM_ARGS} -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# GC 로그 활성화 (트러블슈팅 필수)
export USER_MEM_ARGS="${USER_MEM_ARGS} -Xlog:gc*:file=/u01/app/oracle/domains/base_domain/servers/AdminServer/logs/gc.log:time,uptime:filecount=5,filesize=20m"

# 핵심 튜닝: /dev/random 대신 urandom 사용으로 기동 속도 개선
export USER_MEM_ARGS="${USER_MEM_ARGS} -Djava.security.egd=file:/dev/./urandom"

# 타임존 명시적 설정
export USER_MEM_ARGS="${USER_MEM_ARGS} -Duser.timezone=Asia/Seoul"

# WebLogic 내장 진단 활성화
export USER_MEM_ARGS="${USER_MEM_ARGS} -Dweblogic.StdoutSeverityLevel=Info"
EOF

chmod +x /u01/app/oracle/domains/base_domain/bin/setUserOverrides.sh
```

### 4.2. boot.properties 생성 (자동 로그인)

Admin Server 기동 시 콘솔에서 ID/PW를 매번 입력하지 않으려면 `boot.properties`를 생성합니다. 파일 내용은 최초 기동 시 자동으로 암호화됩니다.

```bash
mkdir -p /u01/app/oracle/domains/base_domain/servers/AdminServer/security

cat <<EOF > /u01/app/oracle/domains/base_domain/servers/AdminServer/security/boot.properties
username=weblogic
password=Welcome1#
EOF
```

---

## 5. Admin Server 기동 및 확인

### 5.1. 포그라운드 기동 (최초 확인용)

처음에는 포그라운드로 기동하여 로그를 직접 확인하는 것이 좋습니다.

```bash
cd /u01/app/oracle/domains/base_domain
./bin/startWebLogic.sh
```

기동 성공 시 출력되는 핵심 메시지:

```
<Jun 12, 2026 10:00:00,000 AM KST> <Notice> <WebLogicServer> <BEA-000360>
<Server started in RUNNING mode>
```

`RUNNING mode` 메시지가 출력되면 정상 기동된 것입니다.

### 5.2. 관리 콘솔 접속 확인

브라우저에서 아래 URL로 접속합니다.

```
http://<서버IP>:7001/console
```

- **ID**: `weblogic`
- **PW**: `Welcome1#`

방화벽이 설정된 경우 7001 포트를 먼저 개방합니다.

```bash
# firewalld 포트 허용
sudo firewall-cmd --permanent --add-port=7001/tcp
sudo firewall-cmd --reload
```

### 5.3. 백그라운드 기동

운영 환경에서는 `nohup`으로 백그라운드에서 기동합니다.

```bash
nohup ./bin/startWebLogic.sh > /u01/app/oracle/domains/base_domain/servers/AdminServer/logs/startWebLogic.out 2>&1 &

# 기동 상태 확인
tail -f /u01/app/oracle/domains/base_domain/servers/AdminServer/logs/startWebLogic.out
```

---

## 6. systemd 서비스 등록 (운영 환경 필수)

서버 재부팅 시 WebLogic이 자동으로 기동되도록 systemd 서비스로 등록합니다.

```bash
# root 계정으로 실행
sudo bash -c 'cat <<EOF > /etc/systemd/system/weblogic-admin.service
[Unit]
Description=Oracle WebLogic Admin Server (base_domain)
After=network.target

[Service]
Type=simple
User=oracle
Group=oinstall
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk"
Environment="MW_HOME=/u01/app/oracle/middleware/wlserver"
WorkingDirectory=/u01/app/oracle/domains/base_domain
ExecStart=/u01/app/oracle/domains/base_domain/bin/startWebLogic.sh
ExecStop=/u01/app/oracle/domains/base_domain/bin/stopWebLogic.sh
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
EOF'

# 서비스 등록 및 활성화
sudo systemctl daemon-reload
sudo systemctl enable weblogic-admin.service
sudo systemctl start  weblogic-admin.service

# 상태 확인
sudo systemctl status weblogic-admin.service
```

---

## 7. 실무 트러블슈팅

### Q1. 기동 후 콘솔에 접속이 안 되는 경우

```bash
# 1. 프로세스 확인
ps -ef | grep weblogic | grep -v grep

# 2. 리슨 포트 확인 (LISTEN 상태인지 체크)
ss -tnlp | grep 7001

# 3. 서버 로그에서 에러 확인
grep -i "ERROR\|Exception" \
  /u01/app/oracle/domains/base_domain/servers/AdminServer/logs/AdminServer.log | tail -30
```

### Q2. "서버가 STARTING 상태에서 멈추는 경우"

엔트로피 부족 또는 데이터베이스 연결 대기가 원인인 경우가 많습니다.

```bash
# 엔트로피 수준 확인
cat /proc/sys/kernel/random/entropy_avail
# 200 이하이면 setUserOverrides.sh의 urandom 설정 확인

# rngd 데몬으로 엔트로피 보충 (최후 수단)
sudo dnf install -y rng-tools
sudo systemctl enable --now rngd
```

### Q3. boot.properties가 암호화되지 않는 경우

`boot.properties` 파일 위치가 틀린 경우입니다. 서버명과 디렉터리 명이 일치해야 합니다.

```bash
# 올바른 경로 확인
ls /u01/app/oracle/domains/base_domain/servers/
# 출력된 디렉터리명(= 서버명)과 boot.properties 경로를 일치시킴
```

### Q4. systemd 서비스가 timeout으로 실패하는 경우

Admin Server는 기동에 시간이 걸릴 수 있습니다. `TimeoutStartSec` 값을 늘립니다.

```bash
sudo systemctl edit weblogic-admin.service
# [Service] 섹션에 아래 추가
# TimeoutStartSec=600
sudo systemctl daemon-reload
```

---

## 8. Domain 기동·정지 명령 정리

| 구분 | 명령 |
|------|------|
| Admin Server 기동 | `./bin/startWebLogic.sh` |
| Admin Server 정지 | `./bin/stopWebLogic.sh` |
| WLST 접속(온라인) | `$MW_HOME/common/bin/wlst.sh` → `connect('weblogic','passwd','t3://localhost:7001')` |
| 서비스 기동 | `sudo systemctl start weblogic-admin` |
| 서비스 정지 | `sudo systemctl stop weblogic-admin` |
| 서비스 상태 | `sudo systemctl status weblogic-admin` |

---

## 9. 다음 단계

Admin Server 기동까지 완료했습니다. 이제 실제 서비스를 올리기 위해서는 다음 단계가 필요합니다.

- **Managed Server 추가**: 애플리케이션을 실제로 서빙하는 서버 추가
- **JDBC 데이터소스 설정**: DB 연결 풀 구성
- **애플리케이션 배포(Deployment)**: WAR/EAR 배포 및 운영 중 무중단 재배포 전략

다음 포스트에서는 WLST를 활용한 Managed Server 추가 및 클러스터 구성 자동화를 다룰 예정입니다.
