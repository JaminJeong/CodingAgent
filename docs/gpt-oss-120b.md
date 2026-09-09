# gpt-oss-120b 조사 및 구성 기록

작성일: 2026-08-04
대상: `openai/gpt-oss-120b`
결과물: [`../gpt-oss-120b/`](../gpt-oss-120b/) (docker-compose.yml, .env.example, README.md, opencode.json.example)

> **상태: 조사 + 설정 작성 + 프로파일 A(TP=1) 기동 성공 확인. 이후 사용자 지시로 중단, GPU 반납.**
> 실제 추론 요청과 opencode tool-calling 검증은 하지 않았다. 자세한 실행 로그는 9절.

---

## 1. 왜 이 모델인가

이 저장소가 지금까지 다룬 모델은 전부 8×H100 전체를 요구했다.

| 모델 | 가중치 | 필요 GPU |
|---|---|---|
| Qwen3-Coder-480B-FP8 | — | 8장 (TP4×PP2) |
| **gpt-oss-120b MXFP4** | **65.3GB** | **1장** |

여기는 **공유 서버**이고 다른 사용자가 GPU를 쓰고 있을 수 있다는 것이 이 저장소 전체를
관통하는 제약이다([`../CLAUDE.md`](../CLAUDE.md)). `TASK.md` 2-3 항목은 "8장을 다 요구하는
모델끼리는 동시 실행 불가 → 온디맨드 전환이 강제됨"을 미해결 문제로 남겨두고 있다.

gpt-oss-120b는 그 제약을 처음으로 벗어나는 후보다. MoE 가중치가 **MXFP4로 사전 양자화되어**
배포되므로 별도 양자화 작업 없이 H100 80GB 한 장에 들어간다.

## 2. 모델 사양 (config.json 직접 확인)

`https://huggingface.co/openai/gpt-oss-120b/raw/main/config.json` 실측:

```
architectures            GptOssForCausalLM
model_type               gpt_oss
num_hidden_layers        36
layer_types              sliding_attention / full_attention 교대 (각 18개)
sliding_window           128
num_attention_heads      64
num_key_value_heads      8
head_dim                 64
hidden_size              2880
intermediate_size        2880
num_local_experts        128
num_experts_per_tok      4
max_position_embeddings  131072
rope_scaling             yarn, factor 32, original 4096
quantization_config      quant_method = mxfp4
                         modules_to_not_convert = self_attn, mlp.router, embed_tokens, lm_head
```

- 총 117B / 활성 5.1B
- 라이선스 Apache 2.0 (copyleft·특허 리스크 없음)
- gated 아님 → HF_TOKEN 없이 다운로드 가능

### 다운로드 크기 — 195.8GB가 아니라 65.3GB

HF API(`?blobs=true`)로 파일별 크기를 집계한 결과:

| 경로 | 파일 수 | 크기 |
|---|---|---|
| `(root)` — safetensors 15샤드 + config/tokenizer | 26 | **65.3 GB** |
| `original/` | 10 | 65.2 GB |
| `metal/` | 1 | 65.2 GB |
| 합계 | 37 | 195.8 GB |

`original/`(OpenAI 원본 포맷)과 `metal/`(Apple Metal용)은 같은 가중치의 **중복본**이다.
vLLM/transformers 경로는 루트 safetensors만 받으므로 **실제 다운로드는 약 65.3GB**.
`git clone`으로 통째로 받으면 195.8GB를 받게 되니 주의.

현재 HF 캐시(`/data/models/huggingface/hub`, 1.5TB 사용 중, `/data` 여유 73TB)에
gpt-oss는 **아직 없다** — 최초 기동 시 다운로드가 발생한다.

## 3. KV 캐시 계산 (프로파일 선택의 근거)

이 저장소는 지금까지 KV 캐시 부족으로 두 번 기동에 실패한 경험이 있다.
그래서 이번에는 **띄우기 전에** 계산해둔다.

