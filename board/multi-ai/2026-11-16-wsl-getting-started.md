---
title: "WSL에서 Hermes Agent 시작하기 — Docker 격리, Claude·GPT 연결, 첫 협업 시험"
---

* 목차
{:toc}


> 이 문서는 Windows 안의 리눅스(WSL Ubuntu)에 **Hermes Agent를 Docker 상자 안에서 안전하게** 설치하고, **Claude(총괄)와 GPT(검증관)를 구독 로그인으로 연결**해 첫 협업까지 해 본 2026년 10월 20~27일 기록을 초보자용으로 정리한 것입니다. 명령어는 모두 **실제로 입력되고 결과가 확인된 것만** 옮겼고, 확인하지 못한 부분은 만들어 쓰지 않았습니다. 사용자 이름·컴퓨터 이름·토큰 같은 개인 정보는 `<…>` 자리표시자로 바꾸었습니다.

## 0. 먼저 읽기

### 0.1 이 매뉴얼의 성격

- **따라 하기 매뉴얼**입니다. 단계마다 ① 할 일 ② 입력할 명령 ③ 성공하면 보이는 화면 순서로 적었습니다.
- 처음이라면 **2~3시간**쯤 걸립니다. 대부분 내려받는 시간입니다.
- 이 환경은 **초기 구축 환경**(Hermes v0.22.2 시점)입니다. 이후 서버·폰으로 확장되었고, 그 내용은 별도 매뉴얼에서 다룹니다. 버전이 달라지면 화면과 명령이 달라질 수 있습니다.
- **완전한 재설치 보증서가 아닙니다.** 기록에 남지 않은 단계는 "확인되지 않음"으로 표시했습니다(15장).

### 0.2 읽는 방법

- 명령은 **한 줄씩** 복사해 붙여 넣고 `Enter`를 누르세요. 여러 줄을 한꺼번에 붙이면 두 줄이 한 줄로 합쳐져 오류가 납니다(실제로 여러 번 났습니다).
- ✅ **성공 화면**과 다르면 **다음 단계로 넘어가지 말고** 13장 "오류가 났을 때"를 먼저 보세요.
- 🔒 표시는 비밀번호·토큰처럼 **남에게 보여 주면 안 되는 값**입니다. 채팅, 캡처, 문서, 공개 저장소에 절대 붙이지 마세요.

### 0.3 용어

| 용어 | 쉬운 설명 |
| --- | --- |
| **Hermes Agent** | Claude·GPT 같은 AI를 연결해 일을 시키는 프로그램 |
| **PowerShell** | Windows에서 글자로 명령을 입력하는 창 |
| **WSL / Ubuntu** | Windows 안에 들어 있는 작은 리눅스 컴퓨터. Hermes를 여기에 설치 |
| **Docker** | AI가 실행하는 명령을 가둬 두는 "투명한 상자" |
| **Rootless Docker** | 관리자 권한 없이 일반 사용자 권한으로 도는 Docker. 더 안전하고 권한 오류가 적음 |
| **가상환경(venv)** | Hermes 전용으로 만든 별도 프로그램 보관함 |
| **OAuth 로그인** | API 키 대신 내 계정으로 로그인해서 연결하는 방식(구독 로그인) |
| **위임(delegation)** | 주 AI(Claude)가 다른 AI(GPT)에게 일을 넘기는 기능 |
| **도구(tools)** | AI가 쓸 수 있는 능력(파일 쓰기, 터미널 실행, 웹 검색 등). 위험한 것은 꺼 둠 |
| **Staging(임시 작업 폴더)** | AI의 결과를 Windows 폴더로 옮기기 전에 두는 중간 폴더 |

### 0.4 자리표시자 — 내 컴퓨터에 맞게 바꿀 값

| 자리표시자 | 뜻 | 확인하는 방법 |
| --- | --- | --- |
| `<WSL_사용자>` | Ubuntu 사용자 이름 | 3.4단계의 `whoami` 결과 |
| `<Windows_실험폴더>` | AI 실험용 Windows 폴더(예: D 드라이브의 새 폴더) | 직접 정하기 |
| `<WSL_실험폴더>` | 위 폴더를 Ubuntu에서 부르는 경로(`/mnt/d/…` 형태) | 3.5단계 |
| `<Hermes_폴더>` | Hermes를 내려받는 폴더(`/home/<WSL_사용자>/hermes-agent`) | 6장 |
| `<인증_코드>` | 로그인 때 화면에 나오는 짧은 코드 🔒 | 매번 다름 |
| `<셋업_토큰>` | Claude 구독 로그인 때 나오는 긴 토큰 🔒 | **절대 기록하지 않음** |

> 이 문서의 명령에서 `<WSL_사용자>`를 본인 이름으로 바꿔 넣으세요. 노란색으로 표시된 부분이 바꿀 자리입니다.

### 0.5 전체 순서 한눈에 보기

