---
layout: single
author_profile: true
title: "Apache Tomcat 10 설치 및 systemd 서비스 등록 가이드 (Oracle Linux 8)"
date: 2026-06-11 11:00:00 +0900
categories:
  - Middleware
tags:
  - Tomcat
  - Oracle Linux
  - WAS
  - Middleware
toc: true
toc_sticky: true
header:
  overlay_image: https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80
  overlay_filter: 0.5
  show_overlay_excerpt: false
excerpt: "Oracle Linux 8에서 Apache Tomcat 10을 OpenJDK 설치부터 CATALINA_HOME 구성, server.xml 기본 튜닝, systemd 서비스 등록, 테스트 WAR 배포, 로그 로테이션까지 실무 가이드로 완전히 안내합니다."
---

## 1. 개요

Apache Tomcat 10은 Jakarta EE 9.1 스펙을 기반으로 하며, 특히 Jakarta Servlet 5.0, Jakarta Server Pages 3.0을 지원합니다. Tomcat 9와의 가장 큰 차이점은 패키지 네임스페이스가 `javax.*`에서 `jakarta.*`로 전환된 것입니다. 신규 프로젝트에서는 Tomcat 10을 선택하되, 기존 Tomcat 9용 애플리케이션은 마이그레이션 도구(`jakartaee-migration-tool`)를 활용해야 합니다.

본 가이드는 Oracle Linux 8에서 Tomcat 10을 바이너리 배포판으로 설치하고 실운영 환경에 맞는 구성을 갖추는 전 과정을 다룹니다.

## 2. 사전 준비: OpenJDK 설치

Tomcat 10은 Java 11 이상을 요구합니다. Oracle Linux 8의 기본 리포지토리에서 OpenJDK 11 또는 17을 설치할 수 있습니다.

```bash
# 사용 가능한 JDK 버전 목록 확인
sudo dnf list available java-*-openjdk-devel

# OpenJDK 11 설치 (LTS, 안정성 우선 시)
sudo dnf install -y java-11-openjdk-devel

# 또는 OpenJDK 17 설치 (최신 LTS, 성능 향상)
sudo dnf install -y java-17-openjdk-devel

# 여러 JDK 설치 시 기본값 변경
sudo alternatives --config java
sudo alternatives --config javac

# Java 버전 확인
java -version
javac -version
```

## 3. Tomcat 전용 계정 및 디렉터리 구성

```bash
# Tomcat 전용 그룹 및 계정 생성
sudo groupadd -g 1200 tomcat
sudo useradd -u 1200 -g tomcat -m -d /home/tomcat -s /bin/bash tomcat

# 설치 디렉터리 생성
sudo mkdir -p /opt/tomcat
sudo mkdir -p /opt/tomcat/logs
sudo mkdir -p /opt/tomcat/webapps

# 소유권 설정
sudo chown -R tomcat:tomcat /opt/tomcat
```

## 4. Tomcat 10 다운로드 및 설치

### 4.1. 공식 바이너리 다운로드

