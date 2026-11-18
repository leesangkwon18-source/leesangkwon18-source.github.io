---
title: "갤럭시 노트9로 만드는 독립 Hermes Agent 서버 — 초보자용 설치·운영·복구 매뉴얼"
---

* 목차
{:toc}


- **버전:** 게시본 v1.0 (홈페이지 게시용, 개인 환경 정보 가림 처리)
- **검증 기준일:** 2026-11-08 (2026-11-05 ~ 11-08에 실제로 수행한 기록을 정리)
- **독자:** Windows PowerShell, Linux, Android Termux를 처음 접하는 사람
- **성격:** 실제 작업 기록을 바탕으로 한 **운영·복구 매뉴얼**입니다. 처음부터 끝까지 재설치를 보장하는 "완전 재현서"는 아닙니다(4장 참고).

> **이 문서의 `<…>` 표시는 독자가 자기 값으로 바꿔 넣는 자리입니다.** 원본 기록에 있던 서버 주소, 계정 이름, 키 파일 이름, 기기 일련번호, 봇 이름, 해시값, PC 사용자 경로는 공개하지 않기 위해 모두 자리표시자로 바꿨습니다(부록 C).

---

## 0. 먼저 읽기 — 안전 원칙

1. **한 번에 명령어 하나씩** 실행하고, 실제 결과를 확인한 뒤 다음으로 넘어갑니다. 문서의 명령을 한꺼번에 붙여넣지 마세요.
2. **어느 기기에서 실행하는지** 매번 확인합니다. 이 문서는 모든 명령 위에 `실행 기기`를 표시합니다.
3. **재부팅, 서비스 시작·중지, 파일 권한 변경, 복원**은 영향이 큰 작업입니다. 현재 상태를 읽기 전용으로 확인한 뒤에만 하세요.
4. **SSH 개인키, 인증 파일, 봇 토큰, 백업 파일을 삭제·덮어쓰기·재발급하지 마세요.** 접속이 안 될 때 키부터 새로 만들지 마세요.
5. **비밀 값은 어디에도 붙여넣지 않습니다.** 채팅, 문서, 공개 저장소 모두 해당합니다. 명령 출력에 토큰이나 키가 보이면 가린 뒤 공유하세요.
6. 문서에 적힌 **"성공"은 그 날짜·그 범위의 관찰**입니다. 오래된 실패를 지우거나 최신 성공을 과거로 소급해 적지 않았습니다.

---

## 1. 한눈에 보기

### 1.1 구성도

```text
[외부에서 사용]  다른 스마트폰의 Telegram
                  │ 인터넷
                  ▼
        갤럭시 노트9 (Android 10)
         └ Termux (SSH 8022, runit 서비스, Termux:Boot)
             └ PRoot Ubuntu 24.04
                 └ Hermes Agent + Telegram Gateway
                     ├ 바코 (Claude, 구독 경로)
                     └ 지코 (GPT, Codex OAuth 경로)

[설정·관리할 때만]  메인 PC (Windows) ── SSH ──► 중계 서버 (Ubuntu)
                                         ├ USB / ADB ──► 노트9 Termux
                                         └ 또는 SSH 역방향 터널 ──► 노트9 Termux
```

- **독립 서비스:** Telegram으로 노트9의 Hermes와 대화하는 데는 중계 서버나 메인 PC가 필요하지 않다는 응답 시험이 있었습니다.
- **관리 경로:** SSH로 노트9 안에 들어가 점검·수정하려면 중계 서버가 켜져 있어야 합니다(USB/ADB 또는 역방향 터널).

### 1.2 2026-11-08 기준 상태 (검증 수준 구분)

| 항목 | 상태 | 근거와 한계 |
| --- | --- | --- |
| Android 10 + Termux + SSH | 확인 | 2026-11-05~08 기록 |
| PRoot Ubuntu 24.04 + Hermes Agent | 확인 | Hermes `v0.22.7+365`, Python 3.14.9 |
| 바코(Claude)·지코(GPT) 연결 | 구독 경로 이력 확인 | 실제 호출의 인증 경로·비용 여부는 추가 검증 필요 |
| Telegram 응답 | 확인 | 메인 PC·중계 서버 전원 OFF, 노트9 화면 OFF 상태의 **단기** 시험 |
| USB·ADB·SSH 접속 | 성공 | 반복 확인 |
| USB 없는 SSH 역방향 터널 | 성공 | 중계 서버가 켜져 있고 터널이 살아 있을 때만 |
| 재부팅 후 SSH·Gateway 자동 실행 | 확인(1회) | 재부팅 직후 두 서비스 `run` 확인 |
| 장시간 안정성, 배터리·발열, Wi-Fi 전환 | **미검증** | 실제 장기 시험 전 |
| 6.2GB 전체 백업 | 해시 3곳 동일 | 실제 **복원 시험은 하지 않음** |

---

## 2. 장비와 셸을 혼동하지 않는 법

| 이름 | 환경 | 하는 일 | 프롬프트 예시 |
| --- | --- | --- | --- |
| 메인 PC | Windows (PowerShell, 필요하면 WSL) | 원격 관리, 백업 보관 | `PS C:\Users\<사용자>>` |
| 중계 서버 | Ubuntu 단독 설치 PC | 노트9의 USB/ADB·SSH 관리 중계 | `<서버_계정>@<서버_호스트>:~$` |
| 노트9 Termux | Android 앱 기반 Linux 환경 | SSH, runit 서비스, PRoot 실행 | `~ $` |
| 노트9 PRoot Ubuntu | Termux 안의 Ubuntu 24.04 사용자 공간 | **주력 Hermes Agent / Gateway** | `root@localhost:~#` |
| 다른 스마트폰 | 일반 스마트폰 | 외부에서 Telegram으로 접속 | 앱 화면 |

