# opencode 설치 및 연동 (Task 3)

## 목표

Task 1에서 기동한 로컬 vLLM 서버(Qwen3-Coder-480B-FP8, `http://localhost:8000/v1`)에
실제 코딩 에이전트 프론트엔드를 연결해 사용할 수 있게 하는 것. 프론트엔드로 opencode
(오픈소스 CLI 코딩 에이전트, vLLM과 엔드투엔드 결합 가능)를 선택했다.

## 설치 방법 조사

[`./INSTALL.md`](./INSTALL.md)에 curl 스크립트/npm/Homebrew/배포판 패키지 매니저/
mise/Windows 전용/데스크톱 앱 등 7가지 방법을 정리하고 이 서버(Linux, apt 기반) 환경에서의
적합도를 비교했다. Node.js/npm, Homebrew 등이 이미 설치돼 있는지 불확실했기 때문에,
별도 의존성 없이 바로 되는 **curl 설치 스크립트**를 1순위로 추천했다.

## 설치 실행

```bash
curl -fsSL https://opencode.ai/install | bash
```

- 설치된 버전: **1.18.5**
- 설치 경로: `~/.opencode/bin/opencode`
- PATH: `~/.bashrc`에 자동 추가됨 (새 셸부터 적용, 현재 셸에서는 `export PATH="$HOME/.opencode/bin:$PATH"`로 즉시 사용 가능)

설치 확인:
```bash
opencode --version
# 1.18.5
```

## 로컬 provider 설정

opencode는 기본적으로 Anthropic/OpenAI 등 클라우드 provider를 쓰도록 되어 있어,
로컬 vLLM 서버를 커스텀 OpenAI 호환 provider로 등록했다.

설정 파일: `~/.config/opencode/opencode.json` (전역 설정 — 어느 디렉터리에서 opencode를 실행해도 적용됨).
저장소에는 참고용으로 [`./opencode.json.example`](./opencode.json.example)로 동일 내용을 커밋해둠.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "local-vllm/qwen3-coder-480b",
  "provider": {
    "local-vllm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local vLLM (Qwen3-Coder-480B)",
      "options": { "baseURL": "http://127.0.0.1:8000/v1" },
      "models": {
        "qwen3-coder-480b": {
          "name": "Qwen3-Coder-480B-A35B-Instruct-FP8 (Local)",
          "limit": { "context": 131072, "output": 8192 }
        }
      }
    }
  }
}
```

최상위 `"model"` 필드는 `<provider ID>/<model ID>` 형식이어야 함 — provider ID는
`provider` 객체의 키(`local-vllm`)와 정확히 일치해야 한다. 한 번 `"local/qwen3-coder-480b"`처럼
provider 키와 다른 이름으로 잘못 들어가 있었던 적이 있는데, 이 경우 `-m` 없이 `opencode run`을
실행하면 기본 모델이 해석되지 않는다 — provider 키와 정확히 맞춰야 함.

설정 후 `opencode models`로 `local-vllm/qwen3-coder-480b`가 목록에 뜨는지, `-m` 옵션 없이
`opencode run "..."`을 실행했을 때 상단에 `> build · qwen3-coder-480b`로 표시되는지 확인.

## 문제 발생 — tool-calling 미지원 에러

`opencode run -m local-vllm/qwen3-coder-480b "..."` 실행 시:

```
Error: "auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set
```

opencode는 파일 읽기/쓰기 등에 tool call(`tool_choice: auto`)을 사용하는데, vLLM 서버 기동 시
이 기능이 꺼져 있어 발생. 컨테이너 안에서 `vllm serve --help=Frontend`로 지원되는
`--tool-call-parser` 목록을 확인해 Qwen3-Coder 전용 파서 `qwen3_coder`를 확인.

**해결**: [`../qwen3-coder-480b/docker-compose.yml`](../qwen3-coder-480b/docker-compose.yml)에
`--enable-auto-tool-choice`, `--tool-call-parser qwen3_coder` 추가 후 컨테이너 재기동
(가중치는 이미 로컬에 캐시돼 있어 재다운로드 없이 로딩만 다시 진행, 약 13분).

## 테스트 결과 — 성공

1. **일반 코드 생성**: "팔린드롬 체크 함수 작성" 요청 → 정상 코드 반환
2. **Tool-calling(에이전틱 파일 작업)**: "hello.py 파일을 실제로 생성해줘" 요청 →
   opencode가 `Write` 도구를 호출해 `hello.py`를 정확한 내용으로 실제 생성함을 확인
   (`opencode ↔ 로컬 vLLM ↔ tool-calling` 전체 파이프라인 검증 완료)

## 작업 후 GPU 반납

테스트 완료 후 `../qwen3-coder-480b` 폴더에서 `docker compose down`으로 컨테이너 종료,
`nvidia-smi`로 8개 GPU 전부 0MiB 사용/고아 프로세스 없음 확인 — 다른 사용자가 쓸 수 있도록 반납.

재사용 시: 같은 폴더에서 `sudo docker compose up -d`로 재기동 후,
"Application startup complete" 로그 확인 → `opencode run -m local-vllm/qwen3-coder-480b "..."`
로 바로 사용 가능 (설정은 이미 저장되어 있어 추가 작업 불필요).