```text
[1장]  선택: Windows 데스크톱 앱 (가장 쉬움, 단 Docker 격리 없음)
[2장]  원칙과 지켜야 할 것
[3장]  WSL·Ubuntu 상태 확인
[4장]  Ubuntu에 Docker 설치
[5장]  Docker를 일반 사용자 모드(Rootless)로
[6장]  Hermes 내려받기·설치 (Python 3.14)
[7장]  Docker 상자 안에서만 일하게 설정 + 위험한 도구 끄기
[8장]  GPT 연결 (ChatGPT 구독 로그인)
[9장]  Claude 연결 (Claude 구독 로그인)
[10장] Claude=총괄, GPT=검증관 지정 + 첫 협업 시험
[11장] 안전한 작업 폴더(staging) 설정
[12장] 매일 사용하는 방법
[13장] 오류가 났을 때
[14장] 스킬 승인 설정(선택)과 보안 점검 결과
[15장] 확인되지 않은 것 · 다음 단계
```

## 1. (선택) Windows 데스크톱 앱으로 시작하기

처음에는 이 방법으로 설치했습니다. **가장 쉽지만 AI가 Windows 파일에 직접 접근할 수 있고 Docker 격리가 없어서**, 이후 WSL 방식으로 옮겼습니다. 안전하게 쓰려면 **3장부터** 진행하세요.

1. 공식 배포 페이지(Nous Research의 Hermes Agent releases)에서 Windows용 설치 파일을 내려받아 실행합니다.
2. 가운데 **INSTALL** 버튼을 누릅니다. 설치 중에는 창을 닫거나 취소하지 마세요(10~15분 넘게 멈추면 그때 확인).
3. 설치가 끝나면 Windows 검색창에 `Hermes`를 입력해 **앱**을 실행합니다(설치 파일을 다시 누르지 마세요).
4. 첫 화면의 연결 마법사는 건너뛰어도 됩니다.
5. AI 연결: **Settings → Providers**에서 ChatGPT 구독 로그인(`Connected` 확인)과 Claude 연결을 설정합니다.
6. 확인해 둘 설정 위치: Model(주 AI), Advanced → Subagents(검증 AI), Advanced → Tool access, Safety → Approvals.

설정 파일을 바꾸기 전에 **백업**하세요.

```powershell
Copy-Item -LiteralPath "$env:LOCALAPPDATA\hermes\config.yaml" -Destination "$env:LOCALAPPDATA\hermes\config.backup.yaml" -ErrorAction Stop
```

## 2. 원칙 — 시작하기 전에 꼭

1. **구독 로그인을 우선**합니다. ChatGPT 구독(Codex OAuth)과 Claude 구독으로 연결하고, **별도 API 키를 추가하면 종량제 요금이 나갈 수 있으니** 먼저 확인하세요.
2. **말로 하는 지시는 보안 장치가 아닙니다.** "파일은 건드리지 마"라고 적어도 실제로 막는 것은 **도구 끄기와 Docker 격리**입니다.
3. **기존 소중한 파일 폴더를 Docker에 연결하지 않습니다.** AI가 지울 수 있습니다.
4. 설정을 바꾸기 전에 **백업**합니다.
5. `--yolo`(승인 우회), `-z`(한 번 실행·승인 우회 가능), `--safe-mode`(내 설정을 무시)는 **연결 시험용으로만** 쓰고 실제 작업에는 쓰지 않습니다.
6. 확인하지 않은 것은 "성공"이라고 하지 않습니다.
7. 명령은 **한 번에 하나씩**, 결과를 확인한 뒤 다음으로 갑니다.

## 3. WSL·Ubuntu 상태 확인

**PowerShell 여는 법:** Windows 키 → `PowerShell` 입력 → **Windows PowerShell** 클릭(관리자 권한 필요 없음).

> 이 매뉴얼의 모든 명령은 PowerShell 창에 입력합니다. `wsl -d Ubuntu --`로 시작하는 명령은 "Ubuntu에게 이 명령을 대신 실행해 줘"라는 뜻입니다.

### 3.1 WSL이 설치되어 있는지

```powershell
wsl --status
```

✅ 성공 화면: `기본 배포: Ubuntu`, `기본 버전: 2`

```powershell
wsl -l -v
```

✅ `Ubuntu` 줄에 `VERSION 2`가 보이면 됩니다(`Stopped`여도 괜찮음).

> "WSL이 설치되어 있지 않다"고 나오면 이 기록이 다루지 않은 상황입니다. Microsoft 공식 안내로 WSL을 먼저 설치하세요.

### 3.2 Ubuntu 버전

```powershell
wsl -d Ubuntu -- cat /etc/os-release
```

✅ 24.04 계열(예: `Ubuntu 24.04.x LTS`, 코드명 `noble`)

### 3.3 systemd(자동 서비스 관리) 확인

```powershell
wsl -d Ubuntu -- ps -p 1 -o comm=
```

✅ `systemd` 한 단어

### 3.4 Ubuntu 사용자 이름 확인 (중요)

```powershell
wsl -d Ubuntu -- whoami
```

나온 이름을 적어 두세요. 이후 명령의 `<WSL_사용자>`를 이 이름으로 바꿉니다.

### 3.5 실험 폴더가 Ubuntu에서 보이는지

```powershell
wsl -d Ubuntu -- ls -ld <WSL_실험폴더>
```

✅ `drwx… <WSL_실험폴더>` 같은 한 줄이 나옵니다.

## 4. Ubuntu에 Docker 설치