**주의:** `root@localhost`만 보고 중계 서버라고 판단하지 마세요. PRoot의 `root`는 **Android의 실제 root 권한이 아닙니다.** 세 환경(PC/WSL, 중계 서버, 노트9)의 설치 파일과 인증 경로는 서로 다릅니다.

---

## 3. 준비물

- 갤럭시 노트9 (Android 10), USB 케이블, 안정적인 전원과 Wi-Fi
- 노트9에 **Termux**와 **Termux:Boot** 앱(공식 배포본), **proot-distro**로 설치한 Ubuntu 24.04
- 중계 서버: `adb`(Android platform-tools)가 설치된 Ubuntu, SSH 키로 접속 가능한 상태
- 메인 PC: Windows PowerShell과 SSH 클라이언트, SSH 키

> 이 매뉴얼은 위 환경이 이미 구성돼 있다는 전제에서 **접속·점검·복구·백업**을 다룹니다. 처음 설치 절차의 일부는 기록이 없어 4장에서 한계를 밝힙니다.

---

## 4. 설치는 실제로 어떻게 진행됐나

### 4.1 첫 시도 — Termux 단독 Hermes (11월 5일)

- Android 10(SDK 29), Termux Python 3.14.8, Node.js, uv를 사용했습니다.
- **Codex OAuth(지코) 등록은 성공**했습니다. 장치 코드 방식으로 브라우저에서 일회용 코드를 입력합니다(코드는 기록하지 않음).
- **Claude 구독(바코) 연결은 Termux 단독 환경에서 막혔습니다.** Claude Code CLI를 `npm`으로 설치해도 `claude native binary not installed`가 나왔고, 확인 결과 설치된 패키지의 배포 메타데이터에 Android용 네이티브 패키지 항목이 없었습니다(`E404`). Android 지원이 영구 중단됐다고 단정할 수는 없습니다.
- `advanced-delegation` 플러그인은 `watchfiles`/`maturin` 빌드가 `Failed to determine Android API level`로 실패했습니다. 원인 확인은 다음과 같았습니다.

  ```bash
  getprop ro.build.version.sdk
  ```

  Android 시스템 SDK는 29였지만 Python의 빌드 설정은 `android-24-arm64_v8a`였습니다. `ANDROID_API_LEVEL=29`로 다시 시도하면 만들어진 wheel이 현재 Python과 호환되지 않았고, 아래처럼 **Python 설정과 같은 값(24)** 을 지정했을 때 활성화됐습니다.

  ```bash
  ANDROID_API_LEVEL=24 LD_PRELOAD=$PREFIX/lib/libpython3.14.so <hermes 실행 경로> plugins enable --no-allow-tool-override advanced-delegation
  ```

  이 값은 **당시 환경의 진단 결과**이며 다른 기기에 그대로 쓰면 안 됩니다.
- `jico` 모델 경로(Model route)는 공급자 선택 화면에 `Custom endpoint`만 보여 **저장하지 못했습니다.** 임의의 endpoint나 API 키를 넣지 않고 취소했습니다.

### 4.2 주력 전환 — PRoot Ubuntu 24.04 (11월 6일 이후)

- Termux 단독 환경은 삭제하지 않고 **보존**했고, PRoot Ubuntu를 주력으로 전환했습니다.
- 진입: `proot-distro login ubuntu`
- Hermes 소스 `/root/hermes-agent`, 실행 파일 `/root/.local/bin/hermes`, 설정 `/root/.hermes`
- 확인한 버전: Hermes Agent `v0.22.7+365`, Python 3.14.9, OpenAI SDK 2.54.0
- **한계:** PRoot 안의 전체 설치 명령과 의존성 설치 순서는 남은 기록만으로 복원되지 않습니다. 이 문서는 검증되지 않은 설치 명령을 "성공한 것처럼" 쓰지 않습니다. **동작 중인 노트9를 초기화하거나 다시 설치하지 마세요.**

### 4.3 구독과 비용

- 바코=Claude, 지코=GPT입니다. 기록된 경로는 Claude 구독(DirectSDK), GPT Codex OAuth입니다.
- 인증 파일이 있다는 사실만으로 **모든 요청이 구독에 포함된다고 단정할 수 없습니다.**
- 종량제 키(`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`)로 임의 전환하지 마세요. 과금될 수 있는 시험은 비용을 확인한 뒤 하세요.

---

## 5. 원격 접속 ① — USB와 ADB 포트 전달 (실제 성공 절차)

연결이 이미 정상이라면 불필요하게 다시 실행하지 마세요.

