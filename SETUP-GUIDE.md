# OTA-CE + Aktualizr 실전 설정 가이드

Mac(Apple Silicon)에서 ota-ce 서버를 실행하고, Jetson Orin Nano에 aktualizr를 설치해 OTA 통신을 연결하는 전체 과정입니다.

---

## 환경

| 항목 | 내용 |
|------|------|
| 서버 | MacBook Air (Apple Silicon, M 시리즈) |
| 클라이언트 | NVIDIA Jetson Orin Nano |
| OS (Jetson) | Ubuntu 22.04 (JetPack 기반) |
| Docker | Docker Desktop v28+ |

---

## 전체 흐름

```
[Mac - ota-ce 서버]                    [Jetson Orin Nano - aktualizr]
┌──────────────────────┐               ┌──────────────────────────┐
│  Docker (ota-ce)     │◄──mTLS 30443──►  aktualizr              │
│  - director          │               │  - config.toml           │
│  - reposerver        │               │  - 인증서 (ca/client/key) │
│  - treehub           │               └──────────────────────────┘
│  - deviceregistry    │
└──────────────────────┘
```

---

## 1단계: Mac에서 ota-ce 서버 설정

### 1-1. 저장소 클론

```bash
git clone https://github.com/SOR-OTA-Team/sor_ota_ce.git
cd sor_ota_ce
```

### 1-2. `/etc/hosts` 설정

> `nano`가 Ghostty 터미널에서 오류를 낼 수 있으므로 `tee` 방식을 사용합니다.

```bash
sudo tee -a /etc/hosts << 'EOF'
127.0.0.1    reposerver.ota.ce
127.0.0.1    keyserver.ota.ce
127.0.0.1    director.ota.ce
127.0.0.1    treehub.ota.ce
127.0.0.1    deviceregistry.ota.ce
127.0.0.1    campaigner.ota.ce
127.0.0.1    app.ota.ce
127.0.0.1    ota.ce
EOF
```

> **주의:** 명령어를 한 번에 붙여넣지 말고 `EOF`까지 한 블록씩 실행하세요.

### 1-3. 서버 인증서 생성

```bash
bash scripts/gen-server-certs.sh
```

### 1-4. 서버 시작

Apple Silicon Mac은 `linux/arm64` 이미지를 지원하지 않는 경우가 있으므로 `amd64`로 강제 실행합니다.

```bash
# DB 먼저 시작 (20초 대기)
docker compose -f ota-ce.yaml up db -d
sleep 20

# 전체 서비스 시작
DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose -f ota-ce.yaml up -d
```

### 1-5. 동작 확인

```bash
curl director.ota.ce/health/version
# {"name":"director-v2", ...} 가 출력되면 정상
```

---

## 2단계: TUF 저장소 초기화 및 디바이스 인증서 생성

### 2-1. TUF 저장소 초기화

디바이스 등록 전 반드시 실행해야 합니다. 이 단계를 빠뜨리면 aktualizr가 `403 Forbidden: No repo for namespace` 오류를 냅니다.

```bash
bash scripts/get-credentials.sh
```

### 2-2. `gnu-sed` 설치 (macOS 필수)

macOS 기본 `sed`는 `-z` 옵션을 지원하지 않아 `config.toml` 생성에 실패합니다.

```bash
brew install gnu-sed
export PATH="/opt/homebrew/opt/gnu-sed/libexec/gnubin:$PATH"
```

### 2-3. 디바이스 인증서 생성

```bash
bash scripts/gen-device.sh
```

생성 결과 (`ota-ce-gen/devices/<uuid>/`):
- `config.toml` — aktualizr 설정 파일
- `client.pem` — 클라이언트 인증서
- `ca.pem` — 서버 CA 인증서
- `pkey.pem` — 개인 키
- `gateway.url` — 게이트웨이 URL

---

## 3단계: Jetson 헤드리스 접속 (모니터 없이)

모니터 없이 Mac에서 Jetson에 접속하는 방법입니다.

### 방법 1: USB 시리얼 (초기 부팅 확인용)

```bash
ls /dev/tty.usbmodem*
screen /dev/tty.usbmodem* 115200
```

### 방법 2: SSH (권장)

```bash
ssh <username>@<jetson-ip>
```

Jetson IP 확인: Jetson에서 `ip addr show` 또는 라우터 관리 페이지에서 확인

