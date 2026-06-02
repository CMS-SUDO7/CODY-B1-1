# 요구사항 수행 내역서 (서버 관제 및 로깅 자동화 미션)

본 문서는 리눅스 서버 환경에서 다중 사용자 권한 관리, 네트워크 보안 설정, 애플리케이션 배포 환경 구축 및 시스템 상태 관제 자동화 스크립트(`monitor.sh`)를 구현한 전체 수행 내역을 기록한 문서입니다.


---

## 📋 목차
1. [설정 및 명령어 기록](#1-설정-및-명령어-기록)
2. [자동화 스크립트 소스코드](#2-자동화-스크립트-소스코드-monitorsh)

---

## 1. 설정 및 명령어 기록

### 1.0 가상환경 셋티 
```bash
orb create ubuntu:22.04 MD
```
### 1.1 기본 보안 및 네트워크 설정 (SSH & 방화벽)
* **SSH 포트 변경(20022) 및 Root 로그인 차단**
  ```bash
  # 0. 기초 업데이트
  sudo apt update
  sudo apt install -y openssh-server
  sudo apt install -y ufw
  sudo apt install acl -y
  
  # 1. sshd_config 파일 내 포트 및 Root 로그인 차단 설정 수정
  sudo sed -i -E 's/^#?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
  sudo sed -i -E 's/^#?Port 22/Port 20022/' /etc/ssh/sshd_config
  
  # 2. systemd SSH 소켓 서비스 포트 변경 (최신 우분투 대응)
  sudo sed -i 's/ListenStream=22/ListenStream=20022/' /lib/systemd/system/ssh.socket
  
  # 3. 설정 리로드 및 서비스 전체 재시작
  sudo systemctl daemon-reload
  sudo systemctl restart ssh.socket
  sudo systemctl restart ssh

  # 4.포트체크 // 루트 로그인 차단 확인
  sudo ss -tulnp | grep ssh
  sudo sshd -T | grep permitrootlogin
  ```
<img width="1157" height="105" alt="Screenshot 2026-05-28 at 7 03 33 PM" src="https://github.com/user-attachments/assets/78240321-5b73-48f9-8e64-a04598319430" />

<img width="840" height="47" alt="Screenshot 2026-05-28 at 7 03 58 PM" src="https://github.com/user-attachments/assets/c89b16f6-e4c5-4038-bba8-fea366e421dd" />


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

<img width="738" height="205" alt="Screenshot 2026-05-28 at 7 05 19 PM" src="https://github.com/user-attachments/assets/23fc7535-40d6-4be7-8add-0fd1d7799f6f" />


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
  getent group agent-core
  getent group agent-common
  
  id agent-admin 
  id agent-dev 
  id agent-test
  ```

<img width="836" height="83" alt="Screenshot 2026-05-28 at 7 09 19 PM" src="https://github.com/user-attachments/assets/bab1418b-580d-4f7b-8cac-1c9eebb688d7" />


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
  # agent-dev가 들어올수 있도록 권한 수정
  sudo setfacl -m u:agent-dev:rx /home/agent-admin /home/agent-admin/agent-app
  
  # 5. 권한체크
  앱 폴더 내부 확인 
  sudo ls -l /home/agent-admin/agent-app/ 
  로그 폴더 확인 
  sudo ls -ld /var/log/agent-app
  ```
  
* **다운받은 파일 복사하기**
  ```bash
  sudo cp /Users/herebattle6145/Downloads/agent-app/agent-app-linux-x86 /home/agent-admin/agent-app/
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

  환경변수 즉시 적용
  source ~/.bashrc
  
  환경변수 체크 
  env | grep AGENT


  # 3. 애플리케이션 검증용 비밀 키 생성 및 소유권 잠금
  echo "agent_api_key_test" > /home/agent-admin/agent-app/api_keys/secret.key
  chgrp agent-core /home/agent-admin/agent-app/api_keys/secret.key
  chmod 660 /home/agent-admin/agent-app/api_keys/secret.key
  ```

* **애플리케이션 구동 (백그라운드 실행 프로세스)**
  ```bash
  # 앱 디렉토리로 이동 후 앱 실행
  cd $AGENT_HOME
  ./agent-app-linux-x86
  ```

<img width="1169" height="350" alt="Screenshot 2026-05-28 at 7 41 05 PM" src="https://github.com/user-attachments/assets/c710da80-417f-4580-9a64-913b56411374" />




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

  1단계: 파일이 잘 만들어졌는지 눈으로 확인하기
  cat /etc/logrotate.d/agent-app

  2단계: 가짜로 실행해 보기 (Dry-run / 가장 중요 ⭐️)
  sudo logrotate -d /etc/logrotate.d/agent-app

  3단계: 당장 강제로 실행해 보기 (Force)
  sudo logrotate -f /etc/logrotate.d/agent-app
  ```

* **크론탭(crontab)을 이용한 무인 모니터링 자동화**
  ```bash
  # 1. cron 시스템 서비스 활성화 및 시작
  sudo systemctl enable cron
  sudo service cron start

  # 2. agent-admin 계정의 크론탭 스케줄러에 스크립트 상주 등록 (매분 실행)
  (crontab -l 2>/dev/null; echo "* * * * * /home/agent-admin/agent-app/bin/monitor.sh") | crontab -

  # 3.실행중인 크론탭 목록
  crontab -l
  ```


---

## 2. 자동화 스크립트 소스코드 (`monitor.sh`)

* **파일 시스템 배치 경로:** `/home/agent-admin/agent-app/bin/monitor.sh`
* **소유 구조 및 권한:** 소유자 `agent-dev` / 소유그룹 `agent-core` / 권한 `750` (`rwxr-x---`)

```bash
sudo bash -c 'cat << '\''EOF'\'' > /home/agent-admin/agent-app/bin/monitor.sh
#!/bin/bash

# 실행 중인 애플리케이션 자동 감지
if pgrep -f "agent-leak-app-x86" > /dev/null; then
    APP_NAME="agent-leak-app-x86"
else
    APP_NAME="agent-app-linux-x86"
fi

APP_PORT=15034

LOG_FILE="/var/log/agent-app/monitor.log"
DATE_STR=$(date +"%Y-%m-%d %H:%M:%S")

echo "====== SYSTEM MONITOR RESULT ======"
echo ""
echo "[HEALTH CHECK]"

APP_PID=$(pgrep -f "$APP_NAME" | grep -v "monitor.sh" | head -n 1)

if [ -n "$APP_PID" ]; then
    echo "Checking process '\''$APP_NAME'\''... [OK] (PID: $APP_PID)"
else
    echo "Checking process '\''$APP_NAME'\''... [FAIL]"
    exit 1
fi

if ss -tulnp 2>/dev/null | grep -q ":$APP_PORT "; then
    echo "Checking port $APP_PORT... [OK]"
else
    echo "Checking port $APP_PORT... [FAIL]"
    exit 1
fi

echo ""
# [수정] 특정 프로세스의 자원 수집 (CPU는 %, 메모리는 RSS(물리메모리, KB를 MB로 변환))
PROC_CPU=$(ps -p $APP_PID -o %cpu= | awk '{print $1}')
PROC_MEM_KB=$(ps -p $APP_PID -o rss= | awk '{print $1}')
PROC_MEM_MB=$(( PROC_MEM_KB / 1024 ))

DISK_USAGE=$(df -h / | awk "NR==2 {print \$5}" | sed "s/%//")

echo "[RESOURCE MONITORING]"
echo "Process CPU Usage : ${PROC_CPU}%"
echo "Process MEM Usage : ${PROC_MEM_MB} MB"
echo "DISK Used         : ${DISK_USAGE}%"
echo ""

# 경고 및 로그 메세지 생성
WARNINGS=""
# CPU 스파이크 감지 기준점 (예: 80% 이상)
if awk "BEGIN {exit !($PROC_CPU > 80.0)}"; then 
    echo "[WARNING] CPU threshold exceeded (${PROC_CPU}% > 80%)"
    WARNINGS="$WARNINGS [WARNING] CPU > 80%(${PROC_CPU}%)"
fi

# 메모리 사용량 경고 (단위: MB)
if [ "$PROC_MEM_MB" -gt 300 ]; then 
    echo "[WARNING] MEM threshold exceeded (${PROC_MEM_MB}MB > 300MB)"
    WARNINGS="$WARNINGS [WARNING] MEM > 300MB(${PROC_MEM_MB}MB)"
fi

# 로그 기록 (awk가 잘 파싱하도록 형태 유지 및 실수(float) 파싱 지원)
LOG_MESSAGE="[$DATE_STR] PID:$APP_PID CPU:${PROC_CPU}% MEM:${PROC_MEM_MB}MB DISK_USED:${DISK_USAGE}% $WARNINGS"
echo "$LOG_MESSAGE" | sed "s/  */ /g" >> "$LOG_FILE"
echo ""

# 통계 리포트 
echo "====== STATISTICS REPORT ======"
if [ -f "$LOG_FILE" ]; then
    awk '\''
    BEGIN { c_sum=0; m_sum=0; cnt=0; c_max=0; m_max=0; c_min=1000; m_min=999999; }
    {
        date_time = $1" "$2;
        # [수정] 소수점(float)도 파싱되도록 정규식 [0-9.] 사용
        if(match($4, /[0-9.]+/)) { cpu = substr($4, RSTART, RLENGTH) + 0; }
        if(match($5, /[0-9.]+/)) { mem = substr($5, RSTART, RLENGTH) + 0; }
        
        cnt++; c_sum += cpu; m_sum += mem;
        if(cpu >= c_max) { c_max = cpu; c_max_dt = date_time; }
        if(cpu <= c_min) { c_min = cpu; c_min_dt = date_time; }
        if(mem >= m_max) { m_max = mem; m_max_dt = date_time; }
        if(mem <= m_min) { m_min = mem; m_min_dt = date_time; }
    }
    END {
        if(cnt > 0) {
            printf "[Process CPU]\n"
            printf "Average : %.1f%%\n", c_sum/cnt
            printf "Maximum : %.1f%% at %s\n", c_max, substr(c_max_dt, 2, length(c_max_dt)-2)
            printf "Minimum : %.1f%% at %s\n", c_min, substr(c_min_dt, 2, length(c_min_dt)-2)
            printf "[Process Memory (Physical)]\n"
            printf "Average : %.1f MB\n", m_sum/cnt
            printf "Maximum : %.1f MB at %s\n", m_max, substr(m_max_dt, 2, length(m_max_dt)-2)
            printf "Minimum : %.1f MB at %s\n", m_min, substr(m_min_dt, 2, length(m_min_dt)-2)
            printf "[Samples]\n"
            printf "Data Points: %d samples\n", cnt
        }
    }'\'' "$LOG_FILE"
fi
echo ""
echo "[INFO] Log appended: $LOG_FILE"
exit 0
EOF'
```

```bash
# 2-2.권한 변경
sudo chown agent-dev:agent-core /home/agent-admin/agent-app/bin/monitor.sh
sudo chmod 750 /home/agent-admin/agent-app/bin/monitor.sh
sudo systemctl restart ufw

# 2-3. 출력
sudo -u agent-admin /home/agent-admin/agent-app/bin/monitor.sh

# 2-4. monitor.log 보기
sudo tail -f $AGENT_HOME/logs/monitor.log
```

 # 실습 해보기
 ```bash
 # 1. 관리자 계정으로 전환
 sudo su - agent-admin
 # 2. 디렉토리 이동
 cd $AGENT_HOME
 # 3. 앱을 백그라운드로 실행 
 nohup ./agent-app-linux-x86 > /dev/null 2>&1 &
 # 4. 앱 종료 명령어
 pkill -f agent-app-linux-x86
 # 5. 실행중인 앱 찾기
 ps -ef | grep agent-app-linux-x86
 sudo ss -tulnp | grep 15034
 ```
<img width="1154" height="140" alt="Screenshot 2026-05-28 at 8 03 46 PM" src="https://github.com/user-attachments/assets/fa36dd2e-0226-47e5-b4af-94a8a5687fcd" />

<img width="1154" height="152" alt="Screenshot 2026-05-28 at 8 20 18 PM" src="https://github.com/user-attachments/assets/cc4fa3dd-6c57-4ded-97df-684eca8319e9" />

<img width="1160" height="538" alt="Screenshot 2026-05-28 at 8 37 37 PM" src="https://github.com/user-attachments/assets/128d0b39-f9d8-44af-b6d6-0e0b27566aa0" />

<img width="1161" height="143" alt="Screenshot 2026-05-29 at 7 10 17 PM" src="https://github.com/user-attachments/assets/df6ac677-04b8-4794-b9d4-41ad9f082e83" />

