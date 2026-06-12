---
layout: single
author_profile: true
title: "Apache HTTPD 2.4 설치 및 기본 설정 가이드 (Oracle Linux 8)"
date: 2026-06-15 10:00:00 +0900
categories:
  - Middleware
tags:
  - Apache
  - Oracle Linux
  - Web
  - Middleware
toc: true
toc_sticky: true
header:
  overlay_image: https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80
  overlay_filter: 0.5
  show_overlay_excerpt: false
excerpt: "Oracle Linux 8에서 Apache HTTPD 2.4를 dnf로 설치하고 디렉터리 구조, httpd.conf 핵심 설정, 가상 호스트, mod_ssl, 방화벽, systemd, 로그 로테이션까지 실무 운영 관점의 완전 가이드를 제공합니다."
---

## 1. 개요

Apache HTTP Server(HTTPD) 2.4는 세계에서 가장 널리 사용되는 웹 서버로, 높은 안정성, 풍부한 모듈 생태계, 유연한 설정 구조를 자랑합니다. Oracle Linux 8 / RHEL 8 환경에서는 AppStream 리포지토리를 통해 패키지 설치가 가능하며, 실무에서는 WAS(WebLogic, JBoss, Tomcat) 앞단의 리버스 프록시나 OHS 대체 Web Server로 자주 활용됩니다.

본 가이드는 `dnf` 패키지 설치 방식을 기준으로 httpd.conf 핵심 설정, 가상 호스트 구성, mod_ssl 설정, 방화벽, systemd 서비스 관리, 로그 로테이션까지 실무 운영 환경에 맞게 구성하는 방법을 다룹니다.

## 2. Apache HTTPD 설치

### 2.1. 패키지 설치

```bash
# 시스템 업데이트
sudo dnf update -y

# Apache HTTPD 2.4 설치
sudo dnf install -y httpd

# SSL 모듈 및 추가 유틸리티 설치
sudo dnf install -y mod_ssl openssl

# 리버스 프록시 사용 시 추가 모듈
sudo dnf install -y mod_proxy_html

# 설치 버전 확인
httpd -v
# Server version: Apache/2.4.xx (Oracle Linux Server)

# 로드된 모듈 확인
httpd -M 2>/dev/null | head -30
```

### 2.2. 설치 후 디렉터리 구조

```bash
# 주요 디렉터리 및 파일 위치 확인
ls -la /etc/httpd/
# ├── conf/           # 메인 설정 파일 (httpd.conf)
# ├── conf.d/         # 추가 설정 파일 디렉터리 (*.conf)
# ├── conf.modules.d/ # 모듈 로드 설정
# ├── logs -> /var/log/httpd  # 로그 심볼릭 링크
# ├── modules -> /usr/lib64/httpd/modules
# └── run -> /run/httpd

# 문서 루트 기본 경로
ls -la /var/www/html/

# 로그 파일 경로
ls -la /var/log/httpd/
# access_log  → 접근 로그
# error_log   → 에러 로그
```

## 3. httpd.conf 기본 설정

### 3.1. 핵심 설정 항목

```bash
# 원본 백업
sudo cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.orig

# 주요 설정 편집
sudo vi /etc/httpd/conf/httpd.conf
```

운영 환경 필수 확인 항목:

```apache
# 서버명 설정 (없으면 경고 발생)
ServerName www.example.com:80

# 문서 루트
DocumentRoot "/var/www/html"

# 서버 관리자 이메일
ServerAdmin webmaster@example.com

# 서버 버전 정보 은닉 (보안)
ServerTokens Prod
ServerSignature Off

# 에러 문서 커스터마이징
ErrorDocument 403 /errors/403.html
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html

# Keep-Alive 설정 (성능 최적화)
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5

# 로그 형식 설정 (combined 형식에 응답 시간 추가)
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %D" combined_time
CustomLog /var/log/httpd/access_log combined_time

# 에러 로그 레벨 (운영: warn, 디버그: debug)
LogLevel warn
ErrorLog /var/log/httpd/error_log

# 서버 서명 비활성화
TraceEnable Off

# 디렉터리 인덱싱 비활성화
<Directory "/var/www/html">
    Options -Indexes +FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

### 3.2. MPM(Multi-Processing Module) 선택

Oracle Linux 8에서 기본 MPM은 `event`입니다. 설정은 `/etc/httpd/conf.modules.d/00-mpm.conf`에서 변경합니다.

```bash
# 현재 MPM 확인
httpd -V | grep MPM
# Server MPM: event

# MPM별 특성:
# event  → 고성능, 비동기 처리 (권장, Keep-Alive에 효율적)
# worker → 멀티스레드 (중간 성능)
# prefork → 프로세스 기반 (mod_php 사용 시 필요, 메모리 소비 많음)