> Windows용 "Docker Desktop"은 설치하지 않았습니다. Ubuntu 안에 직접 설치했습니다.
> `sudo`가 붙은 명령은 Ubuntu 비밀번호 🔒를 물을 수 있습니다. **입력해도 화면에 글자가 안 보이는 것이 정상**이며, 비밀번호는 **본인이 직접 입력**합니다.

### 4.1 Docker가 이미 있는지

```powershell
wsl -d Ubuntu -- sh -c "command -v docker || echo DOCKER_NOT_FOUND"
```

- `DOCKER_NOT_FOUND` → 4.2로
- 경로가 나오면 이미 설치됨 → 4.6으로

### 4.2 준비 프로그램

```powershell
wsl -d Ubuntu -- sudo apt update
```

```powershell
wsl -d Ubuntu -- sudo apt install -y ca-certificates curl
```

✅ 마지막에 오류(`E:`) 없이 끝납니다.

### 4.3 Docker 공식 서명 키

```powershell
wsl -d Ubuntu -- sudo install -m 0755 -d /etc/apt/keyrings
```

```powershell
wsl -d Ubuntu -- sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

```powershell
wsl -d Ubuntu -- ls -l /etc/apt/keyrings/docker.asc
```

✅ `-rw-r--r-- … /etc/apt/keyrings/docker.asc`

### 4.4 Docker 공식 저장소 등록

아래는 **한 줄**입니다. 통째로 복사하세요.

```powershell
wsl -d Ubuntu -- sudo sh -c 'echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable" > /etc/apt/sources.list.d/docker.list'
```

✅ 아무 글자도 안 나오고 다음 줄로 넘어가면 성공입니다.

```powershell
wsl -d Ubuntu -- sudo apt update
```

✅ 맨 위에 `Get:1 https://download.docker.com/linux/ubuntu noble InRelease`가 보입니다(`… packages can be upgraded` 문구는 무시).

### 4.5 Docker 설치

```powershell
wsl -d Ubuntu -- sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

`Do you want to continue? [Y/n]`이 나오면 `Y` → `Enter`.

### 4.6 Docker 동작 확인

{% raw %}
```powershell
wsl -d Ubuntu -- sudo docker info --format '{{.ServerVersion}}'
```
{% endraw %}

```powershell
wsl -d Ubuntu -- sudo docker run --rm hello-world
```

✅ 버전 숫자가 나오고, `Hello from Docker!`가 보입니다.

## 5. Docker를 "일반 사용자 모드(Rootless)"로 바꾸기

**왜 하나요?** 4장의 Docker는 관리자(`sudo`)로만 동작합니다. Hermes는 일반 사용자로 실행되므로 그대로 두면 `permission denied`가 납니다. 또 일반 사용자 Docker가 더 안전합니다.

### 5.1 필요한 프로그램 확인

```powershell
wsl -d Ubuntu -- dockerd-rootless-setuptool.sh check
```

- `Missing system requirements … uidmap` → 5.2로
- `Requirements are satisfied` → 5.3으로

### 5.2 uidmap 설치

```powershell
wsl -d Ubuntu -- sudo apt-get install -y uidmap
```

### 5.3 Rootless Docker 설치

```powershell
wsl -d Ubuntu -- dockerd-rootless-setuptool.sh install
```

✅ 중요한 줄:

```text
Active: active (running)
Installed docker.service successfully.
Current context is now "rootless"
```

### 5.4 시작 및 시험

```powershell
wsl -d Ubuntu -- systemctl --user start docker.service
```

```powershell
wsl -d Ubuntu -- docker run --rm --network none alpine echo HERMES_DOCKER_OK
```

✅ 마지막 줄에 `HERMES_DOCKER_OK`

{% raw %}
```powershell
wsl -d Ubuntu -- docker info --format '{{.SecurityOptions}}'
```
{% endraw %}

✅ 출력에 `rootless`가 들어 있으면 됩니다.

> 설치 로그의 `No io.max … support` 경고는 동작에 문제가 없었습니다.
> 컴퓨터를 켤 때 자동 시작하게 하는 `loginctl enable-linger` 안내가 나왔지만 **이 기록에서는 실행하지 않았습니다**(자동 시작은 미확인).

## 6. Hermes 내려받기·설치

> ⚠ **가장 중요한 함정:** Hermes v0.22.2는 **Python 3.14에서만** 필요한 부품이 설치됩니다. Ubuntu 기본 Python(3.12)으로 설치하면 "설치는 성공"한 것처럼 보이지만 실행하면 `No module named …` 오류가 납니다. 그래서 **Python 3.14 전용 가상환경(`.venv314`)** 을 따로 만듭니다.

### 6.1 기본 도구 확인

```powershell
wsl -d Ubuntu -- bash -lc 'python3 --version; git --version; curl --version | head -n 1'
```

✅ Python, git, curl 버전 세 줄

### 6.2 pip·venv

```powershell
wsl -d Ubuntu -- sudo apt-get install -y python3-pip python3-venv
```

```powershell
wsl -d Ubuntu -- python3 -m pip --version
```

### 6.3 Hermes 원본 내려받기 (v0.22.2 고정)

```powershell
wsl -d Ubuntu -- git clone --branch v0.22.2 --depth 1 https://github.com/NousResearch/hermes-agent.git <Hermes_폴더>
```

✅ `Updating files: 100% (…), done.`(폴더가 이미 있으면 실패하며 기존 폴더를 덮어쓰지 않음)

### 6.4 설치 도우미(uv)용 임시 보관함

```powershell
wsl -d Ubuntu -- python3 -m venv <Hermes_폴더>/.venv
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv/bin/python -m pip install uv
```

✅ `Successfully installed uv-…`

### 6.5 Python 3.14 설치

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv/bin/uv python install 3.14
```

