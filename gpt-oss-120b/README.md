# gpt-oss-120b (openai/gpt-oss-120b) — vLLM 서빙 구성

> **현재 상태 (2026-08-04): 프로파일 A(TP=1, GPU 1장)로 기동 성공 확인 후 사용자 지시로 중단, GPU 반납.**
> 가중치 60.77GiB는 HF 캐시에 남아 있어 재기동 시 재다운로드 불필요.
> `/v1/models` 헬스체크까지 통과했고, **실제 추론 요청과 opencode tool-calling 검증은 하지 않았다.**
> 조사 근거와 수치의 출처는 [`../docs/gpt-oss-120b.md`](../docs/gpt-oss-120b.md)에 정리했다.

## 이 모델이 이 저장소에서 특별한 이유

이 저장소의 기존 모델(Qwen3-Coder-480B)은 **8×H100 전체**를
요구해서 한 번에 하나만 띄울 수 있었다(→ [`../TASK.md`](../TASK.md) 2-3 항목).

gpt-oss-120b는 MoE 가중치가 **MXFP4로 사전 양자화되어 배포**되고 실측 65.3GB(≈60.8GiB)라
**H100 80GB 한 장에 들어간다.** 즉 이 저장소에서 처음으로

- GPU 1장만 점유하고 나머지 7장을 다른 사용자/모델에 남겨둘 수 있고,
- 다른 대형 모델과 **동시 운영**이 가능한

구성이다. 공유 서버라는 이 환경의 제약을 감안하면 가장 실용적인 후보다.

## 모델 사양 (config.json 실측)

| 항목 | 값 |
|---|---|
| 총 파라미터 / 활성 | 117B / 5.1B (MoE) |
| 전문가 수 / 토큰당 활성 | 128 / 4 |
| 레이어 | 36 (sliding 18 + full attention 18 교대) |
| attention heads / KV heads / head_dim | 64 / 8 / 64 |
| hidden / intermediate | 2880 / 2880 |
| sliding window | 128 |
| 최대 컨텍스트 | 131072 (YaRN, 원본 4096 × factor 32) |
| 양자화 | MXFP4 (MoE만. attention/router/embed/lm_head는 비양자화) |
| 라이선스 | Apache 2.0 |
| 다운로드 크기 | 루트 safetensors 65.3GB (repo 전체는 195.8GB — `original/`, `metal/`은 다른 런타임용 중복본) |

## 사용법

```bash
cd gpt-oss-120b/
cp .env.example .env

# 반드시 먼저 — 이 서버는 공유 서버다. 8장이 다 비어 있다고 가정하지 말 것.
nvidia-smi --query-gpu=index,memory.total,memory.used,memory.free --format=csv

# .env에서 GPU_DEVICE_IDS / TENSOR_PARALLEL_SIZE를 실제 여유 GPU에 맞춘다 (장수 일치 필수)
sudo docker compose up -d
sudo docker logs -f vllm-gpt-oss-120b
```

헬스체크:

```bash
curl -s http://localhost:8500/v1/models

curl -s http://localhost:8500/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"gpt-oss-120b","messages":[{"role":"user","content":"hello"}],"max_tokens":100}'
```

추론 강도(reasoning effort) 지정 — gpt-oss는 low/medium/high 3단계를 지원하며, vLLM에서는
chat template 인자로 넘긴다:

```bash
curl -s http://localhost:8500/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"gpt-oss-120b",
       "messages":[{"role":"user","content":"두 수의 최대공약수를 구하는 파이썬 함수"}],
       "chat_template_kwargs":{"reasoning_effort":"high"},
       "max_tokens":2048}'
```

종료 및 GPU 반납 (이 저장소의 규칙: `down` 후 반드시 확인):

```bash
sudo docker compose down
nvidia-smi --query-compute-apps=pid,used_memory,process_name --format=csv
```

opencode 연동은 [`opencode.json.example`](./opencode.json.example)의 `local-gptoss` provider
블록을 `~/.config/opencode/opencode.json`에 병합한다.

```bash
opencode models | grep gpt-oss
opencode run -m local-gptoss/gpt-oss-120b "이 리포의 docs/README.md를 요약해줘"
```

## 실행 프로파일

