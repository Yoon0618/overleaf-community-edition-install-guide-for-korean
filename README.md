# Ubuntu ARM64 환경에서 Overleaf 서버 완벽 구축 가이드 (수정본)

이 문서는 ARM64 아키텍처를 사용하는 Ubuntu 서버(예: AWS Graviton, Raspberry Pi, Apple Silicon VM 등)에서 Overleaf Community Edition을 소스 코드부터 직접 빌드하고, 외부 접속 및 추가 패키지 설치까지 완료하는 전체 과정을 안내합니다.

## 목차
1.  **사전 준비**: Docker, Git 등 필수 도구 설치
2.  **ARM64 이미지 빌드**: Overleaf 소스 코드를 ARM64 환경에 맞게 컴파일
3.  **Toolkit 연동 및 설정**: 빌드한 이미지를 `overleaf-toolkit`에서 사용하도록 설정
4.  **서버 실행 및 외부 접속**: 방화벽을 설정하고 서버를 실행하여 외부 접속 확인
5.  **(선택) 추가 패키지 설치 및 영구 저장**: TeX Live 전체 패키지를 설치하고 이미지에 변경사항 저장
6.  **(보너스) 불필요한 Docker 이미지 정리**

---

## 1. 사전 준비

서버에 접속하여 Docker, Docker Compose, Git을 설치합니다.

```bash
# 시스템 패키지 목록 업데이트 및 필수 패키지 설치
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg git

# Docker 공식 GPG 키 추가
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker 저장소 설정
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker Engine 및 Compose 플러그인 설치
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# (중요) sudo 없이 Docker 사용 설정
sudo usermod -aG docker $USER

# ==> 중요: 그룹 설정을 적용하려면 터미널을 종료했다가 다시 접속하세요!
```

---

## 2. ARM64 이미지 빌드

ARM64 환경에서는 공식 Docker 이미지를 사용할 수 없으므로, 소스 코드에서 직접 빌드해야 합니다.

```bash
# 1. 홈 디렉토리로 이동하여 Overleaf 소스 코드 복제
cd ~
git clone [https://github.com/overleaf/overleaf.git](https://github.com/overleaf/overleaf.git)

# 2. server-ce 디렉토리로 이동
cd ~/overleaf/server-ce

# 3. Base 이미지 빌드 (TeX Live 포함, 시간이 오래 걸릴 수 있음)
make build-base

# 4. Community 이미지 빌드 (최종 Overleaf 이미지 생성)
make build-community
```
이 과정이 성공적으로 끝나면, 로컬 Docker 환경에 `sharelatex/sharelatex:main` 이라는 ARM64용 이미지가 생성됩니다.

---

## 3. Toolkit 연동 및 설정

이제 빌드한 이미지를 `overleaf-toolkit`이 사용하도록 설정합니다.

```bash
# 1. 홈 디렉토리로 이동하여 overleaf-toolkit 복제 및 초기화
cd ~
git clone [https://github.com/overleaf/toolkit.git](https://github.com/overleaf/toolkit.git) ./overleaf-toolkit
cd ./overleaf-toolkit
bin/init

# 2. Toolkit이 요구하는 버전 확인 (예: 5.5.4)
cat config/version
# 출력된 버전(예: 5.5.4)을 기억합니다.

# 3. 빌드한 이미지에 버전 태그 지정 (예시: 5.5.4)
# '5.5.4' 부분은 위에서 확인한 실제 버전으로 바꿔주세요.
docker tag sharelatex/sharelatex:main sharelatex/sharelatex:5.5.4

# 4. 외부 접속을 위한 설정 파일 수정
nano config/overleaf.rc
```

`overleaf.rc` 파일에서 아래 부분을 수정하고 저장(`Ctrl+X`, `Y`, `Enter`)합니다.

```bash
# 외부 접속을 허용하도록 IP를 0.0.0.0으로 변경합니다.
OVERLEAF_LISTEN_IP=0.0.0.0

# Community Edition에서는 지원되지 않는 기능이므로 false로 변경하여 경고를 제거합니다.
SIBLING_CONTAINERS_ENABLED=false
```

---

## 4. 서버 실행 및 외부 접속

```bash
# 1. 방화벽에서 80번(HTTP), 22번(SSH) 포트 허용
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 22 -j ACCEPT

# 2. Overleaf 서버를 백그라운드에서 시작
bin/up -d

# 3. 공인 IP 주소 확인
curl ifconfig.me
```

이제 **다른 기기(PC, 스마트폰 등)**의 웹 브라우저에서 `http://<확인된_공인_IP>` 주소로 접속하여 `/launchpad` 페이지에서 관리자 계정을 생성합니다.

---

## 5. (선택) 추가 패키지 설치 및 영구 저장

Overleaf Community Edition은 필수 패키지만 포함하고 있으므로, 모든 패키지를 설치해 패키지 누락으로 인한 문제를 방지합니다.

```bash
# 1. 실행 중인 Overleaf 컨테이너의 셸에 접속
bin/shell

# --- 아래부터는 컨테이너 내부에서 실행 ---

# 2. TeX Live 패키지 매니저 업데이트
tlmgr update --self

# 3. 모든 패키지 설치 (시간이 매우 오래 걸릴 수 있습니다)
tlmgr install scheme-full

# 4. (중요) 설치된 바이너리 경로 업데이트
tlmgr path add

# 5. 컨테이너에서 빠져나오기
exit

# --- 다시 서버 터미널로 돌아옴 ---

# 6. 현재 버전 확인 (예: 5.5.4)
cat config/version

# 7. 변경사항을 새로운 태그와 함께 이미지로 저장(commit)
# "5.5.4" 부분은 실제 버전에 맞게, 접미사는 반드시 "-with-texlive-full"로 지정
docker commit sharelatex sharelatex/sharelatex:5.5.4-with-texlive-full

# 8. Toolkit이 새로운 이미지를 사용하도록 버전 파일 업데이트
echo 5.5.4-with-texlive-full > config/version

# 9. 변경된 설정으로 서버 재시작
bin/up -d
```
이제 서버가 재시작되어도 모든 TeX Live 패키지가 포함된 상태로 유지됩니다.

---

## 6. (보너스) 불필요한 Docker 이미지 정리

`docker commit` 이나 빌드 과정에서 이름이 없는(`<none>`) 중간 이미지들이 생길 수 있습니다. 이런 이미지들은 공간만 차지하므로 아래 명령어로 깔끔하게 정리할 수 있습니다.

```bash
docker image prune
```
`y`를 입력하여 삭제를 진행하면 됩니다.