# event MPM 튜닝 (/etc/httpd/conf.d/mpm-tuning.conf)
sudo tee /etc/httpd/conf.d/mpm-tuning.conf << 'EOF'
<IfModule mpm_event_module>
    ServerLimit          16
    StartServers          2
    MaxRequestWorkers   200
    MinSpareThreads      25
    MaxSpareThreads      75
    ThreadsPerChild      25
    MaxConnectionsPerChild 1000
</IfModule>
EOF
```

## 4. 가상 호스트(Virtual Host) 구성

운영 환경에서는 하나의 서버에 여러 도메인/서비스를 호스팅하는 경우가 많습니다.

```bash
# 가상 호스트 설정 파일 생성 (/etc/httpd/conf.d/ 하위)
sudo tee /etc/httpd/conf.d/myapp.conf << 'EOF'
# ── HTTP → HTTPS 리다이렉트 ──────────────────────
<VirtualHost *:80>
    ServerName myapp.example.com
    ServerAlias www.myapp.example.com

    # HTTP 접근 시 HTTPS로 리다이렉트
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>

# ── HTTPS 가상 호스트 ────────────────────────────
<VirtualHost *:443>
    ServerName myapp.example.com
    ServerAlias www.myapp.example.com
    ServerAdmin ops@example.com

    DocumentRoot /var/www/myapp
    ErrorLog  /var/log/httpd/myapp_error.log
    CustomLog /var/log/httpd/myapp_access.log combined

    # SSL 인증서
    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/myapp.crt
    SSLCertificateKeyFile /etc/ssl/private/myapp.key
    # 중간 인증서가 있는 경우
    # SSLCertificateChainFile /etc/ssl/certs/chain.crt

    # WAS 리버스 프록시 (Tomcat/JBoss 연동 예시)
    ProxyPreserveHost On
    ProxyRequests Off
    ProxyPass /api http://127.0.0.1:8080/api
    ProxyPassReverse /api http://127.0.0.1:8080/api

    # 정적 파일은 Apache가 직접 제공
    Alias /static /var/www/myapp/static
    <Directory /var/www/myapp/static>
        Options -Indexes
        Require all granted
    </Directory>
</VirtualHost>
EOF

# 문서 루트 생성
sudo mkdir -p /var/www/myapp
echo "<h1>MyApp is running!</h1>" | sudo tee /var/www/myapp/index.html
sudo chown -R apache:apache /var/www/myapp
```

## 5. mod_ssl 설정 (HTTPS)

### 5.1. 자체 서명(Self-Signed) 인증서 생성 (테스트 환경)

```bash
# 디렉터리 생성
sudo mkdir -p /etc/ssl/private

# RSA 개인키 생성 (4096비트)
sudo openssl genrsa -out /etc/ssl/private/myapp.key 4096
sudo chmod 600 /etc/ssl/private/myapp.key

# 자체 서명 인증서 생성 (유효기간 365일)
sudo openssl req -new -x509 \
  -key /etc/ssl/private/myapp.key \
  -out /etc/ssl/certs/myapp.crt \
  -days 365 \
  -subj "/C=KR/ST=Seoul/L=Seoul/O=TechGHO/OU=SRE/CN=myapp.example.com"

# 권한 설정
sudo chmod 644 /etc/ssl/certs/myapp.crt
```

### 5.2. SSL 보안 강화 설정

```bash
sudo tee /etc/httpd/conf.d/ssl-hardening.conf << 'EOF'
# SSL 프로토콜 설정 (TLS 1.2 이상만 허용)
SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

# 강력한 암호화 스위트 사용
SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384

# 서버 암호화 스위트 우선 적용
SSLHonorCipherOrder on

# HSTS (HTTP Strict Transport Security) 헤더
<IfModule mod_headers.c>
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
</IfModule>

# OCSP Stapling (인증서 검증 성능 향상)
SSLUseStapling On
SSLStaplingResponderTimeout 5
SSLStaplingReturnResponderErrors off
SSLStaplingCache shmcb:/var/run/ocsp(128000)
EOF
```

## 6. systemd 서비스 관리

```bash
# HTTPD 서비스 시작 및 부팅 자동 시작 설정
sudo systemctl enable --now httpd

# 서비스 상태 확인
sudo systemctl status httpd

# 설정 구문 검증 (서비스 재시작 전 필수)
sudo apachectl configtest
# 출력: Syntax OK

# 설정 변경 후 무중단 리로드 (서비스 중단 없이 설정 반영)
sudo systemctl reload httpd

# 서비스 재시작 (모듈 변경 시 필요)
sudo systemctl restart httpd

# 로그 실시간 모니터링
sudo tail -f /var/log/httpd/error_log
sudo tail -f /var/log/httpd/access_log
```

## 7. 방화벽 설정

```bash
# HTTP/HTTPS 포트 개방
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# 개방된 서비스 확인
sudo firewall-cmd --list-services