| 프로파일 | GPU | TP | 특징 |
|---|---|---|---|
| **A (기본)** | 1장 | 1 | 7장을 남긴다. KV 여유 약 11~12GiB → 131K 컨텍스트 기준 동시 2~3 시퀀스 |
| **B** | 2장 | 2 | 6장을 남긴다. KV 여유가 크게 늘어 동시성 확보. `--gpu-memory-utilization`은 0.95 **미만**으로 |
| **C** | 8장 | 8 | 처리량 최대. 대신 다른 모델과 동시 운영 불가 — 이 저장소가 계속 겪은 문제로 되돌아감 |

수치 근거는 [`../docs/gpt-oss-120b.md`](../docs/gpt-oss-120b.md)의 "KV 캐시 계산" 절 참고.

## 실행 결과 (2026-08-04, 프로파일 A / TP=1)

vLLM 0.25.1, H100 1장, `max-model-len=131072`, `gpu-memory-utilization=0.95`로 **기동 성공**.

```
quantization=gpt_oss_mxfp4, dtype=torch.bfloat16, tensor_parallel_size=1
Using FLASH_ATTN attention backend / FlashAttention version 3
Using 'TRITON' Mxfp4 MoE backend
Checkpoint size: 60.77 GiB          # 다운로드 100.7초
Model loading took 61.29 GiB memory and 227.6 seconds
Available KV cache memory: 10.52 GiB
GPU KV cache size: 303,646 tokens
INFO: Application startup complete.
```

- `curl http://localhost:8500/v1/models` → `max_model_len: 131072` 정상 응답
- **사전 계산이 실측과 일치했다**: 토큰당 36KiB × 303,646 = 10.42GiB ≈ 실측 10.52GiB.
  131072 컨텍스트 기준 동시 처리 가능 시퀀스는 303,646 ÷ 131,072 = **약 2.3개** — 사전 추정(2~3개)과 같다.
- 즉 **TP=1에서 공식 최대 컨텍스트 131072가 실제로 잡힌다.** `max-model-len`을 낮출 필요가 없었다.
- `--enable-auto-tool-choice --tool-call-parser openai --reasoning-parser openai_gptoss`
  세 플래그 모두 거부 없이 수락됨(`reasoning_parser='openai_gptoss'`가 config에 반영된 것 확인).

opencode에는 `local-gptoss` provider를 등록해 `opencode models`에
`local-gptoss/gpt-oss-120b`가 뜨는 것까지 확인했다.

**하지 않은 것**: 실제 추론 요청(`/v1/chat/completions`) 응답 확인, opencode tool-calling 검증.
사용자 지시로 기동 실험을 중단하고 `docker compose down` + 고아 프로세스 없음 확인 후 GPU를 반납했다.

## 설정에 반영한 사항 / 주의점

### 0. GPU 지정이 안 먹는 함정 — 실제로 겪음 (중요)

`.env`에 `GPU_DEVICE_IDS=4`를 넣고 기동했는데 **다른 사용자가 쓰고 있던 물리 GPU 0에 올라갔다.**

원인: `deploy.resources.reservations.devices`에 `device_ids` 없이 `capabilities: [gpu]`만 두면
Docker가 `--gpus all` 상당으로 처리하면서 **`NVIDIA_VISIBLE_DEVICES`를 덮어쓴다.**
컨테이너 안에서 `nvidia-smi`를 돌려보니 8장이 전부 보였고, vLLM은 TP=1이므로 그중 0번을 잡았다.

```
# 컨테이너 내부 — 4번만 보여야 하는데 8장 전부 보임
0, GPU-c5a139ac..., 78107 MiB   <- 여기에 올라감 (다른 사용자와 공유)
1, GPU-0e94d18f...,  2974 MiB
...
```

이 저장소의 기존 폴더는 전부 TP=8(8장 전체)이라 이 함정이 드러날 일이 없었다.
**수정**: `CUDA_VISIBLE_DEVICES=${GPU_DEVICE_IDS}`를 environment에 추가.
컨테이너 안 CUDA 런타임이 직접 해석하므로 확실하게 먹는다(재기동해서 `CUDA_VISIBLE_DEVICES: "4"`가
반영되는 것까지 확인). 공유 서버에서 1~2장만 쓰는 구성이라면 반드시 필요한 설정이다.

### 1. 파서 이름 검증 (모델명과 다름)

gpt-oss는 harmony 응답 포맷을 쓰기 때문에 파서 플래그 이름이 `gpt_oss`가 **아니다.**
로컬 `vllm/vllm-openai:latest`(vLLM **0.25.1**) 이미지 안에서 등록 이름을 직접 확인했다:

```
vllm/tool_parsers/__init__.py        "openai"        -> gptoss_tool_parser.GptOssToolParser
vllm/reasoning/__init__.py           "openai_gptoss" -> gptoss_reasoning_parser.GptOssReasoningParser
```