✅ `Installed Python 3.14.x`(`… is not on your PATH` 경고는 무시)

### 6.6 Hermes 전용 Python 3.14 보관함

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv/bin/uv venv --python 3.14 <Hermes_폴더>/.venv314
```

### 6.7 Hermes 설치

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv/bin/uv pip install --python <Hermes_폴더>/.venv314/bin/python -e <Hermes_폴더>
```

✅ `Installed 68 packages`(숫자는 버전에 따라 다름)

### 6.8 설치 확인

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes --version
```

✅ 성공 화면:

```text
Hermes Agent v0.22.2 (…) · stable
Python: 3.14.x
OpenAI SDK: …
```

> **앞으로 Hermes 실행 파일은 항상** `<Hermes_폴더>/.venv314/bin/hermes` 입니다(`.venv`가 아니라 `.venv314`).

## 7. Docker 상자 안에서만 일하게 설정하고 위험한 도구 끄기

설정 파일은 Ubuntu의 `/home/<WSL_사용자>/.hermes/config.yaml`에 저장됩니다. 아래를 하나씩 입력하세요. 각 명령의 성공 화면은 `✓ Set terminal.… = …` 입니다.

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set terminal.backend docker
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set terminal.docker_network false
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set terminal.container_persistent false
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set terminal.degraded_mode error
```

| 설정 | 뜻 |
| --- | --- |
| `backend docker` | AI의 명령을 Docker 상자 안에서 실행 |
| `docker_network false` | 상자 안에서 인터넷 사용 금지(Claude·GPT 통신은 별개라 계속 됨) |
| `container_persistent false` | 작업이 끝나면 상자를 버림 |
| `degraded_mode error` | Docker가 고장 나면 **상자 없이 실행하지 말고 멈춤**(기본값 `warn`은 위험) |

### 7.1 설정 백업 (설정 파일이 생긴 뒤에)

```powershell
wsl -d Ubuntu -- bash -lc "cp -p /home/<WSL_사용자>/.hermes/config.yaml /home/<WSL_사용자>/.hermes/config.yaml.backup && chmod 600 /home/<WSL_사용자>/.hermes/config.yaml.backup && ls -l /home/<WSL_사용자>/.hermes/config.yaml.backup"
```

✅ `-rw------- … config.yaml.backup`(첫 `config set` **전에** 하면 `No such file or directory`로 실패합니다. 정상)

### 7.2 점검

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes doctor
```

✅ `✓ Python 3.14.x`, `✓ Docker (daemon running)`. 아직 AI를 연결하지 않았으므로 `No API key found`, `OpenAI Codex auth (not logged in)` 경고는 정상입니다.

### 7.3 위험한 도구 끄기

AI가 파일을 마음대로 쓰거나 명령을 실행하지 못하게 먼저 끕니다.

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes tools disable terminal file code_execution computer_use browser
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes tools disable cronjob connections skills
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes tools list
```

✅ `terminal`, `file`, `code_execution`, `browser`, `computer_use`, `skills`, `connections`, `cronjob` 줄이 모두 `✗ disabled`

> `browser-use`, `desktop_ui` 같은 이름은 이 버전에 없어 `Unknown toolset`이 나옵니다. 무시하세요.
> **`delegation`(위임)은 10장의 협업에 필요하므로 끄지 않습니다.**
> 기록 당시 CLI에서 켜진 것: `web`, `vision`, `image_gen`, `tts`, `todo`, `memory`, `session_search`, `clarify`, `delegation`. `memory`와 `web`은 저장·외부 통신 가능성이 있다는 점을 기억하세요.

## 8. GPT 연결 (ChatGPT 구독 로그인)

ChatGPT 구독으로 연결합니다. **OpenAI API 키(종량제)는 쓰지 않습니다.**

### 8.1 로그인 시작

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes auth add openai-codex --type oauth
```

화면에 안내가 나옵니다.

```text
1. Open this URL in your browser:  (OpenAI의 기기 인증 주소)
2. Enter this code:  <인증_코드>      ← 🔒 매번 다른 코드
Waiting for sign-in...
```

### 8.2 브라우저에서 승인

1. 화면에 나온 주소를 브라우저로 엽니다.
2. ChatGPT 계정으로 로그인합니다.
3. PowerShell에 보이는 코드를 입력하고 승인합니다.
4. PowerShell로 돌아오면 ✅ `Added openai-codex OAuth credential #1`

### 8.3 GPT 모델 선택

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes model
```

화살표 키(↑↓)와 `Enter`로 차례대로 고릅니다.

1. `OpenAI ▸ (ChatGPT/Codex subscription or direct OpenAI API)`
2. `ChatGPT or Codex Subscription`
3. `Use existing credentials`
4. 목록에서 GPT 모델 선택(기록 당시 `gpt-6-sol`)
5. 추론 강도: `high`

✅ `Default model set to: gpt-6-sol (via OpenAI Codex)`

### 8.4 응답 시험

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes --safe-mode -m gpt-6-sol -z "Reply with exactly: GPT_OK"
```

