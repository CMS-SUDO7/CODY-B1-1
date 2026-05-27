# 요구사항 수행 내역서 (서버 관제 및 로깅 자동화 미션)

본 문서는 리눅스 서버 환경에서 다중 사용자 권한 관리, 네트워크 보안 설정, 애플리케이션 배포 환경 구축 및 시스템 상태 관제 자동화 스크립트(`monitor.sh`)를 구현한 전체 수행 내역을 기록한 문서입니다.

---

## 📋 목차
1. [설정 및 명령어 기록](#1-설정-및-명령어-기록)
2. [필수 증거 자료 체크리스트](#2-필수-증거-자료-체크리스트)
3. [자동화 스크립트 소스코드](#3-자동화-스크립트-소스코드-monitorsh)

---

## 1. 설정 및 명령어 기록

### 1.1 기본 보안 및 네트워크 설정 (SSH & 방화벽)
* **SSH 포트 변경(20022) 및 Root 로그인 차단**
  ```bash
  # 0. 기초 업데이트
  sudo apt update
  sudo apt install -y openssh-server
  sudo apt install -y ufw
  
  # 1. sshd_config 파일 내 포트 및 Root 로그인 차단 설정 수정
  sudo sed -i -E 's/^#?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
  sudo sed -i -E 's/^#?Port 22/Port 20022/' /etc/ssh/sshd_config
  
  # 2. systemd SSH 소켓 서비스 포트 변경 (최신 우분투 대응)
  sudo sed -i 's/ListenStream=22/ListenStream=20022/' /lib/systemd/system/ssh.socket
  
  # 3. 설정 리로드 및 서비스 전체 재시작
  sudo systemctl daemon-reload
  sudo systemctl restart ssh.socket
  sudo systemctl restart sshd

  # 4.포트체크
  sudo ss -tulnp | grep sshd
  ```

* **방화벽(UFW) 설치 및 인바운드 정책 구성**
  ```bash

  # 1. 필수 서비스 포트만 명시적 허용 (화이트리스트 방식)
  sudo ufw allow 20022/tcp
  sudo ufw allow 15034/tcp

  # 2. 방화벽 활성화
  sudo ufw enable

  # 3. 방화벽 체크
  sudo ufw status
  ```

### 1.2 계정/그룹 관리 및 디렉토리 권한 설정 (ACL)
* **역할 기반 사용자 계정 및 그룹 생성**
  ```bash
  # 1. 협업 그룹 생성
  sudo groupadd agent-common
  sudo groupadd agent-core

  # 2. 엔지니어별 계정 생성
  sudo useradd -m -s /bin/bash agent-admin
  sudo useradd -m -s /bin/bash agent-dev
  sudo useradd -m -s /bin/bash agent-test

  # 3. 그룹별 멤버 할당
  sudo usermod -aG agent-common,agent-core agent-admin
  sudo usermod -aG agent-common,agent-core agent-dev
  sudo usermod -aG agent-common agent-test

  # 4.그룹 체크
  id agent-admin 
  id agent-dev 
  id agent-test
  ```

* **공유/보안 디렉토리 구조 설계 및 권한 분리**
  ```bash
  # 1. 기준 디렉토리 변수 설정
  export AGENT_HOME=/home/agent-admin/agent-app

  # 2. 폴더 구조 생성
  sudo -u agent-admin mkdir -p $AGENT_HOME/upload_files
  sudo -u agent-admin mkdir -p $AGENT_HOME/api_keys
  sudo -u agent-admin mkdir -p $AGENT_HOME/bin
  sudo mkdir -p /var/log/agent-app

  # 3. 권한 정책 적용 (최소 권한의 원칙)
  # upload_files: 공유 디렉토리 (agent-common 그룹 전체 R/W 가능)
  sudo chgrp agent-common $AGENT_HOME/upload_files
  sudo chmod 770 $AGENT_HOME/upload_files

  # api_keys 및 로그: 보안 디렉토리 (agent-core 핵심 그룹 ONLY R/W 가능)
  sudo chgrp agent-core $AGENT_HOME/api_keys
  sudo chmod 770 $AGENT_HOME/api_keys
  sudo chgrp agent-core /var/log/agent-app
  sudo chmod 770 /var/log/agent-app

  # 5. 권한체크
  앱 폴더 내부 확인 
  sudo ls -l /home/agent-admin/agent-app/ 
  로그 폴더 확인 
  sudo ls -ld /var/log/agent-app
  ```

### 1.3 애플리케이션 실행 환경 구축
* **환경 변수 영구 등록 및 키 파일 생성**
  ```bash
  # 1. 관리자 계정으로 전환
  sudo su - agent-admin
  
  # 2. 프로필 파일(~/.bashrc) 내 실행 환경 변수 영구 등록
  cat << 'EOF' >> ~/.bashrc
  export AGENT_HOME=/home/agent-admin/agent-app
  export AGENT_PORT=15034
  export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
  export AGENT_KEY_PATH=$AGENT_HOME/api_keys
  export AGENT_LOG_DIR=/var/log/agent-app
  EOF

  echo 'export AGENT_HOME=/home/agent-admin/agent-app' >> ~/.bashrc 
  echo 'export AGENT_PORT=15034' >> ~/.bashrc 
  echo 'export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files' >> ~/.bashrc 
  echo 'export AGENT_KEY_PATH=$AGENT_HOME/api_keys' >> ~/.bashrc 
  echo 'export AGENT_LOG_DIR=/var/log/agent-app' >> ~/.bashrc
  
  환경변수 체크 
  env | grep AGENT
  환경변수 즉시 적용
  source ~/.bashrc

  # 3. 애플리케이션 검증용 비밀 키 생성 및 소유권 잠금
  echo "agent_api_key_test" > $AGENT_KEY_PATH/secret.key
  chmod 640 $AGENT_KEY_PATH/secret.key
  ```

* **애플리케이션 구동 (백그라운드 실행 프로세스)**
  ```bash
  # 앱 디렉토리로 이동 후 백그라운드 모드로 실행 (터미널 종료 후에도 유지)
  cd $AGENT_HOME
  ./agent-app-linux-x86
  nohup ./agent-app-linux-x86 > /dev/null 2>&1 &
  ```

### 1.4 자동화 스크립트 설정 (cron & logrotate)
* **로그 무한 증식 방지 정책 구현 (logrotate)**
  ```bash
  # root 권한으로 logrotate 정책 정의 파일 생성 (최대 10MB 크기, 10개 파일 보존)
  sudo bash -c 'cat << EOF > /etc/logrotate.d/agent-app
  /var/log/agent-app/monitor.log {
      su agent-admin agent-core
      size 10M
      rotate 10
      missingok
      notifempty
      create 0660 agent-admin agent-core
  }
  EOF'
  ```

* **크론탭(crontab)을 이용한 무인 모니터링 자동화**
  ```bash
  # 1. cron 시스템 서비스 활성화 및 시작
  sudo systemctl enable cron
  sudo service cron start

  # 2. agent-admin 계정의 크론탭 스케줄러에 스크립트 상주 등록 (매분 실행)
  (crontab -l 2>/dev/null; echo "* * * * * /home/agent-admin/agent-app/bin/monitor.sh") | crontab -
  
* **복사한 파일 옮기기**
  sudo cp /Users/herebattle6145/Downloads/agent-app/agent-app-linux-x86 /home/agent-admin/agent-app/
  ```

---

## 2. 필수 증거 자료 체크리스트

| 점검 항목 | 결과 | 검증 내용 및 확보 데이터 요약 |
| :--- | :---: | :--- |
| **SSH 포트(20022) 및 Root 접속 차단** | `[PASS]` | `sudo ss -tulnp \| grep sshd` 조회 시 `0.0.0.0:20022` 인바운드 대기 상태 확인 완료. |
| **방화벽(UFW) 포트 허용 정책** | `[PASS]` | `sudo ufw status` 조회 시 `Status: active` 및 지정된 두 포트(20022/15034) ALLOW 적용 확인 완료. |
| **계정 및 권한 그룹 편성** | `[PASS]` | `id agent-admin` 명령어 결과 내역에 `agent-common`, `agent-core` 그룹 매핑 데이터 일치 확인. |
| **디렉토리 구조 및 권한 분리** | `[PASS]` | `ls -ld` 확인 시 권한 트리(770)가 구축되었으며, 미인가자의 접근이 거부(`Permission denied`)됨을 교차 검증. |
| **Boot Sequence 및 Agent READY** | `[PASS]` | 5단계 체크(계정, 환경변수, 키 파일, 포트, 권한) `[OK]` 패스 및 15034 포트 Listen 바인딩 성공 내역 확인. |
| **monitor.sh 실행 및 모니터링 로직** | `[PASS]` | 백그라운드 프로세스 감지, UFW 활성 점검, 서버 자원(CPU/MEM/DISK) 메트릭 추출 등 스크립트 정상 동작 확인. |
| **monitor.log 누적 데이터 정합성** | `[PASS]` | `/var/log/agent-app/monitor.log` 열람 시 규정된 포맷(`[시간] PID CPU MEM DISK 경고`)으로 안전하게 인입됨 확인. |
| **crontab 자동 실행(1분 간격 누적)** | `[PASS]` | `15:38:44`, `15:39:01`, `15:40:01` 등 시스템 스케줄러를 통해 1분 간격으로 시계열 로그가 정상 누적됨을 확인. |

---

## 3. 자동화 스크립트 소스코드 (`monitor.sh`)

* **파일 시스템 배치 경로:** `/home/agent-admin/agent-app/bin/monitor.sh`
* **소유 구조 및 권한:** 소유자 `agent-dev` / 소유그룹 `agent-core` / 권한 `750` (`rwxr-x---`)

```bash
sudo bash -c 'cat << '\''EOF'\'' > /home/agent-admin/agent-app/bin/monitor.sh
#!/bin/bash
APP_NAME="agent-app-linux-x86"
APP_PORT=15034
LOG_FILE="/var/log/agent-app/monitor.log"
DATE_STR=$(date +"%Y-%m-%d %H:%M:%S")

echo "====== SYSTEM MONITOR RESULT ======"
echo ""
echo "[HEALTH CHECK]"
APP_PID=$(pgrep -f "$APP_NAME" | head -n 1)
if [ -n "$APP_PID" ]; then
    echo "Checking process '\$APP_NAME'... [OK] (PID: $APP_PID)"
else
    echo "Checking process '\$APP_NAME'... [FAIL]"
    exit 1
fi

if ss -tulnp 2>/dev/null | grep -q ":$APP_PORT "; then
    echo "Checking port $APP_PORT... [OK]"
else
    echo "Checking port $APP_PORT... [FAIL]"
    exit 1
fi
echo ""

# 자원 수집
CPU_IDLE=$(vmstat 1 2 | tail -1 | awk "{print \$15}")
CPU_USAGE=$((100 - CPU_IDLE))
MEM_TOTAL=$(free -m | awk "/Mem:/ {print \$2}")
MEM_USED=$(free -m | awk "/Mem:/ {print \$3}")
MEM_USAGE=$(( MEM_USED * 100 / MEM_TOTAL ))
DISK_USAGE=$(df -h / | awk "NR==2 {print \$5}" | sed "s/%//")

echo "[RESOURCE MONITORING]"
echo "CPU Usage : ${CPU_USAGE}%"
echo "MEM Usage : ${MEM_USAGE}%"
echo "DISK Used  : ${DISK_USAGE}%"
echo ""

# 경고 및 로그 메세지 생성
WARNINGS=""
if [ "$(systemctl is-active ufw)" != "active" ]; then 
    echo "[WARNING] UFW is inactive"
    WARNINGS="$WARNINGS [WARNING] UFW is inactive"
fi
if [ "$CPU_USAGE" -gt 20 ]; then 
    echo "[WARNING] CPU threshold exceeded (${CPU_USAGE}% > 20%)"
    WARNINGS="$WARNINGS [WARNING] CPU > 20%(${CPU_USAGE}%)"
fi
if [ "$MEM_USAGE" -gt 10 ]; then 
    echo "[WARNING] MEM threshold exceeded (${MEM_USAGE}% > 10%)"
    WARNINGS="$WARNINGS [WARNING] MEM > 10%(${MEM_USAGE}%)"
fi
if [ "$DISK_USAGE" -gt 80 ]; then 
    echo "[WARNING] DISK threshold exceeded (${DISK_USAGE}% > 80%)"
    WARNINGS="$WARNINGS [WARNING] DISK_USED > 80%(${DISK_USAGE}%)"
fi

# 파일에 1줄 로그 기록
LOG_MESSAGE="[$DATE_STR] PID:$APP_PID CPU:${CPU_USAGE}% MEM:${MEM_USAGE}% DISK_USED:${DISK_USAGE}% $WARNINGS"
echo "$LOG_MESSAGE" | sed "s/  */ /g" >> "$LOG_FILE"
echo ""

# 통계 리포트 (기존 로그 파일 분석)
echo "====== STATISTICS REPORT ======"
if [ -f "$LOG_FILE" ]; then
    awk '\''
    BEGIN { c_sum=0; m_sum=0; cnt=0; c_max=0; m_max=0; c_min=1000; m_min=1000; }
    {
        date_time = $1" "$2;
        if(match($4, /[0-9]+/)) { cpu = substr($4, RSTART, RLENGTH) + 0; }
        if(match($5, /[0-9]+/)) { mem = substr($5, RSTART, RLENGTH) + 0; }
        
        cnt++; c_sum += cpu; m_sum += mem;
        if(cpu >= c_max) { c_max = cpu; c_max_dt = date_time; }
        if(cpu <= c_min) { c_min = cpu; c_min_dt = date_time; }
        if(mem >= m_max) { m_max = mem; m_max_dt = date_time; }
        if(mem <= m_min) { m_min = mem; m_min_dt = date_time; }
    }
    END {
        if(cnt > 0) {
            printf "[CPU]\n"
            printf "Average : %.1f%%\n", c_sum/cnt
            printf "Maximum : %d%% at %s\n", c_max, substr(c_max_dt, 2, length(c_max_dt)-2)
            printf "Minimum : %d%% at %s\n", c_min, substr(c_min_dt, 2, length(c_min_dt)-2)
            printf "[Memory]\n"
            printf "Average : %.1f%%\n", m_sum/cnt
            printf "Maximum : %d%% at %s\n", m_max, substr(m_max_dt, 2, length(m_max_dt)-2)
            printf "Minimum : %d%% at %s\n", m_min, substr(m_min_dt, 2, length(m_min_dt)-2)
            printf "[Samples]\n"
            printf "Data Points: %d samples\n", cnt
        }
    }'\'' "$LOG_FILE"
fi

echo ""
echo "[INFO] Log appended: $LOG_FILE"
exit 0
EOF'

# 2.권한 변경
sudo chown agent-dev:agent-core /home/agent-admin/agent-app/bin/monitor.sh
sudo chmod 750 /home/agent-admin/agent-app/bin/monitor.sh

# 3. 출력
sudo -u agent-admin /home/agent-admin/agent-app/bin/monitor.sh
```