# 특정 포트 직접 지정 방식
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload

# 연결 테스트
curl -I http://localhost/
curl -Ik https://localhost/  # 자체 서명 인증서는 -k 옵션
```

## 8. 로그 로테이션

Apache는 `logrotate`를 통해 로그 파일을 자동으로 관리합니다.

```bash
# 기본 logrotate 설정 확인
cat /etc/logrotate.d/httpd

# 운영 환경 맞춤 설정 (기본값 수정)
sudo tee /etc/logrotate.d/httpd << 'EOF'
/var/log/httpd/*log {
    # 매일 로테이션
    daily
    # 90일치 보관
    rotate 90
    # 압축 보관
    compress
    delaycompress
    # 빈 파일 로테이션 건너뜀
    notifempty
    # 로그 파일 없어도 오류 무시
    missingok
    # 새 로그 파일 생성 권한
    create 0640 root apache
    # 로테이션 후 HTTPD에 USR1 시그널 전송 (새 파일로 전환)
    sharedscripts
    postrotate
        /bin/systemctl reload httpd.service > /dev/null 2>/dev/null || true
    endscript
}
EOF

# logrotate 테스트 실행 (dry-run)
sudo logrotate -d /etc/logrotate.d/httpd
```

## 9. 트러블슈팅

### 9.1. Permission Denied (SELinux)

Oracle Linux 8에서는 SELinux가 기본적으로 활성화되어 있습니다.

```bash
# SELinux 상태 확인
getenforce
# Enforcing → SELinux 활성화 상태

# 증상: 특정 디렉터리에 접근 거부
# 확인: /var/log/audit/audit.log 에서 "denied" 키워드 검색
sudo grep denied /var/log/audit/audit.log | grep httpd | tail -20

# 임시 조치: SELinux 감사 로그 기반 정책 생성
sudo grep denied /var/log/audit/audit.log | audit2allow -M myhttpd
sudo semodule -i myhttpd.pp

# 웹 서비스용 컨텍스트 적용
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/myapp(/.*)?"
sudo restorecon -Rv /var/www/myapp

# httpd가 네트워크 연결 (프록시) 허용
sudo setsebool -P httpd_can_network_connect 1
```

### 9.2. 403 Forbidden

```bash
# 원인 1: DocumentRoot 권한 문제
ls -la /var/www/html/
sudo chmod 755 /var/www/html
sudo chown apache:apache /var/www/html

# 원인 2: .htaccess AllowOverride 설정 문제
# httpd.conf에서 AllowOverride None → AllowOverride All 변경

# 원인 3: SELinux 컨텍스트 문제
ls -Z /var/www/html/index.html
# httpd_sys_content_t 컨텍스트가 있어야 함
sudo restorecon -v /var/www/html/index.html
```

### 9.3. 설정 오류로 기동 실패

```bash
# 문법 오류 위치 확인
sudo apachectl configtest
# 출력 예: AH00526: Syntax error on line 45 of /etc/httpd/conf.d/myapp.conf:

# 에러 로그에서 상세 원인 확인
sudo journalctl -u httpd -n 50 --no-pager
sudo tail -50 /var/log/httpd/error_log
```

### 9.4. 성능 이슈: 응답 지연

```bash
# 접속자 수 및 프로세스 상태 확인
sudo apachectl status  # mod_status 활성화 시
# 또는
ps aux | grep httpd | wc -l

# 느린 요청 로그 분석 (응답시간 1초 이상)
awk '{if ($NF > 1000000) print}' /var/log/httpd/access_log | tail -20

# mod_status 활성화 (실시간 모니터링)
sudo tee /etc/httpd/conf.d/status.conf << 'CONF'
<Location /server-status>
    SetHandler server-status
    Require ip 127.0.0.1 10.0.0.0/8
</Location>
ExtendedStatus On
CONF
sudo systemctl reload httpd
curl http://localhost/server-status?auto
```

## 10. 마무리

Apache HTTPD 2.4의 설치 및 기본 운영 구성이 완료되었습니다. 엔터프라이즈 환경에서 WAS와 함께 사용하는 경우 다음 추가 구성을 권장합니다.

- **mod_jk / mod_proxy_ajp**: Tomcat, JBoss와 AJP 프로토콜로 연동하여 세션 고착성(Session Sticky) 구현
- **mod_cluster**: JBoss EAP의 클러스터 토폴로지를 자동으로 인식하는 동적 로드 밸런서
- **mod_security**: 웹 방화벽(WAF) 모듈로 OWASP 규칙 적용
- **Rate Limiting**: `mod_ratelimit`, `mod_evasive`로 DDoS 방어
- **모니터링**: Prometheus `apache_exporter`로 메트릭 수집 및 Grafana 대시보드 연동