### 5.1 메인 PC → 중계 서버
**실행 기기: 메인 PC / Windows PowerShell**

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소>
```

성공하면 `<서버_계정>@<서버_호스트>:~$`가 나옵니다. 이후 입력은 **중계 서버**에 들어갑니다. 서버 주소는 바뀔 수 있습니다.

### 5.2 노트9가 USB로 보이는지
**실행 기기: 중계 서버 / Ubuntu**

```bash
adb devices -l
```

- 처음에는 `* daemon not running; starting now`가 나올 수 있습니다. 오류가 아니라 ADB가 자동으로 시작됐다는 뜻입니다.
- 목록에 `device`가 보이면 정상입니다(기기 일련번호는 가림 처리: `<기기_일련번호> device usb:… model:SM_N960N`).
- **`unauthorized`이면** 노트9 화면을 켜고 잠금을 푼 뒤 "USB 디버깅을 허용하시겠습니까?"에서 **본인의 신뢰하는 중계 서버인지 확인하고 허용**합니다. 다시 `adb devices -l`을 실행해 `device`로 바뀌는지 확인합니다. (실제 기록에서도 이렇게 해결됐습니다.)

### 5.3 ADB 포트 전달 확인
**실행 기기: 중계 서버 / Ubuntu**

```bash
adb forward --list
```

결과가 비어 있으면 현재 포트 전달이 없는 것입니다. **USB를 다시 꽂거나 노트9가 재부팅되면 이 설정이 사라질 수 있습니다.**

### 5.4 포트 전달 만들기
**실행 기기: 중계 서버 / Ubuntu**

```bash
adb forward tcp:8022 tcp:8022
```

성공하면 `8022`가 출력됩니다. 이는 **중계 서버의 로컬 8022 포트를 USB로 연결된 노트9의 8022 포트로 연결**하는 설정입니다. 로그인이 끝난 것은 아닙니다.

### 5.5 중계 서버 → 노트9 Termux SSH
**실행 기기: 중계 서버 / Ubuntu**

```bash
ssh -i ~/.ssh/<노트9_접속_키> -p 8022 -o IdentitiesOnly=yes <Termux_사용자>@127.0.0.1
```

성공하면 `Welcome to Termux!`가 나옵니다. 이 `127.0.0.1`은 **중계 서버 자기 자신의 포트**이고, ADB가 그것을 노트9로 전달합니다.

옵션: `-i` 키 파일, `-p 8022` 포트, `IdentitiesOnly=yes` 지정한 키만 사용, `BatchMode=yes`는 비밀번호를 묻지 않고 성공·실패만 확인합니다.

### 5.6 노트9 Termux → PRoot Ubuntu
**실행 기기: 노트9 / Termux 셸**

```bash
proot-distro login ubuntu
```

`root@localhost:~#`가 나오면 이제부터는 **노트9 안의 Ubuntu**입니다.

### 5.7 Hermes 버전 확인
**실행 기기: 노트9 / PRoot Ubuntu**

```bash
/root/.local/bin/hermes --version
```

버전 정보가 표시되면 정상입니다. `Update available`은 안내일 뿐 오류가 아닙니다. 설정 보호를 위해 **바로 업데이트하지 마세요.**

### 5.8 PC에서 한 번에 확인하고 싶을 때(점검용)
**실행 기기: 메인 PC / Windows PowerShell**

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ssh -i ~/.ssh/<노트9_접속_키> -p 8022 -o IdentitiesOnly=yes -o BatchMode=yes <Termux_사용자>@127.0.0.1 'whoami; pwd'"
```

노트9 Termux의 사용자 이름과 홈 경로가 출력되면 접속이 정상입니다. PRoot 안은 중첩 따옴표가 꼬이기 쉬워 **단순한 형태**만 쓰세요.

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ssh -i ~/.ssh/<노트9_접속_키> -p 8022 -o IdentitiesOnly=yes -o BatchMode=yes <Termux_사용자>@127.0.0.1 proot-distro login ubuntu -- whoami"
```

`root`가 출력되면 PRoot 접속 성공입니다.

> **실패했던 방식(쓰지 마세요):** `proot-distro login ubuntu -- bash -lc "..."` 안에 파이프(`|`)와 따옴표를 겹쳐 넣으면 PowerShell이 `head` 등을 Windows 명령으로 해석해 오류가 납니다. 단순 명령으로 나눠 실행하세요.

---

## 6. 원격 접속 ② — USB 없이: SSH 역방향 터널

2026-11-06에 노트9가 **중계 서버로 먼저** SSH를 열어, 중계 서버의 `127.0.0.1:18023`에서 노트9 Termux의 8022로 들어오는 터널을 만들었습니다. USB를 뽑은 뒤에도 Termux 로그인과 Hermes 버전 확인에 성공했습니다.

**전제:** 중계 서버 전원 ON, 노트9 네트워크 연결, 터널 프로세스 유지. **터널이 끊긴 뒤 자동 재연결은 구축하지 않았습니다.** `nohup ssh -f -N`은 백그라운드 실행이지 영구 서비스가 아닙니다. 새 터널을 중복으로 만들지 마세요.

### 6.1 노트9 → 중계 서버 키 인증 확인
**실행 기기: 메인 PC / Windows PowerShell**

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ssh -i ~/.ssh/<노트9_접속_키> -p 8022 -o IdentitiesOnly=yes -o BatchMode=yes <Termux_사용자>@127.0.0.1 ssh -i /data/data/com.termux/files/home/.ssh/<노트9→서버_키> -o IdentitiesOnly=yes -o BatchMode=yes -o ConnectTimeout=10 <서버_계정>@<서버_주소> whoami"
```

중계 서버의 계정 이름이 출력되면 성공입니다.

> **주의(실제 실패 사례):** 노트9의 개인키를 `~/.ssh/...`처럼 `~`로 적으면 겹친 SSH에서 `~`가 **중계 서버의 홈**으로 풀려 `Permission denied (publickey)`가 납니다. **노트9의 키는 절대 경로**(`/data/data/com.termux/files/home/.ssh/...`)로 적으세요. 키 존재 확인: 노트9에서 `test -f <키_절대경로> && echo KEY_EXISTS`.

### 6.2 임시 터널(300초)로 시험
**실행 기기: 메인 PC / Windows PowerShell (이 창은 터널이 유지되는 동안 열어 둠)**

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ssh -i ~/.ssh/<노트9_접속_키> -p 8022 -o IdentitiesOnly=yes -o BatchMode=yes <Termux_사용자>@127.0.0.1 ssh -i /data/data/com.termux/files/home/.ssh/<노트9→서버_키> -o IdentitiesOnly=yes -o BatchMode=yes -o ExitOnForwardFailure=yes -o ConnectTimeout=10 -R 127.0.0.1:18022:127.0.0.1:8022 <서버_계정>@<서버_주소> sleep 300"
```

