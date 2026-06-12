---
layout: single
author_profile: true
title: "JBoss EAP 7.4 완전 설치 가이드 (Oracle Linux 8 / RHEL 8 기준)"
date: 2026-06-13 10:00:00 +0900
categories:
  - Middleware
tags:
  - JBoss
  - Oracle Linux
  - RHEL
  - WAS
  - Middleware
toc: true
toc_sticky: true
header:
  overlay_image: https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80
  overlay_filter: 0.5
  show_overlay_excerpt: false
excerpt: "Oracle Linux 8 / RHEL 8 환경에서 JBoss EAP 7.4를 다운로드부터 systemd 서비스 등록, 관리 콘솔 접근까지 현업 엔지니어 관점으로 단계별 완전 설치 가이드를 제공합니다."
---

## 1. 개요

Red Hat JBoss Enterprise Application Platform(EAP) 7.4는 Jakarta EE 8 스펙을 완전히 지원하는 엔터프라이즈급 Java 애플리케이션 서버입니다. WildFly 23을 기반으로 하며, 클러스터링, HA(고가용성), 트랜잭션 관리 등 대규모 운영 환경에 최적화된 기능을 제공합니다.

본 가이드에서는 GUI(X-Window)가 없는 Oracle Linux 8 / RHEL 8 서버 환경에서 JBoss EAP 7.4를 Red Hat 포털에서 다운로드하여 설치하고, 전용 OS 계정 구성, 환경 변수 설정, systemd 서비스 등록, 방화벽 설정, 관리 콘솔 접근까지 실무에서 바로 적용할 수 있는 절차를 단계별로 안내합니다.

## 2. 사전 요구사항

### 2.1. 시스템 요구사항 확인

JBoss EAP 7.4의 공식 지원 플랫폼은 다음과 같습니다.

| 항목 | 요구사항 |
|------|---------|
| OS | Oracle Linux 8.x / RHEL 8.x |
| JDK | OpenJDK 11 또는 Oracle JDK 11 (최소) |
| 메모리 | 최소 2GB RAM (운영 환경 4GB 이상 권장) |
| 디스크 | 최소 3GB 여유 공간 |
| Architecture | x86_64 |

```bash
# 시스템 기본 정보 확인
cat /etc/os-release
uname -m
free -h
df -h /opt
```

### 2.2. 필수 패키지 설치

```bash
# 시스템 업데이트 및 필수 패키지 설치
sudo dnf update -y
sudo dnf install -y \
  java-11-openjdk-devel \
  unzip \
  curl \
  wget \
  net-tools \
  lsof

# Java 버전 확인
java -version
# 출력 예시: openjdk version "11.0.22" 2024-01-16 LTS

# JAVA_HOME 확인
dirname $(dirname $(readlink -f $(which java)))
```

## 3. OS 계정 및 디렉터리 구성

보안 원칙에 따라 WAS 전용 계정을 생성하여 최소 권한으로 운영합니다. root 계정으로 JBoss를 직접 구동하는 것은 보안 위협이므로 반드시 전용 계정을 사용해야 합니다.

### 3.1. 전용 OS 계정 생성

```bash
# jboss 그룹 및 계정 생성
sudo groupadd -g 1100 jboss
sudo useradd -u 1100 -g jboss -m -d /home/jboss -s /bin/bash jboss
sudo passwd jboss

# 계정 확인
id jboss
```

### 3.2. 설치 디렉터리 생성

```bash
# JBoss EAP 설치 표준 경로 생성
sudo mkdir -p /opt/jboss/eap74
sudo mkdir -p /opt/jboss/logs
sudo mkdir -p /opt/jboss/deployments

# 소유권 및 권한 설정
sudo chown -R jboss:jboss /opt/jboss
sudo chmod 755 /opt/jboss
```

## 4. JBoss EAP 7.4 다운로드

### 4.1. Red Hat 포털에서 다운로드