✅ `GPT_OK`

> `-z`와 `--safe-mode`는 **연결 시험용**으로만 씁니다. `--yolo`는 절대 쓰지 않습니다.

## 9. Claude 연결 (Claude 구독 로그인)

Hermes는 Claude 구독 로그인에 **Claude Code 프로그램**을 씁니다. 먼저 설치합니다.

### 9.1 Node.js·npm

```powershell
wsl -d Ubuntu -- sudo apt-get update
```

```powershell
wsl -d Ubuntu -- sudo apt-get install -y nodejs npm
```

```powershell
wsl -d Ubuntu -- bash -lc 'node --version; npm --version'
```

### 9.2 Claude Code 설치

```powershell
wsl -d Ubuntu -- sudo npm install -g @anthropic-ai/claude-code
```

```powershell
wsl -d Ubuntu -- claude --version
```

✅ `… (Claude Code)` 버전 표시

> 설치 중 `npm WARN EBADENGINE … node >=22` 경고가 나왔습니다. Node가 낮아도 이번 용도(로그인 토큰 발급)에서는 동작했지만, **다른 기능까지 정상이라는 보장은 없습니다.**

### 9.3 Hermes에서 Claude 로그인

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes model
```

1. `Anthropic (Claude models via API key or Claude Code)` 선택
2. 인증 방식에서 **Claude Pro/Max subscription (OAuth login)** 선택(`1` 입력)
3. Hermes가 브라우저 주소를 보여 줍니다. 안 열리면 주소를 복사해 브라우저에 붙입니다.
4. 브라우저에서 로그인·승인 → **짧은 인증 코드** 🔒가 나오면 복사해 PowerShell의 `Paste code here if prompted >`에 붙여 넣고 `Enter`
5. 인증이 끝나면 **긴 셋업 토큰** 🔒이 표시됩니다. 이 값을 `Paste setup-token (or Enter to cancel):`에 붙여 넣고 `Enter`
6. 모델 선택(기록 당시 `claude-sonnet-5`) → 추론 강도 `high`

✅ `✓ Setup-token saved.` / `Default model set to: claude-sonnet-5 (via Anthropic)`

> ❗ 4번과 5번은 **서로 다른 값**입니다. 짧은 코드 칸에 긴 토큰을 넣으면 `OAuth error: Invalid code`가 납니다(13장).
> 🔒 **셋업 토큰은 절대 기록하거나 공유하지 마세요.** 이 문서에도 싣지 않았습니다.

### 9.4 Claude 전용 부품(SDK) 설치

이 버전은 자동 설치하지 못합니다. 직접 설치합니다.

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv/bin/uv pip install --python <Hermes_폴더>/.venv314/bin/python "anthropic==0.87.0"
```

✅ `+ anthropic==0.87.0`

### 9.5 응답 시험

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes --safe-mode -m claude-sonnet-5 -z "Reply with exactly: CLAUDE_OK"
```

✅ `CLAUDE_OK`

### 9.6 두 계정이 등록됐는지

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes auth list
```

✅ `anthropic (1 credentials)`와 `openai-codex (1 credentials)` 두 묶음

## 10. Claude=총괄, GPT=검증관으로 지정하고 협업 시험

### 10.1 주 AI를 Claude로 고정

8·9장에서 `hermes model`을 마지막에 실행한 쪽이 기본 AI가 됩니다. 확실하게 지정합니다.

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set model.provider anthropic
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set model.default claude-sonnet-5
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config get model
```

✅ `default: claude-sonnet-5`, `provider: anthropic`

### 10.2 검증 AI를 GPT로 지정

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set delegation.model gpt-6-sol
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set delegation.provider openai-codex
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes tools enable delegation
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config get delegation
```

✅ `model: gpt-6-sol`, `provider: openai-codex`, **`subagent_auto_approve: false`**(마지막 값은 반드시 `false`)

> 지정하지 않으면 GPT가 아니라 **Claude가 Claude에게** 일을 넘깁니다.

