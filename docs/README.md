# docs — 작업 기록 인덱스 (최상위)

이 저장소는 자체 호스팅 코딩 에이전트 구축 작업(참고: [../HowToUseCodingAgent.md](../HowToUseCodingAgent.md),
[../TASK.md](../TASK.md))의 조사·구현·실행 기록이다. 파일은 모델/도구별 폴더로 나뉘어 있다.

## 폴더 구조

| 폴더 | 내용 | 상태 |
|---|---|---|
| [`qwen2.5-coder-32b/`](../qwen2.5-coder-32b/README.md) | 파이프라인 검증용 소형 모델 | 검증 완료, 이후 상태 미확인 |
| [`qwen3-coder-480b/`](../qwen3-coder-480b/README.md) | Task 1 본 대상 — 실제 8x H100 기동, opencode 연동까지 검증 완료 | **실행 완료** (테스트 후 GPU 반납, 종료 상태) |
| [`gpt-oss-120b/`](../gpt-oss-120b/README.md) | openai/gpt-oss-120b (MXFP4) — 이 저장소에서 유일하게 **H100 1장**으로 동작하는 구성 | **TP=1 기동 성공** (131072 컨텍스트 확보, KV 303,646토큰) → 사용자 지시로 실험 중단·GPU 반납. 실제 추론/opencode tool-calling은 미검증 (→ [gpt-oss-120b.md](./gpt-oss-120b.md)) |
| [`opencode/`](../opencode/README.md) | Task 3 — 코딩 에이전트 프론트엔드 설치/연동 | 설치·연동·테스트 완료 |

각 폴더에는 `docker-compose.yml`(또는 conda/실행 스크립트)과 `README.md`(해당 모델 작업 기록)가
함께 들어있다. 환경변수는 폴더별 `.env.example`을 `.env`로 복사해 조정한다 (`.env`는 gitignore됨).

## 현재 실행 중인 서비스

| 서비스 | 포트 | 모델 | 상태 |
|---|---|---|---|
| `vllm-qwen3-coder-480b` | 8000 | Qwen3-Coder-480B-A35B-Instruct-FP8 | opencode 연동 테스트 완료 후 **종료됨** (GPU 반납) |
| `vllm-qwen-coder` (32B 검증용) | 8123 | Qwen2.5-Coder-32B-Instruct | 상태 미확인 |
| `vllm-gpt-oss-120b` | 8500 | openai/gpt-oss-120b (MXFP4) | TP=1(GPU 1장) 기동 및 헬스체크 성공 후 **종료됨** (GPU 반납, 가중치 60.77GiB는 캐시에 남음) |

## 개별 조사 문서

| 문서 | 내용 |
|---|---|
| [`gpt-oss-120b.md`](./gpt-oss-120b.md) | gpt-oss-120b 사양·KV 캐시 계산·서빙 방식 비교·vLLM 플래그 결정 근거·알려진 이슈 |
