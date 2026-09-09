# OpenCode 설치 방법

opencode(오픈소스 CLI 코딩 에이전트, [github.com/anomalyco/opencode](https://github.com/anomalyco/opencode))를
이 서버(Linux x64)에 설치하는 방법들. 방법별 명령어와 이 환경에서의 적합성을 정리했다.

---

## 방법 1. curl 설치 스크립트 (권장 — 별도 의존성 없음)

```bash
curl -fsSL https://opencode.ai/install | bash
```

- 공식 "가장 빠른 범용 설치" 방법. OS/아키텍처(Linux x64/arm64, musl/Alpine, AVX2 여부)를 자동 감지해 맞는 바이너리를 받는다.
- 설치 경로: `$HOME/.opencode/bin/opencode` (기본값 고정, 환경변수로 변경 불가)
- 셸 설정파일(`~/.bashrc` 등)에 PATH를 자동 추가함 — 이게 싫으면 `--no-modify-path` 옵션 사용
- 특정 버전 지정: `curl -fsSL https://opencode.ai/install | bash -s -- -v 1.0.180` (또는 `VERSION=1.0.180` 환경변수)
- **이 환경 적합도**: Node.js/npm, Homebrew 등 아무것도 안 깔려 있어도 바로 되는 가장 가벼운 방법. 서버 환경에 가장 적합.

## 방법 2. npm (Node.js 패키지 매니저)

```bash
npm i -g opencode-ai@latest
```

- Windows 포함 모든 OS에서 동작하는 가장 이식성 좋은 방법(공식 문서 표현: "가장 포터블한 방법")
- 사전 조건: Node.js/npm 설치되어 있어야 함 (`node --version`, `npm --version`으로 확인 필요 — 이 서버에 설치 여부 미확인)
- **이 환경 적합도**: npm이 이미 있다면 간단하지만, 없으면 Node.js부터 깔아야 해서 방법 1보다 무거움.

## 방법 3. Homebrew

```bash
brew install anomalyco/tap/opencode
# 또는
brew install opencode
```

- 공식 tap(`anomalyco/tap`)이 최신 릴리스 반영에 더 빠름
- ripgrep 등 의존성 포함 다운로드 총 ~157MB
- 사전 조건: Homebrew 설치 필요 (Linuxbrew 형태로 리눅스에도 설치 가능하지만 이 서버엔 없을 가능성 높음)
- **이 환경 적합도**: 리눅스 서버에 Homebrew부터 새로 깔아야 한다면 번거로움 — 이미 brew를 쓰는 macOS/Linux 워크스테이션이면 편함.

## 방법 4. Linux 배포판 패키지 매니저

```bash
# Arch Linux / Manjaro (pacman)
sudo pacman -S opencode

# AUR (paru)
paru -S opencode-bin

# Nix
nix run nixpkgs#opencode
```

- **이 환경 적합도**: 이 서버는 Ubuntu 계열(apt 기반)로 보여 `pacman`은 해당 없음. `nix`가 이미 설치돼 있다면 재현성 좋은 옵션.

## 방법 5. mise (버전 매니저)

```bash
mise use -g opencode
```

- 여러 CLI 도구 버전을 한 번에 관리하고 싶을 때 유용
- 사전 조건: mise 설치 필요

## 방법 6. Windows 전용 (해당 없음 — 참고용)

```bash
scoop install opencode
choco install opencode
```

## 방법 7. 데스크톱 앱 (해당 없음 — 서버에는 GUI 없음, 참고용)

- macOS: `brew install --cask opencode-desktop` 또는 DMG
- Windows: `scoop bucket add extras; scoop install extras/opencode-desktop` 또는 EXE
- Linux: `.deb` / `.rpm` / `.AppImage` 다운로드

---

## 요약 추천

| 순위 | 방법 | 이유 |
|---|---|---|
| 1 | **curl 스크립트** | 이 서버에 이미 깔려있는 것에 의존하지 않음, 가장 빠름 |
| 2 | npm | Node.js/npm이 이미 있다면 이것도 간단 (사전 확인 필요) |
| 3 이하 | Homebrew/pacman/nix/mise | 해당 패키지 매니저가 이미 설치돼 있을 때만 고려 |

설치 후에는 `opencode auth login` 또는 config 파일에서 provider를 커스텀 OpenAI 호환 엔드포인트
(`http://localhost:8000/v1`, 모델명 `qwen3-coder-480b`)로 지정해야 로컬 vLLM 서버에 연결된다 —
이 부분은 설치 방법 결정 후 별도로 진행.

## 참고 출처
- [OpenCode GitHub](https://github.com/anomalyco/opencode)
- [OpenCode Quickstart - DEV Community](https://dev.to/rosgluk/opencode-quickstart-install-configure-and-use-the-terminal-ai-coding-agent-4kcb)
- [How to Install OpenCode in 2026 - NxCode](https://www.nxcode.io/resources/news/opencode-install-guide-step-by-step-2026)
- [Install OpenCode: macOS, Linux & Windows (2026) - aicodinghub.app](https://aicodinghub.app/opencode-install/)