**두 번째 PowerShell 창**에서 포트가 열렸는지 확인합니다.

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ss -ltn '( sport = :18022 )'"
```

`127.0.0.1:18022 LISTEN`이면 성공입니다. 300초가 지나면 포트가 닫히므로 **확인 시점에 `Connection refused`가 나도 정상일 수 있습니다.**

### 6.3 호스트키 오류: `HostKeyAlias`
처음 18022로 접속하면 `No ED25519 host key is known for [127.0.0.1]:18022` / `Host key verification failed`가 나옵니다. **검증을 끄지 말고**, 이미 신뢰한 노트9의 8022 호스트키를 별칭으로 지정합니다.

**실행 기기: 중계 서버 / Ubuntu** (또는 PC에서 위와 같이 감싸서)

```bash
ssh -i ~/.ssh/<노트9_접속_키> -p 18022 -o HostKeyAlias='[127.0.0.1]:8022' -o IdentitiesOnly=yes -o BatchMode=yes -o ConnectTimeout=8 -o StrictHostKeyChecking=yes <Termux_사용자>@127.0.0.1 whoami
```

`HostKeyAlias`는 **이미 확인한 같은 노트9의 SSH 서버**라는 전제에서만 씁니다. 다른 기기에 함부로 재사용하지 마세요.

### 6.4 독립 백그라운드 터널(18023) 만들기
**실행 기기: 메인 PC / Windows PowerShell** (USB 연결 상태에서 시작. 노트9 안에서 새 백그라운드 SSH 프로세스가 시작됩니다.)

먼저 포트가 비어 있는지 확인합니다.

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ss -ltn '( sport = :18023 )'"
```

비어 있다면(출력 없음) 터널을 시작합니다.

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ssh -i ~/.ssh/<노트9_접속_키> -p 8022 -o IdentitiesOnly=yes -o BatchMode=yes <Termux_사용자>@127.0.0.1 nohup ssh -f -N -i /data/data/com.termux/files/home/.ssh/<노트9→서버_키> -o IdentitiesOnly=yes -o BatchMode=yes -o ExitOnForwardFailure=yes -o ConnectTimeout=10 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -R 127.0.0.1:18023:127.0.0.1:8022 <서버_계정>@<서버_주소>"
```

위 포트 확인 명령을 다시 실행해 `127.0.0.1:18023 LISTEN`이 보이면 성공입니다.

### 6.5 USB 없이 접속하기 (터널이 살아 있을 때만)
**실행 기기: 메인 PC / Windows PowerShell**

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ssh -i ~/.ssh/<노트9_접속_키> -p 18023 -o HostKeyAlias='[127.0.0.1]:8022' -o IdentitiesOnly=yes -o BatchMode=yes -o ConnectTimeout=8 -o StrictHostKeyChecking=yes <Termux_사용자>@127.0.0.1 whoami"
```

Termux 사용자 이름이 나오면 정상입니다. USB 케이블을 **분리한 뒤에도** 같은 명령이 성공했습니다. PRoot 안의 Hermes까지 확인하려면:

```powershell
ssh -i "$env:USERPROFILE\.ssh\<서버접속_키>" <서버_계정>@<서버_주소> "ssh -i ~/.ssh/<노트9_접속_키> -p 18023 -o HostKeyAlias='[127.0.0.1]:8022' -o IdentitiesOnly=yes -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=yes <Termux_사용자>@127.0.0.1 proot-distro login ubuntu -- /root/.local/bin/hermes --version"
```

`Connection refused`가 나오면 중계 서버에서 `ss -ltn '( sport = :18023 )'`로 포트를 확인하세요. 포트가 닫혔다면 백그라운드 SSH가 종료된 것입니다. **USB 없이는 중계 서버 쪽 명령만으로 노트9의 프로세스를 새로 시작할 수 없습니다.** 노트9 Termux에서 직접 다시 시작하거나 USB/ADB 경로를 복구한 뒤 진행합니다.

### 6.6 Wi-Fi 직접 접속은 동작하지 않았습니다
노트9의 Wi-Fi 주소(`<노트9_Wi-Fi_주소>:8022`)로 중계 서버가 직접 접속하면 시간 초과가 났습니다. 중계 서버와 노트9가 **서로 직접 닿는 같은 네트워크가 아니었기 때문**입니다. 이 구성에서는 **노트9가 먼저 연결을 여는 역방향 터널**을 쓰세요.

---

## 7. 서비스 구조 — Gateway와 자동 시작

### 7.1 구조

```text
노트9 Android 10
└─ Termux
   ├─ SSH 서버 (8022)
   └─ runsvdir  (서비스 관리자, 서비스 폴더: $PREFIX/var/service)
      ├─ sshd 서비스
      └─ hermes-gateway 서비스
         ├─ run  → ~/start-hermes-gateway.sh (심볼릭 링크)
         └─ supervise/
            └ proot-distro login ubuntu
               └ Python 가상환경 → Hermes `gateway run -vv`
```

Gateway 시작 스크립트(경로 속 설치 ID는 가림 처리):