Apache Tomcat 공식 사이트(https://tomcat.apache.org/)에서 최신 Tomcat 10 릴리즈를 다운로드합니다.

```bash
# Tomcat 10 최신 버전 다운로드 (버전 번호는 최신 버전으로 변경)
TOMCAT_VERSION="10.1.24"
TOMCAT_URL="https://dlcdn.apache.org/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz"

cd /tmp
wget -q "$TOMCAT_URL" -O "apache-tomcat-${TOMCAT_VERSION}.tar.gz"

# 체크섬 검증 (보안상 필수)
wget -q "${TOMCAT_URL}.sha512" -O "apache-tomcat-${TOMCAT_VERSION}.tar.gz.sha512"
sha512sum -c "apache-tomcat-${TOMCAT_VERSION}.tar.gz.sha512"
# 출력: apache-tomcat-10.1.24.tar.gz: OK

# 압축 해제 및 배치
sudo tar xzf "apache-tomcat-${TOMCAT_VERSION}.tar.gz" -C /opt/tomcat
sudo mv "/opt/tomcat/apache-tomcat-${TOMCAT_VERSION}" /opt/tomcat/tomcat10

# 심볼릭 링크 생성 (버전 업그레이드 편의성)
sudo ln -sfn /opt/tomcat/tomcat10 /opt/tomcat/current
sudo chown -R tomcat:tomcat /opt/tomcat/tomcat10 /opt/tomcat/current
```

### 4.2. 디렉터리 구조 확인

```bash
ls -la /opt/tomcat/current/
# drwxr-xr-x  bin/        # startup.sh, catalina.sh 등 실행 스크립트
# drwx------  conf/       # server.xml, web.xml, context.xml 등 설정 파일
# drwxr-xr-x  lib/        # 공통 라이브러리 (JAR)
# drwxrwxr-x  logs/       # 로그 파일 (catalina.out 등)
# drwxrwxr-x  temp/       # 임시 파일
# drwxr-xr-x  webapps/    # 배포 애플리케이션 디렉터리
# drwxrwxr-x  work/       # JSP 컴파일 캐시
```

## 5. 환경 변수 설정

```bash
# setenv.sh 파일 생성 (Tomcat 시작 시 자동 로드)
sudo -u tomcat tee /opt/tomcat/current/bin/setenv.sh << 'EOF'
#!/bin/bash
# ==== Tomcat 10 Environment Configuration ====

# JAVA_HOME 설정
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))

# JVM 메모리 설정 (운영 환경에 맞게 조정)
CATALINA_OPTS="-Xms512m -Xmx2g"
CATALINA_OPTS="$CATALINA_OPTS -XX:MetaspaceSize=128m"
CATALINA_OPTS="$CATALINA_OPTS -XX:MaxMetaspaceSize=512m"

# G1 GC 사용 (Java 11+ 권장)
CATALINA_OPTS="$CATALINA_OPTS -XX:+UseG1GC"
CATALINA_OPTS="$CATALINA_OPTS -XX:MaxGCPauseMillis=200"

# GC 로그 설정
CATALINA_OPTS="$CATALINA_OPTS -Xlog:gc*:file=${CATALINA_HOME}/logs/gc.log:time,uptime:filecount=5,filesize=50m"

# 서버 식별자 (클러스터 환경에서 구분용)
CATALINA_OPTS="$CATALINA_OPTS -Djvm.route=$(hostname -s)"

# IPv4 강제 사용
CATALINA_OPTS="$CATALINA_OPTS -Djava.net.preferIPv4Stack=true"

# 보안 난수 생성기 (기동 속도 향상)
CATALINA_OPTS="$CATALINA_OPTS -Djava.security.egd=file:/dev/./urandom"

export CATALINA_OPTS
EOF

sudo chmod 755 /opt/tomcat/current/bin/setenv.sh
```

## 6. server.xml 기본 튜닝

### 6.1. server.xml 주요 설정 항목

```bash
# 원본 백업 후 편집
sudo cp /opt/tomcat/current/conf/server.xml /opt/tomcat/current/conf/server.xml.orig
sudo -u tomcat vi /opt/tomcat/current/conf/server.xml
```

주요 튜닝 포인트:

```xml
<!-- 서버 포트 변경 (기본 8080) -->
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"
           maxThreads="200"
           minSpareThreads="10"
           maxConnections="8192"
           acceptCount="100"
           compression="on"
           compressionMinSize="2048"
           compressableMimeType="text/html,text/xml,text/plain,text/css,text/javascript,application/javascript"
           URIEncoding="UTF-8"
           server="Apache" />

<!-- AJP 커넥터 (Apache HTTPD mod_jk 연동 시 사용, 불필요 시 주석 처리) -->
<!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443"
             address="0.0.0.0" secret="AJP_SECRET_KEY" /> -->
```

### 6.2. 서버 정보 은닉 (보안)

```bash
# Tomcat 버전 정보 노출 방지
sudo -u tomcat vi /opt/tomcat/current/conf/server.xml
# Server 엘리먼트의 version 속성 제거 또는 가짜 값 설정:
# <Server port="8005" shutdown="SHUTDOWN">
# → <Server port="-1" shutdown="DISABLED">  (shutdown 포트 비활성화 권장)
```

## 7. systemd 서비스 등록

```bash
# systemd 서비스 파일 생성
sudo tee /etc/systemd/system/tomcat10.service << 'EOF'
[Unit]
Description=Apache Tomcat 10 Web Application Server
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

# JAVA_HOME 및 CATALINA 경로 설정
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk"
Environment="CATALINA_HOME=/opt/tomcat/current"
Environment="CATALINA_BASE=/opt/tomcat/current"
Environment="CATALINA_PID=/opt/tomcat/current/temp/tomcat.pid"

# 시작/정지 스크립트
ExecStart=/opt/tomcat/current/bin/startup.sh
ExecStop=/opt/tomcat/current/bin/shutdown.sh

# PID 파일 경로
PIDFile=/opt/tomcat/current/temp/tomcat.pid

# 비정상 종료 시 재시작
Restart=on-failure
RestartSec=30

# 열린 파일 제한 수 증가
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 서비스 등록 및 시작
sudo systemctl daemon-reload
sudo systemctl enable tomcat10.service
sudo systemctl start tomcat10.service

# 상태 확인
sudo systemctl status tomcat10.service

# 기동 로그 실시간 확인
tail -f /opt/tomcat/current/logs/catalina.out
```

## 8. 방화벽 설정

```bash
# Tomcat HTTP 포트 개방
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=8443/tcp  # HTTPS 사용 시
sudo firewall-cmd --reload

# 확인
sudo firewall-cmd --list-ports
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/
# 출력: 200 (Tomcat 기본 페이지 응답)
```

## 9. 테스트 WAR 배포

### 9.1. 샘플 애플리케이션 배포

```bash
# Tomcat 기본 예제 애플리케이션 확인
ls /opt/tomcat/current/webapps/
# ROOT/     # 기본 홈
# docs/     # Tomcat 문서
# examples/ # 서블릿/JSP 예제
# host-manager/
# manager/

# 간단한 테스트용 WAR 파일 생성
mkdir -p /tmp/testapp/WEB-INF
cat > /tmp/testapp/index.jsp << 'JSPEOF'
<%@ page contentType="text/html; charset=UTF-8" %>
<html>
<body>
<h1>Tomcat 10 Test</h1>
<p>Server: <%= application.getServerInfo() %></p>
<p>Java Version: <%= System.getProperty("java.version") %></p>
<p>Time: <%= new java.util.Date() %></p>
</body>
</html>
JSPEOF

cat > /tmp/testapp/WEB-INF/web.xml << 'XMLEOF'
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_5_0.xsd"
         version="5.0">
  <display-name>Test Application</display-name>
</web-app>
XMLEOF

# JAR(WAR) 패키징
cd /tmp
jar cvf testapp.war -C testapp .

# webapps 디렉터리에 배포 (자동 배포)
sudo cp /tmp/testapp.war /opt/tomcat/current/webapps/
sudo chown tomcat:tomcat /opt/tomcat/current/webapps/testapp.war

# 배포 확인 (압축 해제 완료 시 폴더 생성)
ls -la /opt/tomcat/current/webapps/testapp/

# 브라우저 또는 curl로 접근 확인
curl http://localhost:8080/testapp/
```

## 10. 로그 로테이션 설정

Tomcat의 `catalina.out`은 기본적으로 무한정 증가하므로 logrotate 설정이 필수입니다.

```bash
# logrotate 설정 파일 생성
sudo tee /etc/logrotate.d/tomcat10 << 'EOF'
/opt/tomcat/current/logs/catalina.out {
    # 매일 로테이션
    daily
    # 30일치 보관
    rotate 30
    # 빈 파일도 로테이션
    missingok
    # 비어있는 파일 로테이션 건너뜀
    notifempty
    # 로테이션 후 압축
    compress
    # 지연 압축 (이전 로그 파일 압축)
    delaycompress
    # 파일 복사 후 원본 비우기 (Tomcat이 파일 핸들 유지하므로 필수)
    copytruncate
    # 새 파일 권한
    create 0644 tomcat tomcat
}

/opt/tomcat/current/logs/*.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
    create 0644 tomcat tomcat
}
EOF

# logrotate 설정 테스트 (dry-run)
sudo logrotate -d /etc/logrotate.d/tomcat10

# 즉시 로테이션 실행
sudo logrotate -f /etc/logrotate.d/tomcat10
```

## 11. 트러블슈팅

### 11.1. 기동 실패: Address already in use

```bash
# 포트 사용 확인
sudo ss -tlnp | grep -E '8080|8005|8443'

# 충돌 프로세스 확인 및 종료
sudo lsof -i :8080
sudo kill -9 <PID>

# Tomcat 셧다운 포트(8005) 충돌 시 server.xml에서 포트 변경
# <Server port="8005" ...> → <Server port="9005" ...>
```

### 11.2. JSP 컴파일 오류 (ClassNotFoundException)

```bash
# Tomcat 10에서 javax.* → jakarta.* 패키지 변경 확인
# 오류 예시: java.lang.ClassNotFoundException: javax.servlet.http.HttpServlet
# 해결: WAR에서 jakarta.* 패키지 사용 또는 마이그레이션 도구 활용

# 마이그레이션 도구 사용 예시
java -jar jakartaee-migration-1.0.7-shaded.jar myapp-tomcat9.war myapp-tomcat10.war
```

### 11.3. 느린 기동 시간 (SecureRandom 이슈)

```bash
# 증상: "Creation of SecureRandom instance for session ID generation using [SHA1PRNG] took [xxxxx] milliseconds"
# 해결: setenv.sh에 아래 옵션 추가 (이미 위 설정에 포함)
# -Djava.security.egd=file:/dev/./urandom

# 또는 JVM 보안 설정 파일 직접 수정
sudo sed -i 's/securerandom.source=file:\/dev\/random/securerandom.source=file:\/dev\/urandom/g' \
  $JAVA_HOME/conf/security/java.security
```

### 11.4. 로그 레벨 조정 (디버그 모드)

```bash
# catalina.out 대신 별도 로그 파일로 분리하여 문제 추적
# conf/logging.properties 수정
sudo -u tomcat vi /opt/tomcat/current/conf/logging.properties

# 특정 패키지 로그 레벨 DEBUG로 변경
# com.example.myapp.handlers = java.util.logging.ConsoleHandler
# com.example.myapp.level = FINE
```

## 12. 정리 및 다음 단계

Apache Tomcat 10 설치 및 기본 구성이 완료되었습니다. 운영 투입 전 다음 작업을 추가로 검토하세요.

- **Manager 앱 접근 제한**: `conf/tomcat-users.xml` 및 `webapps/manager/META-INF/context.xml`에서 접근 IP 제한
- **AJP 커넥터 비활성화**: Apache HTTPD 연동이 없다면 AJP(8009) 포트를 비활성화하여 보안 강화
- **HTTPS 설정**: Let's Encrypt 또는 자체 서명 인증서로 SSL/TLS 구성
- **데이터소스 구성**: JNDI DataSource를 context.xml에 설정하여 커넥션 풀 관리
- **모니터링**: JMX Remote 활성화 후 VisualVM, Prometheus 등으로 JVM 메트릭 수집