### 10.3 대화창 열기

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes --cli
```

✅ 아래쪽 상태 줄에 `claude-sonnet-5`가 보입니다. 시작 화면에 `delegate_task`가 보이는지 확인하세요.

### 10.4 첫 협업 시험 — 아래 글을 **대화창에** 붙여 넣기

```text
GPT 서브에이전트에게 17 × 23을 계산하도록 위임해. GPT의 응답을 받은 뒤 결과를 보고해. 파일 생성·수정, 터미널 실행, 웹 검색은 하지 마. 실제 위임에 실패하면 성공했다고 보고하지 마.
```

✅ 성공 화면:

```text
delegate  1x: 17 × 23을 계산해서 ...
Subagent Task Completed
서브에이전트(모델: gpt-6-sol)가 실제로 위임을 완료했습니다.
17 × 23 = 391
```

끝낼 때는 `/exit` 또는 `Ctrl + C`.

### 10.5 조금 더 의미 있는 시험 (기록에서 성공)

파일을 쓰지 않는 대화 전용 시험으로, Claude가 먼저 의견을 정리하고 GPT에게 독립 검토를 받은 뒤 공통점·차이점·결론을 보고하게 했습니다. 사용한 주제: *"AI 에이전트가 기존 파일을 수정·삭제하기 전에 사용자 승인을 받도록 설계할 때 가장 중요한 안전장치 3가지는?"*

- 결과: Claude가 정리한 3가지는 **변경 내용 사전 공개 / 복구 가능성 확보 / 승인 범위 제한**이었고, GPT는 **승인 시점과 실행 시점 사이에 대상이 바뀌면 다시 승인**해야 한다는 조건을 강조했습니다.
- 위임 기록으로 확인: 위임 완료(`completed`), 수십 초 이내.
- 이 시험은 **한 번의 성공 사례**이며, 두 AI가 정면으로 대립하는 상황은 검증하지 않았습니다.

> 이 문장은 **행동 지시**일 뿐 기술적 차단 장치가 아닙니다. 시험 과정에서 서비스 사용량이 소모될 수 있습니다.

## 11. 안전한 작업 폴더 설정 (staging)

**왜?** Docker 상자에 Windows 실험 폴더를 직접 연결하면 AI가 그 안의 **기존 파일을 지울 수 있습니다**(14장). 그래서 상자에는 Ubuntu 안의 **임시 폴더**만 연결하고, 결과물은 **사람이 확인한 뒤** 옮깁니다.

### 11.1 임시 폴더 만들기

```powershell
wsl -d Ubuntu -- bash -lc "mkdir -p /home/<WSL_사용자>/hermes-work-staging && chmod 700 /home/<WSL_사용자>/hermes-work-staging && ls -ld /home/<WSL_사용자>/hermes-work-staging"
```

✅ `drwx------ … hermes-work-staging`

### 11.2 상자에 임시 폴더만 연결

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set terminal.docker_volumes '["/home/<WSL_사용자>/hermes-work-staging:/workspace:rw"]'
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set terminal.docker_mount_cwd_to_workspace false
```

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config set terminal.cwd /workspace
```

### 11.3 최종 확인

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config get terminal
```

✅ 아래 값이 모두 보이면 성공:

```yaml
backend: docker
degraded_mode: error
cwd: /workspace
docker_volumes:
  - /home/<WSL_사용자>/hermes-work-staging:/workspace:rw
docker_mount_cwd_to_workspace: false
docker_network: false
container_persistent: false
```

### 11.4 더 안전한 방식 — 참조 폴더는 읽기 전용, 결과 폴더만 쓰기 (이후 시험에서 확인)

10월 26일 보고서에서는 **읽기 전용 참조 폴더**와 **쓰기 가능한 결과 폴더**를 분리한 설정을 시험했고, 두 동작이 모두 확인되었습니다.

```yaml
terminal:
  backend: docker
  docker_network: false
  container_persistent: false
  cwd: /workspace/output
  docker_volumes:
    - /home/<WSL_사용자>/hermes-work-staging:/workspace/reference:ro
    - /home/<WSL_사용자>/hermes-output:/workspace/output:rw
  docker_run_as_host_user: false
  degraded_mode: error
  docker_mount_cwd_to_workspace: false
```

| 시험 | 결과 |
| --- | --- |
| 참조 폴더 읽기 | 가능 |
| 참조 폴더 쓰기 시도 | **차단됨**(파일 미생성) |
| 결과 폴더 쓰기 | 성공 |

- `docker_run_as_host_user: false`는 Rootless Docker의 사용자 번호 매핑 때문에 결과 폴더 쓰기 시험이 통과하도록 조정한 값입니다.
- **한계:** 이 보호는 **그 컨테이너 안에서만** 적용됩니다. 다른 도구가 호스트(WSL/Windows) 파일에 직접 접근하는 경로까지 자동으로 막지는 않습니다.

### 11.5 결과물을 Windows로 옮기기

- 결과는 **사람이 확인한 뒤 새 파일만** Windows 폴더로 복사합니다. **같은 이름의 기존 파일은 덮어쓰지 않습니다.**
- 기록에는 임시 폴더의 파일 중 "새 파일"과 "이미 있는 파일"을 **읽기 전용으로 구분해 보여 주는 보조 프로그램**이 있었습니다(복사는 하지 않음, 허용 하위 폴더 제한, 20MB 초과·바로가기 파일 거부). 그 프로그램의 전체 소스와 정확한 실행 인자는 이 문서의 자료만으로 확정할 수 없어 **싣지 않았습니다.**

## 12. 매일 사용하는 방법

1. PowerShell을 엽니다.
2. Docker를 깨웁니다(이미 켜져 있어도 문제없음).

```powershell
wsl -d Ubuntu -- systemctl --user start docker.service
```

3. Hermes 대화창을 엽니다.

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes --cli
```

4. 상태 줄에 `claude-sonnet-5`가 보이는지 확인합니다.
5. 끝낼 때는 `/exit`.

**상태만 확인하는 명령(아무것도 바꾸지 않음)** — 한 줄씩 따로 입력:

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes --version
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config get model
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config get delegation
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes config get terminal
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes tools list
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes auth list
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes mcp list
```