```sh
#!/data/data/com.termux/files/usr/bin/sh
exec /data/data/com.termux/files/usr/bin/proot-distro login ubuntu -- /root/.hermes/installs/<설치_ID>/environments/<환경_ID>/venv/bin/python /root/hermes-agent/hermes gateway run -vv
```

`exec`는 스크립트 셸을 실행 대상으로 **대체**합니다. `-vv`는 상세 로그입니다. 이 구조가 실제 장애 복구를 보장하는지는 별도 검증이 필요합니다.

### 7.2 상태 점검 명령 (모두 읽기 전용, 노트9 Termux에서)
**실행 기기: 노트9 / Termux 셸**

```bash
ps -ef | grep '[r]unsvdir'
```
```bash
sv status "$PREFIX/var/service/hermes-gateway"
```
```bash
ls -la "$PREFIX/var/service/hermes-gateway"
```
```bash
cat ~/start-hermes-gateway.sh
```

> 노트9에서 `sv: command not found`가 나오면 SSH 환경의 PATH 문제일 수 있습니다. 실행 파일은 **전체 경로**로 지정하세요(`/data/data/com.termux/files/usr/bin/sv status <서비스_폴더>`). 서비스 상태가 `run: … (pid N) …s`이면 실행 중입니다.

`normally down`은 서비스 폴더에 `down` 파일이 있어 **기본 자동 시작이 억제되는 설정**일 수 있다는 뜻입니다. **현재 실행 중**이라는 사실과 **부팅 후 자동 실행**은 별개입니다. 이유 없이 `down` 파일을 지우거나 서비스를 재시작하지 마세요.

### 7.3 SSH 서비스 자동 시작 복구 사례 (2026-11-08)
Termux `sshd` 서비스 폴더에 `down` 파일이 있어 runit이 관리하는 SSH 서비스가 내려가 있었고, 이전에 수동으로 띄운 별도 SSH 프로세스만 실행 중이었습니다. 당시 수행한 조치:

1. 원본 `down` 파일을 홈 폴더에 백업 복사(`sshd_down_backup_<날짜>`)
2. 서비스 폴더의 `down`을 `down.disabled_<날짜>`로 **이름 변경**(삭제하지 않음)
3. 수동으로 띄운 SSH 프로세스에 종료 신호 전달
4. `sv up <sshd 서비스 폴더>`로 서비스 시작
5. 8022 포트 LISTEN(IPv4/IPv6)과 SSH 키 접속 확인

**주의:** 이는 당시 환경의 기록입니다. 직접 따라 하기 전에 **현재 상태를 읽기 전용으로 확인**하고, 변경 전에 원본을 백업하세요.

### 7.4 Termux:Boot 시작 스크립트
`~/.termux/boot/10-start-services` 내용:

```sh
#!/data/data/com.termux/files/usr/bin/sh
export PREFIX=/data/data/com.termux/files/usr
export SVDIR="$PREFIX/var/service"
export LOGDIR="$PREFIX/var/log"
/data/data/com.termux/files/usr/bin/service-daemon start
```

Termux:Boot 앱은 **설치한 뒤 한 번 직접 실행**해야 부팅 스크립트를 쓸 수 있습니다(공식 문서의 초기 조건).

### 7.5 재부팅 시험 결과 (2026-11-08)

**실행 기기: 중계 서버 / Ubuntu**

```bash
adb reboot
```
```bash
adb shell getprop sys.boot_completed
```

`1`이 나오면 Android 부팅이 끝난 것입니다. 이어서 서비스 상태를 확인했습니다.

```bash
adb shell run-as com.termux /data/data/com.termux/files/usr/bin/sv status /data/data/com.termux/files/usr/var/service/sshd
```
```bash
adb shell run-as com.termux /data/data/com.termux/files/usr/bin/sv status /data/data/com.termux/files/usr/var/service/hermes-gateway
```

결과: 두 서비스 모두 `run`. 단, **재부팅 직후 중계 서버의 `adb forward`는 사라져** SSH가 `Connection refused`였고, `adb forward tcp:8022 tcp:8022`를 다시 실행하자 접속됐습니다. 이어서 USB를 분리한 상태에서 Telegram 응답을 확인했습니다.

> 이는 **1회 시험**입니다. 장시간 무중단, 배터리 방전 후 복구, 강제 종료 후 복구는 시험하지 않았습니다.

---

## 8. Telegram 동작 확인

- 노트9의 Telegram 봇에 다른 스마트폰에서 메시지를 보내 **실제 응답**을 확인하세요(`<봇_이름>`).
- `sv status`의 `run`은 **프로세스 실행**의 증거이지 Telegram 송수신의 증거가 아닙니다.
- 중계 서버의 다른 Hermes/Telegram 봇과 노트9 봇은 **서로 독립**입니다. 서로의 설정을 건드리지 마세요.
- 2026-11-06에는 Gateway가 `adapter unavailable`, `heartbeat stale` 경고 후 `stopped`로 확인됐고, 11-07 이후 응답 시험은 성공했습니다. 서로 다른 날짜의 관찰이므로 섞어 쓰지 마세요.

---

## 9. 문제 해결 표

