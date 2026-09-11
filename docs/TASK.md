# TASK.md — 자체 호스팅 코딩 에이전트 구축 작업 목록

> 참고: [HowToUseCodingAgent.md](./HowToUseCodingAgent.md) 조사 내용을 기반으로 작성. H100급 GPU 환경 전제.

---

## Task 1. 추론 서버(API 서빙 엔진) 구축 — Qwen3-Coder-480B 기반

**목표**: Qwen3-Coder-480B(또는 하드웨어 제약 시 동작 가능한 대체 모델)를 OpenAI 호환 API로 로컬에 띄운다.

- [x] **1-1. 하드웨어 확인 — 완료** (→ [`qwen3-coder-480b/README.md`](../qwen3-coder-480b/README.md) 환경)
  - H100 80GB × 8장(총 640GB VRAM) 확보 확인, RAM 1.7TB, `/data` 공유 스토리지 77TB
  - GPU 다중 확보 → Qwen3-Coder-480B 풀사이즈(FP8) + tensor-parallel 경로로 진행
- [x] **1-2. 추론 엔진 선택 및 설치 — 완료**
  - **vLLM** 채택(`vllm/vllm-openai:latest`, Docker 기반) — SGLang/TensorRT-LLM은 검토만 하고 미사용
- [x] **1-3. 모델 다운로드 및 서버 기동 — 완료** (→ [`qwen3-coder-480b/docker-compose.yml`](../qwen3-coder-480b/docker-compose.yml))
  - 공식 FP8 양자화(`Qwen3-Coder-480B-A35B-Instruct-FP8`, 482.2GB) 채택 — BF16(960.3GB)은 8×H100 640GB에 불가
  - `--tensor-parallel-size 8`은 FP8 블록 양자화 샤드 크기 문제로 실패 → `--tensor-parallel-size 4 --pipeline-parallel-size 2`로 해결 (상세: README.md 3단계)
- [x] **1-4. 헬스체크 — 완료**
  - `curl http://localhost:8000/v1/models` 정상 응답, `/v1/chat/completions`로 코드 생성 테스트 통과
  - opencode 연동을 위한 `--enable-auto-tool-choice --tool-call-parser qwen3_coder` 플래그까지 검증 완료 (→ README.md 5단계)
- [ ] **1-5. 성능/메모리 튜닝 — 미실시**
  - TP/PP 조합 문제 해결 과정에서 최소한의 조정만 했고, `--gpu-memory-utilization` 등 별도 튜닝이나 vLLM vs SGLang 처리량 비교는 하지 않음
  - opencode 테스트 완료 후 GPU 반납을 위해 컨테이너 종료됨 — 재개 시 진행 항목

---

## Task 2. 에이전틱 코딩 모델 추가 서빙

**목표**: Task 1(Qwen3-Coder-480B) 외에 GPU 자원 제약 안에서 추가로 서빙 가능한 모델을 확보한다.

- [ ] **2-3. 멀티 모델 운영 방식 결정**
  - 현재 폴더별 포트 배정: Qwen2.5-Coder-32B(검증용) 8123, Qwen3-Coder-480B 8000
  - 8×H100 전체를 요구하는 모델(Qwen3-Coder-480B TP4×PP2 등)은 서로 **동시 실행 불가** — 한 번에 하나만 띄우는 온디맨드 전환 방식이 사실상 강제됨. 동시 운영하려면 각 모델을 GPU 부분집합으로 쪼개거나(양자화로 footprint를 더 줄이거나) 더 작은 모델(Qwen2.5-Coder-32B 등)과만 병행 가능
  - **gpt-oss-120b(포트 8500)가 이 제약의 첫 돌파구**: MXFP4 사전 양자화로 가중치가 65.3GB뿐이라 H100 1장(TP=1)으로 서빙 가능 → 나머지 7장을 다른 사용자/모델에 남길 수 있음 (→ [`gpt-oss-120b/`](./gpt-oss-120b/README.md), [`docs/gpt-oss-120b.md`](./docs/gpt-oss-120b.md))