레이어 36개 중 full attention은 18개, 나머지 18개는 sliding window 128이라 사실상 무시 가능
(시퀀스당 18 × 128 × 2KiB ≈ 4.5MiB).

```
토큰 1개 / 레이어 1개당 KV(bf16) = 2(K,V) × 8(kv_heads) × 64(head_dim) × 2byte = 2,048 B = 2 KiB
full attention 18개          → 36 KiB / 토큰
131,072 토큰 (full context)  → 131072 × 36 KiB ≈ 4.5 GiB / 시퀀스
```

시퀀스당 4.5GiB는 이 저장소의 다른 모델들에 비하면 매우 가볍다.

프로파일별 여유 추정 (H100 80GB = 79.6GiB 실사용):

| 프로파일 | GPU | GPU당 가중치 | util | GPU당 KV 여유 | 총 KV | 131K 동시 시퀀스 |
|---|---|---|---|---|---|---|
| A | 1 | ~60.8 GiB | 0.95 | ~11–12 GiB | ~12 GiB | **2~3개** |
| B | 2 | ~30.4 GiB | 0.90 | ~40 GiB | ~80 GiB | 17개 이상 |
| C | 8 | ~7.6 GiB | 0.85 | ~60 GiB | ~480 GiB | 100개 이상 |

> 가중치 외에 activation/CUDA graph 풀이 GPU당 2~3GiB 추가로 필요하므로 A는 여유가 빠듯하다.
> 실제로 vLLM 공식 레시피도 "H100 TP1은 기본 설정에서 CUDA OOM"이라고 명시하고 있다.

**해석**: 1인~소수 사용자가 코딩 에이전트로 쓰는 용도라면 프로파일 A로 충분하다.
동시 세션이 여럿이거나 긴 컨텍스트를 자주 채운다면 B가 안전하다. C는 처리량은 최대지만
"8장 전부 점유 → 다른 모델과 동시 운영 불가"라는 원래 문제로 되돌아간다.

## 4. 서빙 방식 후보

| 방식 | 장점 | 단점 | 판단 |
|---|---|---|---|
| **vLLM (docker)** | 이 저장소 전 폴더가 쓰는 방식, OpenAI 호환 API, MXFP4 native, tool/reasoning 파서 제공, day-0 지원 | 단일 H100 TP=1 OOM 이슈가 문서화되어 있음 | **채택** |
| SGLang | prefix caching 많은 에이전트 워크로드에서 처리량 유리 | 이 저장소에 SGLang docker 경로 선례 없음 | 후순위 |
| Ollama | 설치·실행이 가장 간단, harmony/tool calling 네이티브, Anthropic 호환 엔드포인트도 있어 Claude Code 직결 가능(TASK 3-2) | 처리량/동시성이 vLLM 대비 낮음, 서버 전체를 재구성해야 함 | Task 3-2용 별도 검토 대상 |
| llama.cpp / LM Studio | GGUF, CPU 오프로딩 유연 | H100 8장 환경에서 굳이 쓸 이유 없음 | 제외 |
| TensorRT-LLM | H100 특화 최적화 | 설정 복잡도 높음(TASK.md에서도 후순위로 명시) | 제외 |

vLLM 요구사항: **vLLM ≥ 0.10.0, CUDA ≥ 12.8**. 이 서버는 CUDA 12.8이고,
로컬 `vllm/vllm-openai:latest` 이미지는 **vLLM 0.25.1**(직접 확인)이라 조건을 만족한다.

## 5. vLLM 플래그 결정 근거

### 5-1. tool / reasoning 파서 이름 — 이미지 안에서 직접 확인

gpt-oss는 harmony 응답 포맷을 쓴다. 파서 이름이 모델명(`gpt_oss`)과 다르므로 추측하면 틀린다.
`vllm/vllm-openai:latest` 컨테이너 안에서 등록 테이블을 직접 열어 확인했다:

```
vllm/tool_parsers/__init__.py   :  "openai"        -> gptoss_tool_parser.GptOssToolParser
vllm/reasoning/__init__.py      :  "openai_gptoss" -> gptoss_reasoning_parser.GptOssReasoningParser
```