따라서 `--tool-call-parser openai --reasoning-parser openai_gptoss --enable-auto-tool-choice`.
(`GptOssToolParser`는 스텁이고 실제 파싱은 `HarmonyParser`가 수행한다고 소스에 명시되어 있다.)

### 2. TP=1 OOM은 "알려진 이슈"라서 처음부터 완화책을 넣어둠

vLLM 공식 레시피가 H100 TP=1에서 기본값으로는 CUDA OOM이 난다고 명시하고,
`--gpu-memory-utilization 0.95 --max-num-batched-tokens 1024`를 완화책으로 제시한다.
`.env.example` 기본값에 이미 반영해뒀다. `--max-num-seqs 8`은 이전 모델 기동에서 기본
`max-num-seqs`가 샘플러 워밍업 중 OOM을 냈던 경험을 반영해 보수적으로 잡은 값이다.

### 3. `restart: "no"`로 시작

과거 engine-init 크래시 + `restart: unless-stopped` 조합으로 대용량 가중치를
**126회** 재로드한 사고가 있었다. 안정 확인 전까지 `"no"` 유지 — 확인 후 변경할 것.

### 4. MXFP4 블록 나눗셈 위험 (미검증)

Qwen3-Coder-480B는 FP8 블록 양자화(block_n=128)와 샤드 크기가 나누어떨어지지 않아 TP=8이
실패했다. gpt-oss도 `intermediate_size = 2880`이고 MXFP4 블록은 32인데,
`2880 / 8 = 360`, `2880 / 4 = 720`은 32로 나누어떨어지지 **않는다**(2880/2 = 1440은 나누어떨어짐).

다만 vLLM 공식 레시피는 TP=4, TP=8 예제를 그대로 제시하고 있어, 실제로는 커널 패딩이나
expert-parallel 경로로 처리되는 것으로 보인다. **직접 실행해 확인한 사실이 아니므로**
TP=4/8을 쓸 경우 실패하면 `docker-compose.yml`의 `--enable-expert-parallel`(전문가 128개를
GPU 수로 분할 — 128/8 = 16으로 깔끔하게 나뉜다) 주석을 해제해볼 것.

### 5. 문제 대비 — harmony 토크나이저 다운로드 실패

vLLM 레시피에 `o200k_base` / `cl100k_base` tiktoken 파일을 런타임에 받아오다 실패하는
사례가 문서화되어 있다. 발생하면:

```bash
mkdir -p tiktoken_encodings
wget -O tiktoken_encodings/o200k_base.tiktoken \
  "https://openaipublic.blob.core.windows.net/encodings/o200k_base.tiktoken"
wget -O tiktoken_encodings/cl100k_base.tiktoken \
  "https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken"
```

받은 뒤 `docker-compose.yml`의 `TIKTOKEN_ENCODINGS_BASE` 환경변수와 볼륨 마운트 주석을 해제한다.

### 6. 이미지 버전 고정 권장

vLLM 이슈 #33155에 "v0.11.2에서 되던 단일 H100 gpt-oss-120b가 v0.14.1로 올리니 KV 캐시
할당 단계에서 OOM"이라는 리포트가 있다(해결 없이 stale 종료). 로컬 `:latest`는 현재
**0.25.1**이며 그 리포트보다 훨씬 상위 버전이라 해당 퇴행이 남아 있는지는 알 수 없다.
정상 기동을 확인하는 즉시 `.env`의 `VLLM_IMAGE`를 실제 버전 태그로 고정할 것.

## 확인 완료 / 미확인

확인 완료:
- [x] 실제 기동 (가중치 60.77GiB 다운로드 100.7초, 로딩 227.6초)
- [x] TP=1에서 `max-model-len=131072`가 실제로 잡힘 (KV 303,646 토큰 = 131K 시퀀스 2.3개분)
- [x] tool/reasoning 파서 플래그 수락 및 `/v1/models` 헬스체크
- [x] GPU 지정 함정(위 0번) 발견 및 수정

미확인:
- [ ] 실제 추론 요청(`/v1/chat/completions`) 응답 품질/속도
- [ ] opencode tool-calling 정상 동작 — 과거 다른 모델은 이 지점에서 크래시했다
- [ ] TP=4/8에서 MXFP4 샤딩이 실패하는지 (아래 4번)
- [ ] `--async-scheduling`, `--enable-expert-parallel` 적용 시 효과