| 증상 | 의미 | 우선 확인·대응 |
| --- | --- | --- |
| `adb devices`에 `unauthorized` | 노트9가 USB 디버깅을 아직 허용하지 않음 | 노트9 화면 잠금 해제 후 허용 → `device` 확인 |
| `adb devices`가 비어 있음 | 케이블·USB 연결·USB 디버깅 문제 | 케이블, 노트9 화면, 연결 상태 |
| `adb forward --list`가 비어 있음 | 포트 전달 없음(USB 재연결·재부팅 후 흔함) | `adb forward tcp:8022 tcp:8022` 다시 실행 |
| `Connection refused` (8022) | 포트 전달이 없거나 Termux SSH 서버가 내려가 있음 | 포트 전달 확인 → 서비스 상태 확인 |
| `Connection refused` (18022/18023) | 터널 종료(18022는 300초 제한) | 중계 서버에서 `ss -ltn`으로 포트 확인 |
| `Permission denied (publickey)` | 사용자 이름·키 경로 오류, 겹친 SSH에서 `~`가 다른 홈으로 풀림 | 노트9 키는 **절대 경로**, 키 재발급 금지 |
| `Host key verification failed` | 터널 포트가 다른 주소로 인식됨 | `HostKeyAlias`(검증된 8022 호스트키), 검증 끄기 금지 |
| `Connection timed out` | 경로 자체가 안 닿음 | USB·네트워크·Wi-Fi 경로 확인(6.6 참고) |
| `sshd: no hostkeys available` | 일반 사용자는 `sshd -T` 진단 불가 | 호스트키 재생성 금지, 실제 접속으로 확인 |
| `sudo: interactive authentication is required` | 비대화식 sudo 불가 | 강제 우회 금지 |
| PowerShell이 `head` 등을 모른다고 함 | Linux 명령이 Windows에서 해석됨 | 중첩 따옴표 없는 단순 명령 사용 |
| `LastWriteTimeGet-ChildItem`처럼 붙어 보임 | 명령 두 개가 한 줄로 붙여넣어짐 | **한 줄씩** 복사 |
| `sv: command not found` (SSH) | 비대화식 셸 PATH 차이 | 전체 경로로 실행 |
| Gateway `heartbeat stale` | 서비스 이상 신호(11-06 관찰) | 현재 로그·상태를 읽기 전용으로 다시 확인 |
| `Gateway: stopped` (`hermes status`) | Gateway 미실행 | **재설치하지 말고** 상태·서비스 점검부터 |

**연결이 끊겼을 때 순서:** ① 중계 서버 전원과 USB 케이블 → ② PC에서 중계 서버 SSH → ③ `adb devices -l` → ④ `unauthorized`면 노트9에서 허용 → ⑤ `adb forward --list` 후 필요하면 다시 설정 → ⑥ Termux SSH → ⑦ PRoot 진입 → ⑧ `hermes --version` → ⑨ `hermes status`로 Gateway 상태. **성공한 단계는 다시 하지 말고** 실패한 단계만 진단하세요.

---

## 10. 백업과 복원 준비

### 10.1 2026-11-08 전체 백업
- **무엇을:** Android 전체 이미지가 아니라 **Termux 앱의 `home` + `usr` 데이터**를 `tar.gz`로 압축했습니다. `usr` 아래에 PRoot Ubuntu의 파일시스템이 있어 Hermes 설정과 실행 파일도 포함됩니다.
- **크기:** 약 6.2GB, 항목 약 29만 개
- **세 곳 복사:** 노트9 원본, 중계 서버, 메인 PC. **세 곳의 SHA-256이 동일**했고 `gzip -t`, `tar -tzf`가 통과했습니다.
- 압축 목록에서 부팅 스크립트, Gateway 시작 스크립트, `usr/var/service/hermes-gateway/run`, PRoot의 설정·인증 파일 경로를 확인했습니다. **파일이 있다는 것과 그 내용이 정상이라는 것은 다릅니다.**

### 10.2 포함되지 않는 것
- Android OS, Termux·Termux:Boot 앱(APK), 앱 권한, 배터리 최적화 예외, 시스템 설정
- 백업 파일 자신, PRoot의 가상 마운트 경로(`dev`, `proc`, `sys`, `storage`, `sdcard`, `mnt`, `apex`, `odm`, `system`, `vendor` 등), 실행 중 소켓, 서비스 감시 폴더, 임시 경로
- 백업 중 `file changed as we read it` 경고가 있었습니다. **실행 중인 시스템을 그대로 압축했으므로 파일 간 시점 일관성은 보장되지 않습니다.**

### 10.3 헷갈리기 쉬운 파일
- 188바이트짜리 `…1차.tar.gz`는 **실패한 전송의 잔재**입니다. 파일명에 `_정상전송`이 붙은 약 6.2GB 파일이 정상본입니다. 임의로 지우지 마세요.
- 이전의 HOME 전용 백업(11-05)은 범위가 다릅니다.
- 중계 서버 자체 Hermes 백업은 노트9 백업과 무관합니다.

### 10.4 무결성 확인 (읽기 전용)
**실행 기기: 중계 서버 / Ubuntu**

```bash
sha256sum <백업_폴더>/Note9_Hermes_전체백업_<날짜>_정상전송.tar.gz
```

**실행 기기: 메인 PC / Windows PowerShell**

```powershell
(Get-FileHash -LiteralPath "<백업_폴더>\Note9_Hermes_전체백업_<날짜>_정상전송.tar.gz" -Algorithm SHA256).Hash
```

두 값이 `<64자리_해시값>`(기록해 둔 값)과 같아야 합니다. **서로 다른 기기에서 각각 실행**하는 명령입니다. 다르면 복원에 쓰지 말고 원인부터 조사하세요.

### 10.5 복원 — 준비 점검 목록 (실제 복원은 하지 않았음)
**이 목록은 복원 지시가 아닙니다.** 동작 중인 노트9 위에 `tar -x`로 덮어쓰지 마세요.

