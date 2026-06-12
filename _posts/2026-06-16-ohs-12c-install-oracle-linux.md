---
layout: single
author_profile: true
title: "Oracle HTTP Server (OHS) 12c 설치 가이드 (Oracle Linux 8 기준)"
date: 2026-06-16 10:00:00 +0900
categories:
  - Middleware
tags:
  - OHS
  - Oracle Linux
  - Web
  - Middleware
  - Oracle
toc: true
toc_sticky: true
header:
  overlay_image: https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80
  overlay_filter: 0.5
  show_overlay_excerpt: false
excerpt: "Oracle Linux 8에서 Oracle HTTP Server(OHS) 12c를 FMW 다운로드부터 oraInst.loc, 응답 파일 기반 Silent 설치, OPatch 적용, 기동·정지, mod_wl_ohs WebLogic 프록시 구성까지 실무 완전 가이드를 제공합니다."
---

## 1. 개요

Oracle HTTP Server(OHS) 12c는 Apache HTTP Server 2.4를 기반으로 Oracle에서 커스터마이징한 엔터프라이즈 웹 서버입니다. WebLogic Server와의 긴밀한 통합을 위해 `mod_wl_ohs` 모듈을 기본 제공하며, Oracle Access Manager(OAM), Oracle Identity Manager(OIM) 등 Oracle Fusion Middleware 제품군과의 연동에 필수적으로 사용됩니다.

Oracle FMW 12.2.1.4.0 기반의 OHS를 Oracle Linux 8에서 GUI 없이 Silent 설치하는 전 과정을 다룹니다. 이 가이드의 설치 방식은 폐쇄망 환경, OCI/AWS 클라우드 인스턴스, CI/CD 파이프라인에서의 자동화 구성에도 동일하게 적용됩니다.

## 2. 사전 요구사항

### 2.1. 시스템 요구사항

| 항목 | 요구사항 |
|------|---------|
| OS | Oracle Linux 8.x / RHEL 8.x |
| JDK | Oracle JDK 8u271+ 또는 OpenJDK 8 (OHS 12c는 JDK 8 권장) |
| 메모리 | 최소 4GB RAM |
| 디스크 | 최소 10GB 여유 공간 (`/u01` 기준) |
| Swap | 최소 4GB 이상 |
| 권한 | 설치는 비root 계정(oracle), sudo 권한 필요 |

### 2.2. 필수 패키지 설치

```bash
# 필수 의존성 패키지
sudo dnf install -y \
  java-1.8.0-openjdk-devel \
  libaio \
  libnsl \
  libnsl2 \
  libstdc++ \
  glibc \
  unzip \
  binutils \
  make \
  ksh

# 설치 후 확인
java -version
# openjdk version "1.8.0_xxx"

rpm -qa | grep -E "libaio|libnsl|libstdc"
```

### 2.3. 커널 파라미터 설정

```bash
# /etc/sysctl.conf에 추가
sudo tee -a /etc/sysctl.conf << 'EOF'

# Oracle FMW 권장 커널 파라미터
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
EOF

# 설정 즉시 적용
sudo sysctl -p

# /etc/security/limits.conf에 ulimit 설정
sudo tee -a /etc/security/limits.conf << 'EOF'
oracle soft nofile 4096
oracle hard nofile 65536
oracle soft nproc  2047
oracle hard nproc  16384
oracle soft stack  10240
oracle hard stack  32768
EOF
```

## 3. OS 계정 및 디렉터리 구성

```bash
# Oracle 전용 그룹 및 계정 생성
sudo groupadd -g 1010 oinstall
sudo groupadd -g 1011 dba
sudo useradd -u 1010 -g oinstall -G dba -m -d /home/oracle -s /bin/bash oracle
sudo passwd oracle

# Oracle FMW 표준 디렉터리 구조 생성
sudo mkdir -p /u01/app/oracle/product/fmw12214
sudo mkdir -p /u01/app/oracle/config/domains
sudo mkdir -p /u01/app/oraInventory

# 소유권 설정
sudo chown -R oracle:oinstall /u01
sudo chmod -R 775 /u01

# oracle 계정으로 전환
sudo su - oracle
```