> ⚠ `hermes model`은 **조회 명령이 아닙니다.** 기본 AI를 바꿀 수 있으니 모델을 바꿀 때만 실행하세요.
> `hermes mcp list`에서 `No MCP servers configured.`가 나오는 것은 오류가 아니라 **연결된 외부 도구가 없다**는 뜻입니다.

## 13. 오류가 났을 때

| 화면에 나온 글자 | 원인 | 해결 |
| --- | --- | --- |
| `docker: … 인식되지 않습니다`(cmdlet) | Windows PowerShell에는 Docker가 없음(정상) | 앞에 `wsl -d Ubuntu --`를 붙여 Ubuntu에서 실행 |
| `E: Unable to locate package …wsl`, `unknown shorthand flag: 'd'` | 명령 두 줄이 한 줄로 붙어 입력됨 | 한 줄씩 다시 복사 |
| `/bin/bash: line 1: … command not found`(grep) | PowerShell이 `|`를 잘못 해석 | `\|` 대신 `-e 단어 -e 단어` 형식 사용 |
| `permission denied while trying to connect to the docker API …` | 관리자용 Docker에 일반 사용자가 접근 | 5장 Rootless Docker |
| `failed to connect to the docker API at … no such file or directory` | Rootless Docker가 꺼져 있음(WSL을 껐다 켠 뒤 자주 발생) | `wsl -d Ubuntu -- systemctl --user start docker.service` |
| `ModuleNotFoundError: No module named …` / `OpenAI SDK: Not installed` | Python 3.12로 설치해 부품이 안 깔림 | 6.4~6.7로 `.venv314`에 다시 설치, 실행은 `.venv314` 경로로 |
| `python3.14: command not found` / `uv: command not found` | 기본으로 없음 | 6.4(uv), 6.5(Python 3.14) |
| `cp: cannot stat …config.yaml` | 설정 파일이 아직 없음 | 7장 `config set`을 한 번 실행한 뒤 백업 |
| `✗ Unknown toolset '…'` | 이 버전에 없는 도구 이름 | 무시 |
| `The 'anthropic' package is required …` | Claude 부품 미설치 | 9.4 |
| `KeyError: 'unknown package: anthropic'`(`pm install`) | `pm install`로는 설치 불가 | 9.4의 명령 사용 |
| `The 'claude' CLI is required for OAuth login` | Claude Code 없음 | 9.1, 9.2 후 9.3 다시 |
| `npm WARN EBADENGINE …` | Node 버전이 낮음 | 경고만, 로그인엔 문제없었음 |
| `OAuth error: Invalid code …` | 짧은 인증 코드 칸에 긴 토큰을 넣었거나 일부만 복사 | `Enter`로 재시도 → **짧은 코드 전체**를 입력 |
| `The 'claude-sonnet-5' model is not supported when using Codex …` | 주 AI 제공자가 GPT로 남아 있는데 모델만 Claude | 10.1의 두 명령 후 대화창 다시 열기 |
| `hermes config: 'list' is not a hermes config command` | 없는 명령 | 전체 보기는 `hermes config show` |
| `Config version outdated (v0 → v…)`(doctor) | 새 설정 항목이 있다는 안내 | 당장 필요 없음. 하려면 먼저 백업 후 `hermes doctor --fix`(기록에서는 실행하지 않음) |
| `~/.local/bin/hermes not found`(doctor) | 짧은 이름 `hermes`로 실행 못 함 | 전체 경로로 실행하면 문제없음 |

## 14. 스킬 승인 설정(선택)과 보안 점검 결과

### 14.1 스킬 변경에 사용자 승인 요구하기

Hermes는 백그라운드에서 스킬을 자동으로 만들거나 고치는 검토를 할 수 있습니다. 이 기능은 유지하되, **스킬 변경에 승인 절차를 적용**했습니다.

1. **먼저 설정 백업**

```powershell
wsl -d Ubuntu -- cp -n /home/<WSL_사용자>/.hermes/config.yaml /home/<WSL_사용자>/.hermes/config.yaml.before_skill_approval
```

2. 사용자 승인 후 설정 끝에 **추가**(덮어쓰지 않음)

```yaml
skills:
  write_approval: true
```

3. 코드가 설정을 인식하는지 확인

```powershell
wsl -d Ubuntu -- bash -lc 'cd <Hermes_폴더> && ./.venv314/bin/python -c "from tools.write_approval import write_approval_enabled, SKILLS; print(write_approval_enabled(SKILLS))"'
```

변경 전 `False`, 변경 후 `True`였습니다.

- **확인한 것:** 코드가 설정을 `True`로 읽음.
- **확인하지 못한 것:** 실행 중인 Hermes에 반영되었는지(재시작 여부), 실제로 승인 대기·승인·거부가 동작하는지.
- **범위:** 스킬 관리 작업에만 해당합니다. **터미널·파일 도구를 통한 변경 전체를 막는 장치가 아닙니다.**

### 14.2 보안 점검에서 확인된 위험 (중요)

2026-10-24 읽기 전용 점검의 결론은 **"조건부 위험"**이었습니다. 실제 악용이 확인된 것이 아니라, 확인된 자동 승인·우회 경로와 쓰기 가능한 폴더에 근거한 평가입니다.

모의 판정 명령(실제로 지우지 않고 판정만 보여 줌):