---

## 4단계: Jetson에 aktualizr 설치

### 4-1. 의존성 설치

```bash
sudo apt update && sudo apt upgrade -y

# build-essential 오류 시
sudo apt --fix-broken install
sudo apt install -y build-essential

# 나머지 의존성
sudo apt install -y \
  cmake git \
  libcurl4-openssl-dev libssl-dev \
  libboost-all-dev libarchive-dev \
  libsodium-dev libsqlite3-dev
```

> `libsqlite3-dev` 누락 시 cmake 빌드 실패합니다.

### 4-2. sor_aktualizr 클론 및 빌드

```bash
git clone https://github.com/SOR-OTA-Team/sor_aktualizr.git
cd sor_aktualizr

# git 태그 추가 (버전 인식용)
git tag v1.0.0

mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

> 빌드 시간: Jetson 기준 5~15분

### 4-3. 라이브러리 경로 등록

```bash
sudo ldconfig
```

### 4-4. 설치 확인

```bash
aktualizr --version
```

---

## 5단계: 인증서 전송 및 네트워크 설정

### 5-1. Jetson에 디렉토리 생성

Jetson에서:
```bash
mkdir -p ~/ota-config
```

### 5-2. Mac에서 인증서 전송

Mac에서 (`<uuid>`는 실제 생성된 UUID로 변경):
```bash
scp -r ota-ce-gen/devices/<uuid>/ <username>@<jetson-ip>:~/ota-config/
```

### 5-3. Jetson `/etc/hosts` 설정

Jetson에서 (`<MAC_IP>`는 Mac의 실제 IP):
```bash
echo "<MAC_IP>    ota.ce director.ota.ce reposerver.ota.ce treehub.ota.ce deviceregistry.ota.ce campaigner.ota.ce" | sudo tee -a /etc/hosts
```

Mac IP 확인 (Mac에서):
```bash
ipconfig getifaddr en0
```

---

## 6단계: aktualizr 실행

> **중요:** 반드시 config 파일이 있는 디렉토리에서 실행해야 합니다. 상대 경로로 인증서를 참조하기 때문입니다.

```bash
cd ~/ota-config/<uuid>/
sudo aktualizr -c config.toml
```

### 정상 동작 출력 예시

```
Aktualizr version v1.0.0 starting
...
All ECUs are already registered with the server.
... provisioned OK
PUT /director/manifest → 200 OK
No new updates found in Uptane metadata.
```

---

## 트러블슈팅

### `no matching manifest for linux/arm64/v8`

Apple Silicon Mac에서 발생. 해결:
```bash
DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose -f ota-ce.yaml up -d
```

### `sed: illegal option -- z`

macOS 기본 sed 문제. 해결:
```bash
brew install gnu-sed
export PATH="/opt/homebrew/opt/gnu-sed/libexec/gnubin:$PATH"
```

### `config.toml` 에서 server가 빈 값

config 디렉토리 밖에서 실행했을 때 발생. 반드시 config가 있는 폴더에서 실행:
```bash
cd ~/ota-config/<uuid>/
sudo aktualizr -c config.toml
```

### `403 Forbidden: No repo for namespace`

TUF 저장소 미초기화. 해결:
```bash
# Mac에서
bash scripts/get-credentials.sh
```

### `libaktualizr.so: cannot open shared object file`

빌드 후 라이브러리 경로 미등록. 해결:
```bash
sudo ldconfig
```

### `Could not find sqlite3` (cmake 오류)

```bash
sudo apt install -y libsqlite3-dev
```

### `could not get current version from git`

git 태그 없음. 해결:
```bash
cd ~/sor_aktualizr
git fetch --tags
# 태그가 없으면
git tag v1.0.0
cd build && cmake ..
```

### SSH `No route to host`

SSH 서비스 확인:
```bash
# Jetson에서
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Docker daemon 응답 없음

```bash
killall Docker
open /Applications/Docker.app
# 메뉴바 아이콘 안정화 대기 후
docker info
```

---

## 참고

- [SOR OTA CE Repository](https://github.com/SOR-OTA-Team/sor_ota_ce)
- [SOR Aktualizr Repository](https://github.com/SOR-OTA-Team/sor_aktualizr)
- [Uptane Standard](https://uptane.github.io/)