- [ ] **2-5. gpt-oss-120b 서빙 (→ [`gpt-oss-120b/`](./gpt-oss-120b/))**
  - [x] 사양·서빙 방식 조사 및 설정 파일 작성 (docker-compose.yml / .env.example / README.md / opencode.json.example) — 근거는 [`docs/gpt-oss-120b.md`](./docs/gpt-oss-120b.md)
  - [x] vLLM 파서 이름을 이미지(vLLM 0.25.1) 안에서 직접 검증: `--tool-call-parser openai`, `--reasoning-parser openai_gptoss` (모델명 `gpt_oss`가 아님)
  - [x] KV 캐시 사전 계산: full attention 18레이어 × 2KiB/토큰 = 36KiB/토큰 → 131K 컨텍스트 1시퀀스당 약 4.5GiB
  - [x] 실행 방식 결정: **프로파일 A (GPU 1장, TP=1) + max-model-len 131072**
  - [x] **실제 기동 및 헬스체크 성공** — 가중치 60.77GiB 다운로드 100.7초 / 로딩 227.6초, `Application startup complete`, `/v1/models` 정상. **KV 캐시 10.52GiB = 303,646 토큰**으로 131072 컨텍스트가 TP=1에서 실제로 잡힘(사전 계산과 일치)
  - [x] **새 함정 발견 및 수정**: compose의 `deploy.resources.reservations.devices`에 `device_ids`가 없으면 Docker가 `NVIDIA_VISIBLE_DEVICES`를 덮어써서 GPU 지정이 무시됨 → `GPU_DEVICE_IDS=4`인데 다른 사용자가 쓰던 GPU 0에 올라감. `CUDA_VISIBLE_DEVICES` 추가로 해결. 기존 폴더는 전부 TP=8이라 드러나지 않았던 문제
  - [x] 사용자 지시로 기동 실험 중단, `docker compose down` + 고아 프로세스 없음 확인 후 GPU 반납 (가중치는 캐시에 남아 재기동 시 재다운로드 불필요)
  - [ ] 실제 추론 요청(`/v1/chat/completions`) 응답 품질/속도 확인 — 미검증
  - [ ] opencode tool-calling 연동 검증 — provider(`local-gptoss/gpt-oss-120b`) 등록까지만 완료, tool-calling 자체는 **미검증**
- [ ] **2-4. 헬스체크 및 장문 컨텍스트 테스트**
  - `curl http://localhost:8001/v1/models` 확인
  - 대형 리포지토리 코드 다건을 컨텍스트에 넣어 1M 컨텍스트 실사용 가능 여부 검증 (지연시간/메모리 관찰)

---

## Task 3. (공통) 에이전트/에디터 프론트엔드 연결

**목표**: 위 두 로컬 API를 실제 코딩 에이전트에서 사용 가능하게 연결한다.

- [ ] **3-1. 프론트엔드 선택**
  - CLI 에이전트: opencode, Aider (`--openai-api-base`로 baseUrl 지정)
  - 자율 에이전트: OpenHands (vLLM/SGLang 백엔드 지원)
  - 에디터 확장: Continue.dev, Cline, Roo Code (provider `openai` + baseUrl을 `localhost:8000/v1` 또는 `localhost:8001/v1`로 지정)
- [ ] **3-2. Claude Code 자체를 로컬 모델과 쓰고 싶은 경우 (선택)**
  - 방법 A: LiteLLM / claude-code-proxy / UniClaudeProxy 로 Anthropic↔OpenAI 포맷 변환 후 `ANTHROPIC_BASE_URL`을 프록시로 지정
  - 방법 B: Ollama v0.14+ 또는 LM Studio 0.4.1의 네이티브 Anthropic 호환 엔드포인트(`/api/messages`, `/v1/messages`) 사용 시 프록시 불필요
- [ ] **3-3. 엔드투엔드 검증**
  - 각 프론트엔드에서 실제 코딩 작업(버그 수정, 함수 생성 등) 1건씩 수행해 응답 품질/속도 확인

---