JBoss EAP는 **Red Hat Customer Portal** (https://access.redhat.com/jbossnetwork/restricted/listSoftware.html) 에서 다운로드합니다. 유효한 Red Hat 구독이 필요합니다.

다운로드할 파일: `jboss-eap-7.4.0.zip` (약 285MB)

```bash
# 다운로드한 파일을 서버로 전송 (로컬 → 서버)
scp jboss-eap-7.4.0.zip oracle@<서버IP>:/tmp/

# 또는 wget으로 직접 다운로드 (Red Hat CDN 토큰 필요)
# wget --auth-no-challenge \
#   --http-user=<RHN_USERNAME> \
#   --http-password=<RHN_PASSWORD> \
#   "https://access.cdn.redhat.com/content/origin/files/.../jboss-eap-7.4.0.zip" \
#   -O /tmp/jboss-eap-7.4.0.zip
```

### 4.2. 압축 해제 및 배치

```bash
# jboss 계정으로 전환
sudo su - jboss

# 설치 파일 압축 해제
cd /opt/jboss
unzip /tmp/jboss-eap-7.4.0.zip -d /opt/jboss/eap74/

# 압축 해제 후 디렉터리 구조 확인
ls -la /opt/jboss/eap74/
# 예상 출력:
# jboss-eap-7.4/
#   ├── bin/          # 실행 스크립트
#   ├── domain/       # 도메인 모드 설정
#   ├── standalone/   # 독립 실행 모드 설정
#   ├── modules/      # 모듈 라이브러리
#   └── welcome-content/

# JBOSS_HOME 심볼릭 링크 생성 (버전 관리 편의)
ln -sfn /opt/jboss/eap74/jboss-eap-7.4 /opt/jboss/current
ls -la /opt/jboss/current
```

## 5. 환경 변수 설정

```bash
# jboss 계정의 .bash_profile에 환경 변수 추가
cat >> /home/jboss/.bash_profile << 'EOF'

# ==== JBoss EAP Environment Variables ====
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
export JBOSS_HOME=/opt/jboss/current
export PATH=$JAVA_HOME/bin:$JBOSS_HOME/bin:$PATH

# JVM 힙 메모리 설정 (운영 환경에 맞게 조정)
export JAVA_OPTS="-Xms512m -Xmx2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m"
export JAVA_OPTS="$JAVA_OPTS -Djava.net.preferIPv4Stack=true"
export JAVA_OPTS="$JAVA_OPTS -Djboss.modules.system.pkgs=org.jboss.byteman"
EOF

# 환경 변수 적용 확인
source /home/jboss/.bash_profile
echo "JBOSS_HOME: $JBOSS_HOME"
echo "JAVA_HOME: $JAVA_HOME"
$JBOSS_HOME/bin/standalone.sh --version
```

## 6. 관리자 계정 생성

JBoss EAP 7.4는 기본적으로 관리 콘솔 접근을 위한 사용자를 별도로 생성해야 합니다.

```bash
# jboss 계정으로 관리자 추가
$JBOSS_HOME/bin/add-user.sh

# 대화형 프롬프트 예시:
# What type of user do you wish to add?
#  a) Management User (mgmt-users.properties)
#  b) Application User (application-users.properties)
# (default a): a                          ← 'a' 입력
#
# Enter the details of the new user to add.
# Realm (ManagementRealm): [Enter]
# Username: admin
# Password: Admin@1234!
# Re-enter Password: Admin@1234!
# What groups do you want this user to belong to? : [Enter]
# Is this new user going to be used for one AS process to connect to another AS process?
# (yes/no): no

# 비대화형 방식 (스크립트 자동화 시 사용)
echo "" | $JBOSS_HOME/bin/add-user.sh -u admin -p 'Admin@Secure2024!' -g SuperUser -s
```

## 7. JBoss EAP 기동 및 확인

### 7.1. Standalone 모드로 기동

```bash
# jboss 계정으로 Standalone 모드 기동
$JBOSS_HOME/bin/standalone.sh &

# 로그에서 기동 완료 메시지 확인
tail -f $JBOSS_HOME/standalone/log/server.log

# 정상 기동 시 출력되는 메시지 예시:
# ... INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: JBoss EAP 7.4.0.GA
# ... INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0060: Http management interface listening on http://127.0.0.1:9990/management
# ... INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0051: Admin console listening on http://127.0.0.1:9990
# ... INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: JBoss EAP 7.4.0.GA (WildFly Core 15.0.1.Final-redhat-00001) started in 8643ms

# 프로세스 확인
ps -ef | grep jboss | grep -v grep

# 포트 확인 (8080: HTTP, 9990: 관리 콘솔)
ss -tlnp | grep -E '8080|9990'
```

### 7.2. 네트워크 인터페이스 바인딩 설정

기본 설정에서는 127.0.0.1에만 바인딩됩니다. 외부 접근이 필요한 경우 바인딩 주소를 변경합니다.

```bash
# 외부 접근 가능하도록 실행 시 옵션 지정
$JBOSS_HOME/bin/standalone.sh \
  -b 0.0.0.0 \
  -bmanagement 0.0.0.0 &

# 또는 standalone.xml에서 영구 설정
# <interfaces>
#   <interface name="management"><inet-address value="${jboss.bind.address.management:0.0.0.0}"/></interface>
#   <interface name="public"><inet-address value="${jboss.bind.address:0.0.0.0}"/></interface>
# </interfaces>
```

## 8. systemd 서비스 등록

운영 환경에서는 서버 재기동 시 자동 시작을 위해 systemd 서비스로 등록합니다.

```bash
# systemd 서비스 파일 생성 (root 권한 필요)
sudo tee /etc/systemd/system/jboss-eap.service << 'EOF'
[Unit]
Description=JBoss EAP 7.4 Application Server
After=network.target
After=syslog.target

[Service]
Type=simple
User=jboss
Group=jboss

# 환경 변수 파일 참조
EnvironmentFile=-/etc/jboss-eap/jboss-eap.conf

# JAVA_HOME 명시 설정
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk"
Environment="JBOSS_HOME=/opt/jboss/current"
Environment="JBOSS_LOG_DIR=/opt/jboss/logs"

# 외부 접속 허용 (0.0.0.0 바인딩)
ExecStart=/opt/jboss/current/bin/standalone.sh \
  -b 0.0.0.0 \
  -bmanagement 0.0.0.0 \
  -c standalone.xml

ExecStop=/opt/jboss/current/bin/jboss-cli.sh \
  --connect \
  --command=":shutdown"

# 프로세스 그룹 킬 (강제 종료 방지)
KillMode=process
KillSignal=SIGTERM

# 비정상 종료 시 재시작 정책
Restart=on-failure
RestartSec=60

# 파일 제한 설정
LimitNOFILE=102642

[Install]
WantedBy=multi-user.target
EOF

# systemd 데몬 리로드 및 서비스 활성화
sudo systemctl daemon-reload
sudo systemctl enable jboss-eap.service
sudo systemctl start jboss-eap.service

# 서비스 상태 확인
sudo systemctl status jboss-eap.service
```

## 9. 방화벽 설정

```bash
# firewalld를 사용하는 경우 포트 개방
sudo firewall-cmd --permanent --add-port=8080/tcp   # HTTP 서비스 포트
sudo firewall-cmd --permanent --add-port=8443/tcp   # HTTPS 서비스 포트
sudo firewall-cmd --permanent --add-port=9990/tcp   # 관리 콘솔 포트
sudo firewall-cmd --reload

# 개방된 포트 확인
sudo firewall-cmd --list-ports

# iptables를 사용하는 경우
# sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
# sudo iptables -I INPUT -p tcp --dport 9990 -j ACCEPT
# sudo service iptables save
```

## 10. 관리 콘솔 접근 및 기본 설정

### 10.1. 관리 콘솔 접근

웹 브라우저에서 다음 URL로 접근합니다.

```
http://<서버IP>:9990/console
```

또는 CLI(jboss-cli.sh)를 사용합니다.

```bash
# jboss-cli를 통한 원격 관리
$JBOSS_HOME/bin/jboss-cli.sh --connect --controller=127.0.0.1:9990

# 서버 상태 확인
:read-attribute(name=server-state)
# 출력: {"outcome" => "success", "result" => "running"}

# 배포된 애플리케이션 목록
deployment-info

# 서버 종료 (graceful shutdown)
:shutdown(timeout=60)
```

### 10.2. 샘플 애플리케이션 배포 테스트

```bash
# JBoss 기본 제공 샘플 WAR 배포
$JBOSS_HOME/bin/jboss-cli.sh --connect \
  --command="deploy $JBOSS_HOME/welcome-content/index.html"

# 또는 deployments 디렉터리에 파일 복사 (auto-deploy)
cp /path/to/myapp.war $JBOSS_HOME/standalone/deployments/
# .dodeploy 마커 파일이 자동 생성되고 배포 완료 시 .deployed로 변경됨

# 배포 상태 확인
ls -la $JBOSS_HOME/standalone/deployments/*.deployed
```

## 11. 트러블슈팅

### 11.1. 기동 시 java.lang.OutOfMemoryError

```bash
# 힙 메모리 부족 시 증상: 로그에 "java.lang.OutOfMemoryError: Java heap space"
# 해결: standalone.conf에서 JVM 메모리 설정 조정

vi $JBOSS_HOME/bin/standalone.conf

# 아래 JAVA_OPTS 항목 수정:
# 기존: JAVA_OPTS="-Xms64m -Xmx512m ..."
# 변경: JAVA_OPTS="-Xms512m -Xmx2g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m ..."

# GC 로그 추가로 메모리 사용 추이 분석
JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:file=/opt/jboss/logs/gc.log:time,uptime:filecount=5,filesize=50m"
```

### 11.2. 포트 충돌 오류

```bash
# 증상: "Address already in use"
# 포트 사용 프로세스 확인
sudo lsof -i :8080
sudo ss -tlnp | grep 8080

# JBoss 오프셋으로 포트 충돌 회피
$JBOSS_HOME/bin/standalone.sh -Djboss.socket.binding.port-offset=100
# 결과: HTTP 8180, 관리 콘솔 10090으로 기동
```

### 11.3. 관리 콘솔 접근 불가

```bash
# 원인 1: 방화벽 차단
sudo firewall-cmd --list-ports | grep 9990

# 원인 2: 관리 인터페이스 바인딩 주소 확인
grep -A5 'management' $JBOSS_HOME/standalone/configuration/standalone.xml | grep inet-address

# 원인 3: 관리자 계정 미등록
cat $JBOSS_HOME/standalone/configuration/mgmt-users.properties
# 내용이 비어있으면 add-user.sh로 계정 추가 필요
```

### 11.4. Deployment Failure 분석

```bash
# 배포 실패 원인 로그 분석
grep -E "ERROR|FAIL|Exception" $JBOSS_HOME/standalone/log/server.log | tail -50

# 배포 상태 마커 파일 확인
ls -la $JBOSS_HOME/standalone/deployments/
# .failed  → 배포 실패 (상세 원인은 server.log에 기록)
# .isdeploying → 배포 진행 중
# .deployed → 배포 성공

# 실패한 배포 강제 제거
$JBOSS_HOME/bin/jboss-cli.sh --connect \
  --command="undeploy myapp.war --keep-content"
```

## 12. 마무리

JBoss EAP 7.4의 설치와 기본 구성이 완료되었습니다. 운영 환경 적용 전 아래 추가 작업을 반드시 수행하세요.

- **보안 강화**: 관리 콘솔 포트(9990)는 내부 네트워크에서만 접근 가능하도록 IP 제한 설정
- **SSL/TLS 구성**: HTTPS(8443) 인증서 설정 및 HTTP → HTTPS 리다이렉트 설정
- **로그 로테이션**: `$JBOSS_HOME/standalone/configuration/logging.properties`에서 최대 보관 일수 설정
- **모니터링 연동**: Prometheus JMX Exporter 또는 Jolokia를 통한 메트릭 수집 구성
- **클러스터링**: 고가용성이 필요한 경우 Domain Mode 또는 mod_cluster를 통한 HA 구성 검토