1. 복원 대상: 현재 노트9가 아닌 **별도 시험 환경**. 기기 모델·CPU 아키텍처·Android·Termux 버전·저장 공간 확인
2. 백업 원본 보호: 검증된 복사본을 따로 보관하고 SHA-256·압축 테스트를 먼저 확인
3. 사전 설치: 호환되는 Termux·Termux:Boot, 권한, PRoot 동작(앱 자체는 이번 백업에 없음)
4. 압축 내부 검토: 경로·심볼릭 링크·권한·소유권. 잘못된 추출 경로는 파일을 덮어쓸 수 있음
5. **격리된 임시 폴더**에 먼저 풀어 내용과 경로 검증
6. 실행 환경 재구성: PRoot 경로, 앱 권한, 배터리 정책, SSH 키 접근 권한, Termux:Boot, runit
7. 인증 검증: OAuth 토큰은 기기에 종속되거나 만료될 수 있어 **복사만으로 인증 성공을 보장하지 않음.** 종량제 키로 자동 전환 금지
8. 중복 실행 방지: 같은 Telegram 봇의 Gateway가 **두 곳에서 동시에** 돌지 않게 계획
9. 성공 판정: `hermes --version`만으로 부족. Telegram 응답, 바코·지코 실제 호출, 재부팅 자동 시작, 장시간 동작, 네트워크 전환, 오류 복구까지 별도 시험

**결론:** 3곳의 SHA-256 일치는 **파일 복제 성공**을 증명합니다. **복원 성공이나 완전한 시스템 이미지**를 뜻하지 않습니다.

---

## 11. 운영 환경 — 전원·네트워크·성능

### 11.1 네트워크와 이동
- 노트9에 **유심이 없어도** Wi-Fi가 연결되면 Hermes는 인터넷에 접근합니다.
- 집 Wi-Fi를 벗어나면 외부 모델 호출과 Telegram 응답이 **불가능**해집니다(프로세스는 남아 있어도).
- 돌아오면 Android가 저장된 Wi-Fi에 자동 재연결할 수 있지만, **Gateway가 자동으로 다시 연결되는지는 시험하지 않았습니다.**
- 처음 접속하는 공용 Wi-Fi는 암호나 웹 로그인이 필요할 수 있고, 보안·품질을 장담할 수 없습니다.
- 다른 스마트폰의 모바일 핫스팟에 연결하는 방법도 있지만 자동 재연결 조건은 기기 설정에 따라 다릅니다.
- **상시 서버 용도에는 "집 Wi-Fi + 안정적인 전원"이 가장 적합합니다.**

### 11.2 전원과 화면
- 기록된 운영 방침은 USB·충전기를 평소에는 분리하고 필요할 때만 연결하는 것이었습니다. **배터리만으로 운영하면 방전 시 서비스가 멈춥니다.** 잔량을 주기적으로 확인하세요.
- 화면을 꺼도 Telegram 응답은 가능했습니다(장시간은 미검증). **화면 OFF는 서버 OFF가 아닙니다.**
- 전원이 연결돼 있다는 것과 Hermes가 실행 중이라는 것은 별개입니다.
- 충전 중 화면이 계속 켜지는 설정이면 화면 번인과 발열에 주의하세요. 24시간 충전은 배터리 노후를 앞당길 수 있습니다.
- Termux와 Termux:Boot를 Android **배터리 최적화 예외**로 등록해 두면 백그라운드 종료를 줄일 수 있습니다.

### 11.3 성능 기준값 (2026-11-08, 대기 중 측정)

| 항목 | 측정 |
| --- | --- |
| 저장공간(`/data`) | 112GB 중 약 55GB 여유 |
| RAM | 총 5.4GB (Android 기준 상태 `normal`) |
| CPU | 전체 약 4~7% 사용 |
| 온도 | CPU·배터리 약 25°C, 열 제한 없음 |
| Hermes Gateway | 같은 프로세스가 약 8시간 36분 연속 실행 |
| Hermes 프로세스 메모리 | 약 300MB 안팎 |

- 시스템 부하 평균(load average)이 13대로 높게 보였지만, **I/O 대기 상태의 커널 스레드가 포함된 값**으로 보이며 CPU 과부하라고 단정할 수 없습니다. 커널 스레드를 강제 종료하지 마세요.
- 위 값은 **대기 중** 측정이며 실제 대화 중 부하나 장시간 발열은 측정하지 않았습니다.

---

## 12. 안전 운영과 개인정보 보호

- `auth.json`, `.env`, OAuth 클라이언트 비밀 파일, SSH 개인키, Telegram 봇 토큰, API 키는 **공개 저장소·홈페이지·채팅에 올리지 않습니다.**
- 백업 파일은 암호화되지 않은 `tar.gz`이며 **인증 정보가 들어 있을 수 있습니다.** 공유·공개 업로드를 하지 마세요. 파일 권한은 소유자만 읽고 쓰도록 제한하는 것이 안전합니다(권한 변경은 영향을 확인한 뒤).
- 공개 문서에는 **서버 주소·계정 이름·키 파일 이름·기기 일련번호·봇 이름·사용자 ID·PC 사용자 경로**를 싣지 마세요. 이 매뉴얼은 모두 `<…>`로 바꿨습니다.
- **작성자 권장(검증된 기록은 아님):** 노트9의 SSH 서버가 Wi-Fi 전체에 열려 있고 비밀번호 로그인을 허용한다면, 관리 경로를 USB 전달/역방향 터널로만 쓰는 구성에서는 **로컬 전용·키 로그인 전용으로 제한**하는 것을 검토하세요. 서버와 노트9가 서로의 키를 가지고 있다면 한쪽 기기를 잃어버렸을 때를 대비해 키의 권한 범위를 줄이는 방법도 있습니다. 변경 전에 반드시 접속 수단을 하나 더 확보하세요.
- 별도 과금 방지: 모델·인증 공급자를 바꾸기 전에 승인과 비용 확인을 받으세요.