## 4. FMW 설치 파일 다운로드

### 4.1. Oracle eDelivery / My Oracle Support에서 다운로드

OHS 12c는 Oracle Fusion Middleware Infrastructure 12.2.1.4.0의 일부로 제공됩니다. [Oracle Software Delivery Cloud](https://edelivery.oracle.com) 또는 [My Oracle Support](https://support.oracle.com)에서 다운로드합니다.

필요한 파일:
- `fmw_12.2.1.4.0_ohs_linux64.bin` — OHS 단독 설치용 (약 900MB)
- 또는 `fmw_12.2.1.4.0_infrastructure.jar` + `fmw_12.2.1.4.0_ohs.jar` — FMW 포함 설치

```bash
# 다운로드한 파일을 /tmp로 전송
ls -lh /tmp/fmw_12.2.1.4.0_ohs_linux64.bin
# 실행 권한 부여
chmod +x /tmp/fmw_12.2.1.4.0_ohs_linux64.bin
```

## 5. oraInst.loc 파일 설정

Oracle Inventory 위치를 지정하는 파일입니다. 시스템에 Oracle 제품이 처음 설치되는 경우 생성합니다.

```bash
# oracle 계정으로 oraInst.loc 생성
cat > /home/oracle/oraInst.loc << 'EOF'
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
EOF

# 또는 전체 경로로
cat > /etc/oraInst.loc << 'EOF'
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
EOF

sudo chown oracle:oinstall /etc/oraInst.loc
```

## 6. Silent 설치를 위한 응답 파일 작성

### 6.1. OHS 설치 응답 파일 생성

```bash
# oracle 계정으로 응답 파일 작성
cat > /home/oracle/ohs_install.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
# Oracle Home 경로
ORACLE_HOME=/u01/app/oracle/product/fmw12214/ohs

# 설치 유형 선택
# 1: Complete Install (OHS + 관리 유틸리티)
# 2: Standalone HTTP Server
INSTALL_TYPE=Standalone HTTP Server

# 보안 업데이트 수신 거부 (폐쇄망 환경)
DECLINE_SECURITY_UPDATES=true

# My Oracle Support 정보 (빈칸으로 설정 시 보안 업데이트 건너뜀)
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
MOS_USERNAME=
MOS_PASSWORD=

[SYSTEM]

[APPLICATIONS]

[RELATIONSHIPS]
EOF
```

### 6.2. Silent 설치 실행

```bash
# oracle 계정으로 Silent 설치 실행
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
export ORACLE_HOME=/u01/app/oracle/product/fmw12214/ohs

# JAR 파일 방식 설치 (fmw_12.2.1.4.0_ohs.jar 기준)
java -jar /tmp/fmw_12.2.1.4.0_ohs.jar \
  -silent \
  -responseFile /home/oracle/ohs_install.rsp \
  -invPtrLoc /home/oracle/oraInst.loc \
  ORACLE_HOME=${ORACLE_HOME} \
  -jreLoc ${JAVA_HOME} \
  2>&1 | tee /home/oracle/ohs_install.log

# .bin 파일 방식 설치 시 (fmw_12.2.1.4.0_ohs_linux64.bin)
/tmp/fmw_12.2.1.4.0_ohs_linux64.bin \
  -silent \
  -responseFile /home/oracle/ohs_install.rsp \
  -invPtrLoc /home/oracle/oraInst.loc \
  2>&1 | tee /home/oracle/ohs_install.log

# 설치 중 진행 상황 확인
tail -f /home/oracle/ohs_install.log

# 설치 완료 확인
# 마지막 라인에 "Successfully Setup Software" 메시지 확인
grep -E "Successfully|ERROR|WARNING" /home/oracle/ohs_install.log
```

### 6.3. root 스크립트 실행 (설치 완료 후 필수)

```bash
# root 권한으로 실행
sudo /u01/app/oraInventory/orainstRoot.sh
# 또는 설치 로그에서 안내되는 스크립트 경로 확인 후 실행
```

## 7. OPatch 최신 버전 적용

보안 패치 및 버그 수정을 위해 OPatch를 최신 버전으로 업데이트합니다.

```bash
# 현재 OPatch 버전 확인
$ORACLE_HOME/OPatch/opatch version
# OPatch Version: 13.9.x.x.x

# My Oracle Support에서 최신 OPatch(p6880880) 다운로드 후 교체
cd /tmp
unzip p6880880_122140_Linux-x86-64.zip -d /tmp/opatch_new/

# 기존 OPatch 백업
mv $ORACLE_HOME/OPatch $ORACLE_HOME/OPatch.$(date +%Y%m%d)

# 새 OPatch 교체
cp -r /tmp/opatch_new/OPatch $ORACLE_HOME/

# 업데이트된 버전 확인
$ORACLE_HOME/OPatch/opatch version

# 설치된 패치 목록 확인
$ORACLE_HOME/OPatch/opatch lspatches
```

## 8. OHS 컴포넌트 생성 (component.rsp)

OHS는 설치 후 별도의 컴포넌트(인스턴스)를 생성해야 합니다.

```bash
# OHS 환경 변수 설정
cat >> /home/oracle/.bash_profile << 'EOF'

# ==== Oracle HTTP Server Environment ====
export ORACLE_HOME=/u01/app/oracle/product/fmw12214/ohs
export ORACLE_INSTANCE=/u01/app/oracle/config/ohs_inst1
export PATH=$ORACLE_HOME/bin:$ORACLE_HOME/ohs/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
EOF

source /home/oracle/.bash_profile

# OHS 컴포넌트 응답 파일 생성
cat > /home/oracle/ohs_component.rsp << 'EOF'
[GENERAL]
componentType=OHS
componentName=ohs1
EOF

# OHS 컴포넌트 생성 (Standalone 모드)
$ORACLE_HOME/bin/ohs createComponent \
  -oracleHome $ORACLE_HOME \
  -oracleInstance $ORACLE_INSTANCE \
  -componentType OHS \
  -componentName ohs1 \
  -adminPort 7779 \
  -listenPort 7777 \
  -sslPort 4443 \
  2>&1 | tee /home/oracle/ohs_component.log

# 생성 확인
ls -la $ORACLE_INSTANCE/config/OHS/ohs1/
```

## 9. OHS 기동 및 정지

```bash
# OHS 인스턴스 기동
$ORACLE_HOME/bin/ohs start \
  -oracleInstance $ORACLE_INSTANCE \
  -componentName ohs1

# 기동 상태 확인
$ORACLE_HOME/bin/ohs status \
  -oracleInstance $ORACLE_INSTANCE \
  -componentName ohs1
# 출력: ohs1 is running (PID: XXXX)

# 포트 확인 (7777: HTTP, 4443: HTTPS)
ss -tlnp | grep -E '7777|4443|7779'

# OHS 정지
$ORACLE_HOME/bin/ohs stop \
  -oracleInstance $ORACLE_INSTANCE \
  -componentName ohs1

# 재기동
$ORACLE_HOME/bin/ohs restart \
  -oracleInstance $ORACLE_INSTANCE \
  -componentName ohs1
```

### 9.1. 기동 스크립트 생성

```bash
# 기동/정지 관리 스크립트 작성
cat > /home/oracle/ohs-ctl.sh << 'SCRIPT'
#!/bin/bash
# OHS 12c 기동/정지 제어 스크립트

ORACLE_HOME=/u01/app/oracle/product/fmw12214/ohs
ORACLE_INSTANCE=/u01/app/oracle/config/ohs_inst1
COMP_NAME=ohs1
OHS_CMD=$ORACLE_HOME/bin/ohs

case "$1" in
  start)
    echo "Starting Oracle HTTP Server..."
    $OHS_CMD start -oracleInstance $ORACLE_INSTANCE -componentName $COMP_NAME
    ;;
  stop)
    echo "Stopping Oracle HTTP Server..."
    $OHS_CMD stop -oracleInstance $ORACLE_INSTANCE -componentName $COMP_NAME
    ;;
  restart)
    echo "Restarting Oracle HTTP Server..."
    $OHS_CMD restart -oracleInstance $ORACLE_INSTANCE -componentName $COMP_NAME
    ;;
  status)
    $OHS_CMD status -oracleInstance $ORACLE_INSTANCE -componentName $COMP_NAME
    ;;
  *)
    echo "Usage: $0 {start|stop|restart|status}"
    exit 1
    ;;
esac
SCRIPT

chmod 755 /home/oracle/ohs-ctl.sh

# root 계정으로 systemd 서비스 등록
sudo tee /etc/systemd/system/ohs.service << 'EOF'
[Unit]
Description=Oracle HTTP Server 12c
After=network.target

[Service]
Type=forking
User=oracle
Group=oinstall
ExecStart=/home/oracle/ohs-ctl.sh start
ExecStop=/home/oracle/ohs-ctl.sh stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ohs.service
```

## 10. mod_wl_ohs WebLogic 프록시 설정

OHS와 WebLogic Server를 연동하여 HTTP 요청을 WebLogic으로 전달합니다.

### 10.1. mod_wl_ohs.conf 설정

```bash
# mod_wl_ohs 설정 파일 생성
cat > $ORACLE_INSTANCE/config/OHS/ohs1/moduleconf/mod_wl_ohs.conf << 'EOF'
# ── mod_wl_ohs 로드 ──────────────────────────────
LoadModule weblogic_module ${ORACLE_HOME}/ohs/modules/mod_wl_ohs.so

# ── WebLogic 클러스터 연결 설정 ─────────────────
<IfModule weblogic_module>
    # WebLogic 관리 서버 정보
    WebLogicHost  wls-admin.example.com
    WebLogicPort  7001

    # 연결 타임아웃 설정
    ConnectTimeoutSecs  10
    ConnectRetrySecs    2

    # 클러스터 설정 (복수의 Managed Server)
    # WebLogicCluster ms01.example.com:8001,ms02.example.com:8001
</IfModule>

# ── URL 패턴 기반 프록시 라우팅 ──────────────────
# /app 경로는 WebLogic으로 프록시
<Location /myapp>
    SetHandler weblogic-handler
    WebLogicHost ms01.example.com
    WebLogicPort 8001
    # 클러스터 라운드로빈 로드밸런싱
    # WebLogicCluster ms01.example.com:8001,ms02.example.com:8001
</Location>

# ── REST API 프록시 ──────────────────────────────
<Location /api>
    SetHandler weblogic-handler
    WebLogicHost ms01.example.com
    WebLogicPort 8001
    PathTrim /api
</Location>

# ── 정적 리소스는 OHS가 직접 제공 ───────────────
<Location /static>
    # mod_wl_ohs 제외 (Apache가 직접 처리)
</Location>
EOF

# 설정 파일 검증
$ORACLE_HOME/bin/ohs status -oracleInstance $ORACLE_INSTANCE -componentName ohs1

# 설정 적용을 위해 재기동
/home/oracle/ohs-ctl.sh restart
```

### 10.2. 세션 고착성(Session Stickiness) 설정

WebLogic 클러스터 환경에서 세션 일관성을 보장합니다.

```bash
# mod_wl_ohs 세션 고착성 설정 추가
cat >> $ORACLE_INSTANCE/config/OHS/ohs1/moduleconf/mod_wl_ohs.conf << 'EOF'

# ── 세션 고착성 (로드밸런서 세션 유지) ─────────
<IfModule weblogic_module>
    # JSESSIONID 쿠키 기반 세션 고착성
    CookieName        JSESSIONID
    CookiePath        /
    WLCookieName      SERVERID
    KeepAliveEnabled  ON
    KeepAliveSecs     20
</IfModule>
EOF
```

## 11. 트러블슈팅

### 11.1. 설치 실패: JVM 초기화 오류

```bash
# 증상: "Could not create the Java virtual machine"
# 원인 1: JDK 버전 불일치 (OHS 12c는 JDK 8 권장)
java -version
# 원인 2: libnsl 라이브러리 누락
rpm -qa | grep libnsl
sudo dnf install -y libnsl libnsl2

# 원인 3: /tmp 공간 부족
df -h /tmp
# 부족하면 TEMP 환경변수 변경
export TEMP=/home/oracle/tmp
mkdir -p $TEMP
```

### 11.2. OHS 기동 실패: Address already in use

```bash
# 포트 충돌 확인
sudo ss -tlnp | grep -E '7777|4443|7779'
sudo lsof -i :7777

# ohs1 컴포넌트 설정에서 포트 변경
vi $ORACLE_INSTANCE/config/OHS/ohs1/httpd.conf
# Listen 7777 → Listen 8777

# 또는 오래된 PID 파일 제거 후 재기동
rm -f $ORACLE_INSTANCE/diagnostics/logs/OHS/ohs1/*.pid
/home/oracle/ohs-ctl.sh start
```

### 11.3. mod_wl_ohs 연결 오류

```bash
# 증상: 502 Bad Gateway 또는 "WebLogic Bridge Message '10002'"
# 원인 확인: OHS 에러 로그
tail -100 $ORACLE_INSTANCE/diagnostics/logs/OHS/ohs1/error_log

# 1. WebLogic 서버 기동 확인
curl -s http://ms01.example.com:8001/  # WebLogic Managed Server 직접 접근 테스트

# 2. 방화벽 확인 (OHS → WebLogic 통신)
telnet ms01.example.com 8001

# 3. WebLogic Cluster 주소/포트 오타 확인
grep WebLogicCluster $ORACLE_INSTANCE/config/OHS/ohs1/moduleconf/mod_wl_ohs.conf

# 4. 연결 타임아웃 증가
# ConnectTimeoutSecs 10 → 30
```

### 11.4. OPatch 적용 실패

```bash
# OPatch 적용 전 확인사항
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph /tmp/patch_xxxxx
# 충돌 없으면 계속 진행

# 패치 적용
cd /tmp/patch_xxxxx
$ORACLE_HOME/OPatch/opatch apply

# 롤백 필요 시
$ORACLE_HOME/OPatch/opatch rollback -id XXXXXXXX
```

## 12. 마무리

Oracle HTTP Server 12c의 Silent 설치와 WebLogic 프록시 연동까지 완료되었습니다. 운영 환경 투입 전 다음 항목을 추가로 검토하세요.

- **SSL 인증서**: 자체 서명 인증서 대신 공인 CA 인증서 또는 Oracle Wallet 기반 SSL 구성
- **로그 로테이션**: `$ORACLE_INSTANCE/diagnostics/logs/OHS/ohs1/` 하위 로그 파일의 logrotate 설정
- **보안 패치**: My Oracle Support에서 정기적으로 CPU(Critical Patch Update) 패치 적용
- **WebLogic Plugin 설정**: `WLProxySSL`, `WLProxySSLPassThrough` 설정으로 SSL 종단 처리 구성
- **모니터링**: OHS 액세스 로그를 ELK Stack 또는 Fluentd로 수집하여 트래픽 분석 환경 구축
