# opencode + H100 (Qwen3 Coder) 연결 설정법

윈도우 사용자는 opencode를 설치한 뒤 WezTerm에서 사용하면 됩니다.
ssh key를 생성한 후 public key를 서버의 ~/.ssh/authorized_keys에 등록해야 합니다.
해당 키를 생성하여 서버 관리자에게 전달하면 등록해 드립니다

## 1. SSH 터널 등록

`~/.ssh/config`에 아래 내용 추가 (본인 개인키 경로/계정 정보로 바꿔서):

```
Host h100
    HostName <서버_IP>          # 예: 203.0.113.10 — 서버 관리자에게 문의
    User <SSH_계정명>            # 서버 관리자에게 발급받은 계정명
    IdentityFile ~/.ssh/id_rsa_h100
    LocalForward 9911 localhost:9911
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

- `IdentityFile`은 본인 개인키 경로로 바꿔주세요. (개인키 파일 자체는 공유받으면 안 되고, 서버 관리자에게 본인 키를 따로 등록받아야 합니다)
- `LocalForward 9911 localhost:9911` 덕분에 `ssh h100` 접속만 하면 로컬 9911 포트가 서버 9911 포트로 자동 포워딩됩니다.

## 2. 터널 열기

```bash
ssh h100
```

- 접속된 상태로 터미널을 계속 켜둬야 합니다 (끊으면 터널도 끊김).
- 확인: 다른 터미널에서 아래 호출했을 때 응답 오면 정상.

```bash
curl http://localhost:9911/v1/models
```

## 3. opencode 설정

`~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "local-vllm/qwen3-coder-480b",
  "provider": {
    "local-vllm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local vLLM (Qwen3-Coder-480B)",
      "options": {
        "baseURL": "http://localhost:9911/v1"
      },
      "models": {
        "qwen3-coder-480b": {
          "name": "Qwen3-Coder-480B-A35B-Instruct-FP8 (Local)",
          "limit": {
            "context": 131072,
            "output": 8192
          }
        }
      }
    }
  }
}
```

> **주의**: `baseURL`은 반드시 `http://localhost:9911/v1`처럼 **localhost**로 설정하고, 1번의 SSH 터널(포트 포워딩)을 통해 접속해야 동작합니다. 서버 IP를 직접 넣는 방식은 동작하지 않았습니다.

## 4. 실행

```bash
cd <프로젝트 폴더>
opencode
```

- 터미널 안에서 채팅형 UI로 바로 사용 가능
- 브라우저로 쓰고 싶으면 `opencode web`

---

# opencode 설치 방법 (OS별)

opencode(오픈소스 CLI 코딩 에이전트, [github.com/anomalyco/opencode](https://github.com/anomalyco/opencode))를
설치하는 방법을 OS별로 정리했습니다. 설치 후에는 위 1~4단계(SSH 터널 + `opencode.json` 설정)를 그대로 따라하면 됩니다.

## macOS

**방법 1. curl 설치 스크립트 (권장)**
```bash
curl -fsSL https://opencode.ai/install | bash
```
- 설치 경로: `$HOME/.opencode/bin/opencode`
- 셸 설정파일(`~/.zshrc` 등)에 PATH가 자동 추가됩니다. 원치 않으면 `--no-modify-path` 옵션 사용.

**방법 2. Homebrew**
```bash
brew install anomalyco/tap/opencode
# 또는
brew install opencode
```

**방법 3. npm** (Node.js 설치되어 있는 경우)
```bash
npm i -g opencode-ai@latest
```

**데스크톱 앱**: `brew install --cask opencode-desktop` 또는 DMG 다운로드

## Windows

**방법 1. npm** (가장 이식성 좋은 방법, Node.js 필요)
```powershell
npm i -g opencode-ai@latest
```

**방법 2. Scoop**
```powershell
scoop install opencode
```

**방법 3. Chocolatey**
```powershell
choco install opencode
```

**데스크톱 앱**: `scoop bucket add extras; scoop install extras/opencode-desktop` 또는 EXE 설치 파일 다운로드

> Windows에서는 opencode 설치 후 **WezTerm** 터미널에서 실행하는 것을 권장합니다 (문서 상단 참고).

## Linux

**방법 1. curl 설치 스크립트 (권장 — 별도 의존성 없음)**
```bash
curl -fsSL https://opencode.ai/install | bash
```
- OS/아키텍처(x64/arm64, musl/Alpine, AVX2 여부)를 자동 감지해 맞는 바이너리를 받습니다.
- 설치 경로: `$HOME/.opencode/bin/opencode`
- 특정 버전 지정: `curl -fsSL https://opencode.ai/install | bash -s -- -v 1.0.180`

**방법 2. npm** (Node.js/npm 설치되어 있는 경우)
```bash
npm i -g opencode-ai@latest
```

**방법 3. Homebrew (Linuxbrew)**
```bash
brew install anomalyco/tap/opencode
```

**방법 4. 배포판별 패키지 매니저**
```bash
# Arch Linux / Manjaro
sudo pacman -S opencode

# AUR
paru -S opencode-bin

# Nix
nix run nixpkgs#opencode
```

**방법 5. 데스크톱 앱**: `.deb` / `.rpm` / `.AppImage` 다운로드

---

### 참고 출처
- [OpenCode GitHub](https://github.com/anomalyco/opencode)
- [OpenCode Quickstart - DEV Community](https://dev.to/rosgluk/opencode-quickstart-install-configure-and-use-the-terminal-ai-coding-agent-4kcb)
- [How to Install OpenCode in 2026 - NxCode](https://www.nxcode.io/resources/news/opencode-install-guide-step-by-step-2026)
- [Install OpenCode: macOS, Linux & Windows (2026) - aicodinghub.app](https://aicodinghub.app/opencode-install/)