---

## 13. 아직 확인하지 않은 것

- 수 시간~수일 장시간 무중단 운영, 배터리 소모·발열, 충전 중 장시간 안정성
- Wi-Fi ↔ 모바일 핫스팟 전환 후 자동 복구, 배터리 완전 방전·강제 종료 후 복구
- 터널 끊김 후 자동 재연결(autossh/서비스) 설계
- 바코·지코의 **현재** 실제 호출 경로와 추가 과금 여부
- 백업의 **실제 복원**
- 재부팅 후 자동 시작의 반복 검증

### 다음 작업 순서
1. 현재 Termux:Boot·runit·Gateway 상태를 **읽기 전용**으로 다시 점검해 위 기록과 차이 확인
2. 변경 전 설정·인증 파일 백업과 승인된 범위의 변경
3. 재부팅·자동 시작·자동 복구 반복 시험
4. 네트워크 전환·화면 OFF 장시간·충전 시험(각각 독립적으로)
5. 별도 시험 환경에서 복원 시험

---

## 부록 A. 용어 풀이

| 용어 | 쉬운 설명 |
| --- | --- |
| Termux | 안드로이드 앱 안에서 리눅스 명령을 쓰게 해 주는 터미널 환경 |
| PRoot / proot-distro | Termux 안에 Ubuntu 같은 리눅스 환경을 "껍데기"로 올려 주는 도구. 별도의 가상머신이 아니라 안드로이드의 커널을 함께 씀 |
| ADB | USB로 안드로이드 기기를 제어하는 도구 |
| `adb forward` | 컴퓨터의 어떤 포트로 들어온 연결을 USB로 연결된 기기의 포트로 넘겨 주는 설정 |
| SSH 역방향 터널(`-R`) | **노트9가 먼저** 서버로 연결을 열어 두고, 서버가 그 길을 거슬러 노트9로 들어오게 하는 방식 |
| runit / `sv` | Termux 안에서 서비스를 시작·감시하는 도구와 그 명령 |
| `down` 파일 | 서비스 폴더에 있으면 "기본 자동 시작 안 함"을 뜻하는 표시 파일 |
| Gateway | Hermes가 Telegram 같은 메신저와 대화하도록 연결해 주는 프로세스 |
| SHA-256 | 파일 내용으로 계산한 지문. 같은 파일이면 같은 값이 나옴 |

## 부록 B. 명령 요약 (실행 기기별)

| 실행 기기 | 목적 | 명령 |
| --- | --- | --- |
| 중계 서버 | USB 인식 | `adb devices -l` |
| 중계 서버 | 포트 전달 확인 | `adb forward --list` |
| 중계 서버 | 포트 전달 만들기 | `adb forward tcp:8022 tcp:8022` |
| 중계 서버 | 터널 포트 확인 | `ss -ltn '( sport = :18023 )'` |
| 노트9 Termux | PRoot 진입 | `proot-distro login ubuntu` |
| 노트9 PRoot | 버전 확인 | `/root/.local/bin/hermes --version` |
| 노트9 Termux | 서비스 관리자 확인 | `ps -ef \| grep '[r]unsvdir'` |
| 노트9 Termux | Gateway 상태 | `sv status "$PREFIX/var/service/hermes-gateway"` |
| 노트9 Termux | 시작 스크립트 확인 | `cat ~/start-hermes-gateway.sh` |
| 중계 서버 | 재부팅 완료 확인 | `adb shell getprop sys.boot_completed` |

## 부록 C. `<…>` 자리표시자 설명

| 표시 | 넣어야 할 값 |
| --- | --- |
| `<서버_주소>` | 중계 서버의 IP 주소 또는 도메인 |
| `<서버_계정>` | 중계 서버의 사용자 이름 |
| `<서버_호스트>` | 중계 서버의 호스트 이름(프롬프트에 보이는 이름) |
| `<서버접속_키>` | 메인 PC가 중계 서버에 접속할 때 쓰는 SSH 개인키 파일 이름 |
| `<노트9_접속_키>` | 중계 서버가 노트9 Termux에 접속할 때 쓰는 키 파일 이름 |
| `<노트9→서버_키>` | 노트9가 중계 서버로 역방향 터널을 열 때 쓰는 키 파일 이름 |
| `<Termux_사용자>` | Termux 사용자 이름(`whoami`로 확인) |
| `<기기_일련번호>` | `adb devices`에 표시되는 기기 일련번호 |
| `<노트9_Wi-Fi_주소>` | 노트9가 Wi-Fi에서 받은 사설 IP |
| `<봇_이름>` | 노트9용 Telegram 봇 이름 |
| `<설치_ID>`, `<환경_ID>` | Hermes가 만든 가상환경 경로 속 무작위 ID |
| `<백업_폴더>`, `<날짜>` | 백업 파일을 보관한 폴더와 날짜 |
| `<사용자>` | Windows 사용자 이름 |
| `<64자리_해시값>` | 백업 생성 직후 기록해 둔 SHA-256 값 |

---

*이 매뉴얼의 "실제로 확인된 것"은 2026-11-05~11-08에 수행한 작업의 출력 기록입니다. 확인되지 않은 항목은 13장에 모았습니다.*
