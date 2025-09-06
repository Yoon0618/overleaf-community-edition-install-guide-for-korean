# Docker를 이용한 Overleaf 개인 서버 구축 및 한글 패키지 설치 가이드

이 문서는 Docker를 사용하여 개인용 Overleaf Community Edition 서버를 구축하고, 한국어 LaTeX 문서 작성을 위해 필수적인 패키지를 설치하는 과정을 안내합니다.

## 왜 개인 서버를 구축하나요?
- **데이터 프라이버시**: 모든 프로젝트 파일이 개인 서버에 저장됩니다.
- **네트워크 독립성**: 인터넷 연결 없이도 문서 작성이 가능합니다.
- **패키지 자유도**: 필요한 LaTeX 패키지를 직접 설치하고 관리할 수 있습니다.
- **무료**: Community Edition은 무료로 제공됩니다.

---

## 1. 사전 준비물

- **Docker & Docker Compose**: [Docker 공식 홈페이지](https://www.docker.com/products/docker-desktop/)에서 자신의 OS에 맞는 버전을 설치합니다. Docker Desktop에는 Docker Compose가 포함되어 있습니다.
- **Git**: [Git 공식 홈페이지](https://git-scm.com/downloads)에서 설치합니다.

---

## 2. Overleaf Toolkit 설치

Overleaf Community Edition은 `overleaf/toolkit`이라는 공식 GitHub 저장소를 통해 배포됩니다.

```bash
# 원하는 위치에 overleaf-toolkit 저장소를 복제합니다.
git clone [https://github.com/overleaf/overleaf-toolkit.git](https://github.com/overleaf/overleaf-toolkit.git)

# 생성된 디렉토리로 이동합니다.
cd overleaf-toolkit
```

---

## 3. 서버 실행 및 관리

모든 서버 관리는 `overleaf-toolkit` 디렉토리 안에서 `docker-compose` 명령어로 이루어집니다.

### 서버 최초 실행
```bash
# Docker 이미지를 다운로드하고 설정에 맞게 컨테이너를 생성 및 실행합니다.
# -d 옵션은 백그라운드에서 실행하라는 의미입니다.
docker-compose up -d
```
최초 실행 시 필요한 이미지를 다운로드하므로 시간이 다소 걸릴 수 있습니다. 실행이 완료되면 웹 브라우저에서 `http://localhost`로 접속하여 Overleaf 화면을 확인할 수 있습니다.

### 주요 관리 명령어

| 목적 | 명령어 | 설명 |
| :--- | :--- | :--- |
| **서버 시작** | `docker-compose up -d` | 백그라운드에서 서버를 시작합니다. |
| **서버 종료** | `docker-compose down` | 서버를 안전하게 종료합니다. (데이터는 보존됨) |
| **상태 확인** | `docker-compose ps` | 실행 중인 컨테이너 상태를 확인합니다. |
| **로그 확인** | `docker-compose logs -f` | 실시간 서버 로그를 확인합니다. (중단: `Ctrl+C`) |


---

## 4. LaTeX 추가 패키지 설치 (한글 설정)

기본 설치된 Overleaf에는 한국어 조판에 필요한 `kotex` 패키지가 포함되어 있지 않습니다. 다음과 같이 직접 설치해야 합니다.

### 1) Overleaf 컨테이너 접속
먼저 실행 중인 `sharelatex` 컨테이너의 셸(shell)에 접속합니다.

```bash
# 'sharelatex'라는 이름의 컨테이너에서 bash 셸을 실행합니다.
docker-compose exec sharelatex /bin/bash
```

### 2) TeX Live 패키지 매니저(tlmgr) 업데이트
컨테이너에 접속한 후, 가장 먼저 `tlmgr` 자체와 패키지 목록을 최신 상태로 업데이트합니다.

```bash
# tlmgr 자체를 업데이트합니다.
tlmgr update --self

# tlmgr의 패키지 저장소 정보를 업데이트합니다.
tlmgr update --all 
# 위 명령은 시간이 오래 걸릴 수 있으므로, --self 업데이트 후 바로 필요한 패키지를 설치해도 무방합니다.
```

### 3) 한글 패키지 컬렉션 설치
`kotex`는 단일 패키지가 아닌, 한국어 관련 패키지 묶음인 `collection-langkorean`에 포함되어 있습니다.

```bash
# 한국어 패키지 컬렉션을 설치합니다.
tlmgr install collection-langkorean
```
설치가 완료되면 `exit` 명령어로 컨테이너 셸을 빠져나옵니다. 이제 Overleaf 프로젝트에서 `\usepackage{kotex}`를 사용하면 정상적으로 한글 문서가 컴파일됩니다.

> 💡 **팁**: 앞으로 다른 패키지가 없다는 오류가 발생하면, 위와 동일한 방법으로 `tlmgr install [패키지명]`을 실행하여 필요한 패키지를 추가할 수 있습니다.

---

## 부록 A: 기본 설치 패키지 정보

Overleaf Community Edition Docker 이미지는 효율성을 위해 TeX Live의 모든 패키지를 포함하지 않고, **최소한의 패키지 묶음(scheme-small)**을 기반으로 필수 패키지 몇 개만 추가한 상태입니다.

정확히 어떤 패키지가 설치되어 있는지 확인하려면, 위에서 설명한 방법으로 컨테이너 셸에 접속한 후 다음 명령어를 실행하세요.

```bash
# 현재 설치된 모든 패키지 목록을 출력합니다.
tlmgr list --only-installed
```

만약 필요한 패키지가 많아 매번 설치하기 번거롭다면, 아래 명령어로 전체 패키지(`scheme-full`)를 설치할 수도 있습니다. **단, 수 GB의 디스크 공간이 추가로 필요하며 설치에 매우 오랜 시간이 걸립니다.**

```bash
# (주의: 용량이 매우 큼) 전체 TeX Live 패키지를 설치합니다.
tlmgr install scheme-full
```

## 부록 B: 공식 문서

- **Overleaf Community Edition 설치 가이드 (영문)**
  - [https://github.com/overleaf/overleaf/wiki/Quick-Start-Guide](https://github.com/overleaf/overleaf/wiki/Quick-Start-Guide)
- **TeX Live 공식 문서 (tlmgr)**
  - [https://www.tug.org/texlive/tlmgr.html](https://www.tug.org/texlive/tlmgr.html)
- **CTAN (종합 TeX 아카이브 네트워크)**
  - LaTeX 패키지를 검색할 수 있는 곳: [https://www.ctan.org/](https://www.ctan.org/)
