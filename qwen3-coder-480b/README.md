# Qwen3-Coder-480B 추론 서버 구축 (Task 1)

## 환경

- GPU: NVIDIA H100 80GB HBM3 × 8장 (총 640GB VRAM), Docker(sudo 권한으로 실행), CUDA 12.8
- RAM 1.7TB, `/data` 공유 스토리지 77TB 여유
- 공유 서버 — 다른 사용자(vision, voice)가 동시에 GPU를 사용하는 경우가 있어 GPU 점유 범위를 그때그때 확인하며 진행

## 1단계 — 소형 모델로 파이프라인 먼저 검증

바로 480B 모델을 받기 전에, `Qwen2.5-Coder-32B-Instruct`로 docker-compose → vLLM → OpenAI 호환 API
전체 흐름을 먼저 검증했다 ([`../qwen2.5-coder-32b/docker-compose.yml`](../qwen2.5-coder-32b/docker-compose.yml),
GPU 2장 tensor-parallel, 포트 8123).
`/v1/models`, `/v1/chat/completions` 정상 응답 확인 후 본 작업으로 진행.

## 2단계 — 실제 모델 크기 확인

HuggingFace API로 실측한 결과:

| 버전 | 총 크기 | 640GB(8×H100)에 적합? |
|---|---|---|
| `Qwen/Qwen3-Coder-480B-A35B-Instruct` (BF16) | 960.3GB | ✗ 불가능 |
| `Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8` (공식 FP8) | 482.2GB | ✓ 여유 있음 |

→ 공식 FP8 양자화 버전 채택.

## 3단계 — 배포 설정 ([`./docker-compose.yml`](./docker-compose.yml))

vLLM(`vllm/vllm-openai:latest`) 이미지, HF 캐시는 `/data/models/huggingface`에 마운트해 공유.

### 문제 1: `tensor-parallel-size: 8` 실패

```
ValueError: The output_size of gate's and up's weight = 320 is not divisible by
weight quantization block_n = 128.
```

FP8 블록 단위 양자화(block_n=128)와 MoE의 gate/up projection을 8-way로 나눴을 때
샤드 크기(320)가 128로 나누어떨어지지 않아 발생. 이 모델의 MoE intermediate size 기준으로
유효한 tensor-parallel 크기는 1, 2, 4, 5, 10, 20 등으로 제한됨 — 8은 불가능.

**해결**: `--tensor-parallel-size 4` + `--pipeline-parallel-size 2` 조합으로 8개 GPU를 모두 사용.
(TP 차원은 128로 나누어떨어지는 값 유지, 나머지는 PP로 확장)

### 문제 2: 크래시 후 좀비 프로세스가 GPU0 점유

`restart: unless-stopped` 정책 때문에 컨테이너가 크래시 후 재시작을 반복하다가,
`docker compose down`으로 컨테이너를 지운 뒤에도 `VLLM::EngineCore` 프로세스(약 70GB)가
GPU0에 남아있는 것을 확인 (`nvidia-smi --query-compute-apps`로 PID 특정).
컨테이너 소속이 아닌 고아 프로세스였음 — `sudo kill -9 <pid>`로 정리 후 GPU 완전히 회수됨.

## 4단계 — 검증 결과

수정된 설정(TP=4 × PP=2)으로 재기동 후:
- 가중치 로딩: 49개 safetensors shard, 약 777초(13분) 소요
- `curl http://localhost:8000/v1/models` → `qwen3-coder-480b` 정상 등록
- `curl http://localhost:8000/v1/chat/completions` → 피보나치 메모이제이션 함수 코드 정상 생성

## 5단계 — opencode 연동을 위한 tool-calling 활성화

opencode(에이전틱 CLI)를 연결해 테스트하는 과정에서 `--enable-auto-tool-choice`,
`--tool-call-parser qwen3_coder` 플래그가 없으면 tool_choice=auto 요청이 거부되는 것을 발견,
compose 파일에 추가. 자세한 내용은 [../opencode/README.md](../opencode/README.md) 참고.

opencode 연동은 [`opencode.json.example`](./opencode.json.example)의 `local-vllm` provider
블록을 `~/.config/opencode/opencode.json`에 병합하면 된다.

## 현재 상태

- 엔드포인트: `http://localhost:8000/v1` (opencode 테스트 이후 **GPU 반납을 위해 컨테이너는 현재 종료됨**)
- 서빙 모델명: `qwen3-coder-480b`
- 재기동: `sudo docker compose -f docker-compose.yml up -d` (이 폴더 안에서 실행)
- 재기동 시 반드시 아래 두 가지를 유지할 것:
  - TP=4/PP=2 설정 (TP=8로 되돌리면 FP8 블록 양자화 에러로 다시 실패함)
  - `--enable-auto-tool-choice` / `--tool-call-parser qwen3_coder` (없으면 opencode 등 tool-calling 클라이언트 연결 시 에러)