`GptOssToolParser`의 docstring에는 "Stub tool parser for gpt-oss/harmony models. All output
parsing is handled by HarmonyParser."라고 되어 있다 — 즉 이 플래그는 기능 선언용이고
실제 파싱은 harmony 파서가 담당한다.

→ `--enable-auto-tool-choice --tool-call-parser openai --reasoning-parser openai_gptoss`

이 저장소의 규칙("tool-calling 클라이언트는 `--enable-auto-tool-choice` + 맞는 파서가 없으면
`tool_choice: auto` 요청이 아예 실패한다")을 그대로 적용한 것이다.

### 5-2. TP=1 OOM 완화책

vLLM 공식 레시피의 트러블슈팅 항목에 그대로 적혀 있다:

> **H100 TP1 OOM:** `--gpu-memory-utilization 0.95 --max-num-batched-tokens 1024`

이걸 `.env.example` 기본값으로 넣었다. `--max-num-seqs 8`은 레시피에 없지만,
이전 모델 기동 시도에서 **기본 `max-num-seqs`가 샘플러 워밍업 중 CUDA OOM을 냈던** 경험을 반영해
보수적으로 잡았다. TP=2 이상에서는 "gpu-memory-utilization을 0.95 미만으로 둘 것"이 레시피의
주의사항이라 프로파일 B는 0.90으로 낮춰뒀다.

### 5-3. MXFP4 블록 나눗셈 — 이론상 위험, 미검증

Qwen3-Coder-480B는 FP8 블록 양자화(block_n=128)와 MoE 샤드 크기가 나누어떨어지지 않아
TP=8이 실패했다. gpt-oss도 같은 계열의 위험이 있다:

```
intermediate_size = 2880,  MXFP4 블록 = 32
2880 / 2 = 1440  →  1440 % 32 = 0   OK
2880 / 4 =  720  →   720 % 32 = 16  나누어떨어지지 않음
2880 / 8 =  360  →   360 % 32 =  8  나누어떨어지지 않음
```

다만 vLLM 공식 레시피는 TP=4, TP=8 예제를 그대로 싣고 있고(8×H100 예제 포함) 실사용
보고도 많으므로, 실제로는 커널 패딩이나 expert-parallel 경로로 처리되는 것으로 보인다.
**직접 실행해 확인한 사실이 아니다.** TP=4/8에서 샤딩 오류가 나면
`--enable-expert-parallel`(전문가 128개를 GPU 수로 분할, 128/8 = 16으로 깔끔)로 우회한다.

### 5-4. `restart: "no"`

과거 engine-init 크래시 + `restart: unless-stopped` 조합으로 대용량 가중치를
**126회 / 5시간** 재로드한 사고가 있었다. 이 저장소는 같은 실수를 두 번 했다.
안정 확인 전까지 `"no"`로 시작한다.

### 5-5. 넣지 않은 옵션

- `--async-scheduling` : 레시피가 gpt-oss 처리량 향상용으로 권장하나, 첫 기동을 최소 구성으로
  성공시킨 뒤 추가하는 것이 이 저장소 절차상 맞다. 주석으로만 남김.
- `--no-enable-prefix-caching` : 레시피의 벤치마크 예제에 있지만, 코딩 에이전트는 동일 시스템
  프롬프트 + 파일 컨텍스트를 반복 전송하므로 prefix caching이 오히려 크게 이득이다. 켠 채로 둔다.
- `--tool-server demo` : gpt-oss 내장 도구(브라우저/파이썬 실행) 데모용. `uv pip install gpt-oss`와
  샌드박스 설정이 추가로 필요하고, opencode가 자체 도구를 제공하므로 불필요.

## 6. 알려진 이슈 / 사전 대비

| 이슈 | 내용 | 대비 |
|---|---|---|
| harmony 토크나이저 다운로드 실패 | `o200k_base` / `cl100k_base` tiktoken 파일을 런타임에 받다 실패 | tiktoken 파일 사전 다운로드 + `TIKTOKEN_ENCODINGS_BASE` (compose에 주석으로 준비) |
| vLLM 버전 퇴행 (#33155) | v0.11.2에서 되던 단일 H100 구동이 v0.14.1에서 KV 캐시 할당 OOM. **해결 없이 stale 종료** | 정상 기동 확인 즉시 `VLLM_IMAGE`를 실제 버전으로 고정 |
| `reasoning_content` vs `reasoning` 키 | vLLM은 `reasoning_content`로 내려주는데 일부 클라이언트는 `reasoning`을 기대 (openai/codex#3721) | opencode에서 추론 텍스트가 안 보이면 이 문제 의심 |
| 스트리밍 + `tool_call_parser=openai` 크래시 (#36849, 20B 사례) | chat completions 스트리밍 중 IndexError | opencode 연동 시 재현되는지 확인 필요 |
| Triton 충돌 | 환경에 `pytorch-triton` 등 중복 설치 시 `tl.language not defined` | 공식 docker 이미지 사용으로 회피 |

## 7. 사용자 결정 사항

사용자가 다음과 같이 결정했다:

1. **실행 프로파일: A** (GPU 1장, TP=1) — GPU 7장을 다른 사용자에게 남기는 쪽
2. **최대 컨텍스트: 131072** (공식 최대 그대로)
3. **실제 기동함** — 이후 기동 실험은 중단하고 GPU 반납

## 8. 실행 결과 (2026-08-04)

### 8-1. 기동 성공 — 사전 계산이 실측과 일치

vLLM 0.25.1 / H100 1장 / TP=1 / max-model-len=131072 / gpu-memory-utilization=0.95:

```
quantization=gpt_oss_mxfp4, dtype=torch.bfloat16, tensor_parallel_size=1
Using FLASH_ATTN attention backend / FlashAttention version 3
Using 'TRITON' Mxfp4 MoE backend
Time spent downloading weights: 100.695930 seconds
Filesystem type for checkpoints: NFS. Checkpoint size: 60.77 GiB. Available RAM: 1708.28 GiB
Model loading took 61.29 GiB memory and 227.558701 seconds
Available KV cache memory: 10.52 GiB
GPU KV cache size: 303,646 tokens
INFO: Application startup complete.
```

3절에서 미리 계산한 값과 대조:

| 항목 | 사전 계산 | 실측 | 결과 |
|---|---|---|---|
| 가중치 | ~60.8 GiB | 61.29 GiB | 일치 |
| KV 여유 | 11~12 GiB | 10.52 GiB | 근사(약간 적음 — CUDA graph 프로파일링 반영) |
| 토큰당 KV | 36 KiB | 10.52GiB ÷ 303,646 = 36.3 KiB | **일치** |
| 131K 동시 시퀀스 | 2~3개 | 303,646 ÷ 131,072 = 2.31개 | 일치 |

→ **TP=1에서 공식 최대 컨텍스트 131072가 실제로 잡힌다.** `--max-model-len`을 낮출 필요가 없었다.

vLLM이 추가로 알려준 것: CUDA graph 메모리 프로파일링(v0.21.0부터 기본)이 켜져 있어
`--gpu-memory-utilization 0.95`는 실효 0.9439에 해당한다. 예전 기준의 KV 크기를 그대로
유지하려면 0.9561로 올리라는 안내가 로그에 나온다 — KV를 더 확보하고 싶을 때의 여지.

헬스체크 `curl http://localhost:8500/v1/models` → `max_model_len: 131072` 정상 응답.
opencode에는 `local-gptoss` provider를 등록해 `opencode models`에
`local-gptoss/gpt-oss-120b`가 표시되는 것까지 확인.

### 8-2. 새로 발견한 함정 — compose의 GPU 지정이 무시된다

`.env`에 `GPU_DEVICE_IDS=4`를 넣었는데 **다른 사용자가 쓰던 물리 GPU 0에 올라갔다.**

`deploy.resources.reservations.devices`에 `device_ids` 없이 `capabilities: [gpu]`만 두면
Docker가 `--gpus all` 상당으로 처리하면서 `NVIDIA_VISIBLE_DEVICES`를 덮어쓴다.
컨테이너 안에서 `nvidia-smi`를 돌려 8장이 전부 보이는 것을 확인했고, vLLM은 TP=1이라
그중 0번을 잡았다. 호스트에서 확인한 결과:

```
$ nvidia-smi -i 0 --query-compute-apps=pid,used_memory,process_name --format=csv,noheader
3382183,  2960 MiB, /data/users/<다른사용자>/utmos-eval/.venv/bin/python   <- 다른 사용자
3387615, 75126 MiB, VLLM::EngineCore                                     <- 우리 컨테이너
```

이 저장소의 기존 폴더는 전부 TP=8(8장 전체)이라 이 함정이 드러날 일이 없었다.
**gpt-oss처럼 GPU 부분집합만 쓰는 구성에서 처음 문제가 된 것이다.**

수정: `CUDA_VISIBLE_DEVICES=${GPU_DEVICE_IDS}`를 environment에 추가.
컨테이너 안 CUDA 런타임이 직접 해석하므로 확실하게 적용된다
(`docker compose config`에서 `CUDA_VISIBLE_DEVICES: "4"` 반영 확인, 재기동까지 수행).

> 앞으로 이 저장소에 GPU 부분집합을 쓰는 모델 폴더를 추가할 때는 반드시 같은 처리를 할 것.

### 8-3. 중단 및 GPU 반납

사용자 지시로 기동 실험을 중단했다. `docker compose down` 후
`nvidia-smi --query-compute-apps`로 VLLM 고아 프로세스가 남지 않은 것을 확인하고 GPU를 반납했다.
가중치 60.77GiB는 `/data/models/huggingface`에 남아 있어 재기동 시 재다운로드가 불필요하다.

**검증하지 않은 것**: 실제 추론 요청의 응답 품질/속도, opencode tool-calling 동작
(과거 다른 모델에서 크래시했던 지점이라 재개 시 최우선 확인 대상).

## 9. 조사 출처

- [openai/gpt-oss-120b — Hugging Face](https://huggingface.co/openai/gpt-oss-120b)
- [openai/gpt-oss-120b — vLLM Recipes](https://recipes.vllm.ai/openai/gpt-oss-120b)
- [GPT OSS — vLLM Recipes (docs.vllm.ai)](https://docs.vllm.ai/projects/recipes/en/stable/OpenAI/GPT-OSS.html)
- [How to run gpt-oss with vLLM — OpenAI Cookbook](https://developers.openai.com/cookbook/articles/gpt-oss/run-vllm)
- [gptoss_reasoning_parser — vLLM Documentation](https://docs.vllm.ai/en/v0.11.0/api/vllm/reasoning/gptoss_reasoning_parser.html)
- [Deploy GPT-OSS 120B on H100 with vLLM — Simplismart](https://simplismart.ai/blog/deploy-gpt-oss-120b-h100-vllm)
- [Example model deployment with vLLM — GitLab Docs](https://docs.gitlab.com/administration/gitlab_duo_self_hosted/vllm_gpt_oss_120b/)
- [\[Bug\] gptoss120B is OOM on one H100 after upgrading v0.11.2 → v0.14.1 — vllm#33155](https://github.com/vllm-project/vllm/issues/33155)
- [\[Bug\] gpt-oss-120b Chat Completions tool_call support — vllm#22578](https://github.com/vllm-project/vllm/issues/22578)
- [\[Bug\] streaming crash with tool_call_parser=openai — vllm#36849](https://github.com/vllm-project/vllm/issues/36849)
- [Fall back to read 'reasoning_content' — openai/codex#3721](https://github.com/openai/codex/issues/3721)
- [GPT-OSS Performance Optimizations on NVIDIA Blackwell — vLLM Blog](https://vllm.ai/blog/2026-02-01-gpt-oss-optimizations)
