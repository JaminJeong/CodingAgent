# H100급 고성능 GPU 환경에서의 코딩 에이전트 구성 (2026-07 기준)

고성능 GPU(H100 등)를 보유한 경우, 아래 3단 구성으로 접근한다.

1. 로컬에서 돌릴 **오픈소스 코딩 모델**
2. 모델을 API로 서빙하는 **추론 엔진**
3. 그 API에 붙는 **에이전트/에디터 프론트엔드**

## 1. 추천 오픈소스 코딩 모델

| 모델 | 특징 | H100 요구사항 |
|---|---|---|
| Qwen3-Coder-480B | SWE-bench Verified 69.6%, 코딩 에이전트용 튜닝 | 다중 H100 (양자화 시 1장도 가능) |
| DeepSeek-V3.2 / V4-Flash | MIT 라이선스, 1M 컨텍스트, SWE-bench ~70% | 80GB H100 1장으로 가동 가능 |
| GLM-5.2 | 자체 호스팅 가능한 모델 중 벤치마크 최상위 | 서버급 (다중 GPU) |
| Qwen3.6-27B, Devstral Small 2 | 로컬 1장 GPU에서 시작하기 좋은 경량 모델 | H100 1장으로 충분, 여유 있음 |

- H100 1장(80GB): DeepSeek-V4-Flash나 Qwen3.6/Devstral 계열로 시작
- H100 여러 장: GLM-5.2 같은 프론티어급 모델도 자체 호스팅 가능

## 2. 추론 서버 (모델을 API로 띄우는 엔진)

- **vLLM**: 가장 널리 쓰이는 OpenAI 호환 서빙 엔진. Docker로 몇 분 내 구동 가능.
  ```bash
  docker run --gpus all -p 8000:8000 vllm/vllm-openai \
    --model Qwen/Qwen3-Coder-... --max-model-len 131072
  ```
- **SGLang**: H100에서 vLLM 대비 약 29% 높은 처리량(prefix 캐싱이 중요한 RAG/에이전트 워크로드에서는 최대 6배). 최근 H100 환경에서 더 선호되는 추세.
- **TensorRT-LLM**: NVIDIA 자체 최적화 엔진. H100 특화 최적화가 강점이지만 설정이 더 복잡함.

## 3. 로컬 서버에 붙일 수 있는 코딩 에이전트

로컬 vLLM/SGLang 서버는 `http://localhost:8000/v1` 같은 OpenAI 호환 엔드포인트를 노출하므로, 아래 에이전트들은 baseUrl만 바꿔주면 연결된다.

- **OpenHands** (구 OpenDevin): 자율 코딩 에이전트, Ollama/vLLM/SGLang 등 다양한 백엔드 지원. 첫 로컬 모델로 Qwen3.6-35B-A3B 추천.
- **Continue.dev**: VS Code/JetBrains 확장. config에서 provider를 "openai"로 두고 baseUrl만 로컬 서버로 지정.
- **Cline, Roo Code**: VS Code 확장, 동일하게 vLLM API(`localhost:8000/v1`)에 연결 가능.
- **Aider**: CLI 기반 에이전트, `--openai-api-base` 옵션으로 로컬 엔드포인트 지정.
- **opencode**: 오픈소스 CLI 코딩 에이전트, vLLM과 엔드투엔드로 결합해 완전 자체 호스팅 파이프라인 구성 가능.

## 4. Claude Code 자체를 로컬 GPU 모델과 함께 쓰고 싶은 경우

Claude Code는 Anthropic API 포맷(Messages API)으로 요청을 보내므로, 로컬 모델(OpenAI 포맷)과 바로 연결되지 않는다. 방법은 두 가지:

1. **번역 프록시 사용**: `LiteLLM`, `claude-code-proxy`, `UniClaudeProxy` 같은 오픈소스 프록시가 Anthropic ↔ OpenAI 포맷을 변환. `ANTHROPIC_BASE_URL`을 프록시 주소로 지정하면 된다.
2. **네이티브 호환 엔드포인트 이용**: 2026년 1월 Ollama v0.14.0부터 `/api/messages`라는 Anthropic 호환 엔드포인트가 추가되어 프록시 없이 `ANTHROPIC_BASE_URL`을 Ollama로 직접 지정 가능. LM Studio 0.4.1도 동일하게 `/v1/messages`를 지원.

## 요약 추천 조합

- **간단하게 시작**: Ollama(또는 LM Studio) + Qwen3.6/Devstral + Continue.dev 또는 Cline
- **본격적인 에이전틱 코딩**: vLLM 또는 SGLang + DeepSeek-V4-Flash/GLM-5.2 + OpenHands 또는 opencode
- **Claude Code UX를 유지하고 싶은 경우**: Ollama v0.14+/LM Studio 0.4.1 네이티브 엔드포인트 또는 LiteLLM 프록시 + 위 오픈소스 모델

## 참고 출처

- [Best Open Source Self-Hosted LLMs for Coding in 2026 - Pinggy](https://pinggy.io/blog/best_open_source_self_hosted_llms_for_coding/)
- [Best Open-Source & Open-Weight Coding Models (2026)](https://kilo.ai/open-source-models)
- [Build a Self-Hosted OpenAI-Compatible API with vLLM in 2026 | Spheron Blog](https://www.spheron.network/blog/openai-compatible-api-self-hosted/)
- [Coding Agent with Self-hosted LLM: Opencode and vLLM](https://cefboud.com/posts/coding-agent-self-hosted-llm-opencode-vllm/)
- [vLLM Complete Setup Guide for Local LLMs (2026)](https://localaimaster.com/blog/vllm-complete-setup-guide)
- [The Best Open Source LLMs (2026): Ranked - Morph](https://www.morphllm.com/best-open-source-llm)
- [Best Open-Source LLMs (Updated July 2026)](https://acecloud.ai/blog/best-open-source-llms/)
- [Best Open-Source LLMs: July 2026 Leaderboard](https://techsy.io/en/blog/best-open-source-llms-2026)
- [How to Self-Host a Model - Continue.dev Docs](https://docs.continue.dev/guides/how-to-self-host-a-model)
- [vLLM | Continue - Docs](https://docs.continue.dev/customize/model-providers/more/vllm)
- [Local LLMs - OpenHands Docs](https://docs.openhands.dev/openhands/usage/llms/local-llms)
- [Configure Local LLM with opencode](https://dev.to/tobrun_vannuland_70632c7/configure-local-llm-with-opencode-1gdb)
- [Use a Different LLM (Custom Model) with Claude Code | Morph](https://www.morphllm.com/use-different-llm-claude-code)
- [GitHub - fuergaosi233/claude-code-proxy](https://github.com/fuergaosi233/claude-code-proxy)
- [GitHub - vibheksoni/UniClaudeProxy](https://github.com/vibheksoni/UniClaudeProxy)
- [Claude Code + Local LLMs: Proxy Guide - Medium](https://medium.com/@michael.hannecke/connecting-claude-code-to-local-llms-two-practical-approaches-faa07f474b0f)
- [Use your LM Studio Models in Claude Code | LM Studio Blog](https://lmstudio.ai/blog/claudecode)