```powershell
wsl -d Ubuntu -- <Hermes_폴더>/.venv314/bin/hermes approvals test --env-type docker -- rm -f /workspace/기존파일.txt
```

```text
verdict : allow  (exit 0)
detail  : env_type 'docker' is an isolated container backend; the runtime skips all command guards for it except approvals.deny
```

**뜻:** Docker 방식에서는 Hermes가 "위험한 명령이니 승인하시겠습니까?"를 **묻지 않습니다.** 상자 밖(Windows 파일)은 보호되지만, **상자에 연결된 쓰기 가능 폴더 안의 파일은 AI가 지울 수 있습니다.** 그래서 11장처럼 **상자에는 임시·결과 폴더만** 연결하고, 기존 자료는 읽기 전용으로 둡니다.

점검에서 확인된 소스 사실(요약):

- 승인 모드 기본값은 `manual`이지만, `manual`이 모든 파일 변경의 개별 사람 승인을 뜻하지는 않습니다(영구 허용 목록, 비대화식 문맥 등 자동 허용 경로가 있음).
- 하위 AI(위임받은 AI)의 도구는 부모의 권한 범위로 제한되도록 설계되어 있으나, 실제 실행 시의 우회 경로 부재까지는 시험하지 않았습니다.
- 권장 보호: 기존 자료는 읽기 전용 마운트, AI가 쓸 폴더는 분리, **자연어 지시를 최종 장벽으로 삼지 않기**, 변경이 필요하면 사람이 대상 파일을 확인하고 별도 승인.

## 15. 확인되지 않은 것 · 다음 단계

### 15.1 확인하지 못한 것

| 항목 | 상태 |
| --- | --- |
| 기존 파일 변경 시 **반드시 승인창이 뜨게 하는 장치** | 미구현 |
| 컴퓨터를 다시 켠 뒤 Docker 자동 시작 | 미확인(12장 2번처럼 직접 켜기) |
| 위임받은 AI의 **실제 실행 격리** | 코드상 일부 확인, 실행 시험 미완료 |
| 모든 Hermes 도구의 Docker 격리 | 미검증 |
| 스킬 승인 흐름의 실동작 | 미시험 |
| Node 18에서 Claude Code 전체 기능 | 미확인 |
| 실제 요금·청구 | 청구 로그를 열어 확인하지 않음. 구독 사용량과 API 종량제는 **별개** |
| 모델 이름(`claude-sonnet-5`, `gpt-6-sol`) | 기록 당시 목록의 이름. 바뀌면 `hermes model` 목록의 이름 사용 |
| 처음부터의 **완전 재설치 명령 전체** | 일부 단계는 기록에 없음 |
| Telegram 원격 사용, 상시 서버 운영 | 이 단계에서는 진행하지 않음 |

### 15.2 다음 단계 (기록에서 이어진 순서)

1. 파일 변경 승인을 강제하는 장치 설계·시험(승인 거부, 변경 내용 불일치, 경로 우회)
2. 안전한 신규 파일 반입과 결과 자동 저장 연결
3. Claude/GPT 의견·검증 기록 자동 생성
4. 비용 기록, 오류 대응, 복구 절차 보강
5. 상시 서버로 확장(메신저 연결, 백업) → 별도 매뉴얼

### 15.3 작성자 권장 (기록이 아니라 일반 권고)

- 7.3의 **도구 끄기 → 10.4 협업 시험** 순서를 지키고, 한 단계가 끝날 때마다 `config get`으로 결과를 확인하세요.
- 쓰기 가능한 폴더에는 **중요한 원본을 두지 마세요.** 백업을 먼저 만드세요.
- 인증이 끝난 뒤에는 `hermes auth list`로 **두 계정이 모두 등록**됐는지 꼭 확인하세요.

## 부록 A. 빠른 점검표

```text
[ ] 이 명령은 PowerShell에서 wsl -d Ubuntu -- 로 보내는가?
[ ] 한 줄씩 붙여 넣었는가? (여러 줄 한꺼번에 X)
[ ] <WSL_사용자>와 <Hermes_폴더>를 내 값으로 바꿨는가?
[ ] sudo 비밀번호는 내가 직접 입력했는가? (남에게 알리지 않음)
[ ] 토큰·코드를 채팅·화면 캡처·문서에 붙이지 않았는가?
[ ] 설정을 바꾸기 전에 백업했는가?
[ ] 위험한 도구는 꺼져 있는가? (tools list)
[ ] subagent_auto_approve 는 false 인가?
[ ] 상자에 연결한 폴더에 중요한 원본이 없는가?
[ ] 성공 화면과 같은 결과인가? (다르면 13장)
```

## 부록 B. 하지 말아야 할 것

- `--yolo`, 실제 작업에서의 `-z`·`--safe-mode`
- 중요한 기존 폴더를 Docker에 **쓰기 가능으로** 연결
- 끈 도구(`terminal`, `file`, `code_execution`, `browser`, `computer_use`)를 범위를 정하지 않고 켜기
- `delegation.subagent_auto_approve`를 `true`로 바꾸기
- 토큰·API 키·인증 코드·비밀번호를 기록·공유
- API 키를 추가하기 전에 요금을 확인하지 않기
- "말로 한 지시"를 보안 장치로 믿기
