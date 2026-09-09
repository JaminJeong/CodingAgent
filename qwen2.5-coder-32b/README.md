# Qwen2.5-Coder-32B-Instruct — 파이프라인 검증용

## 목적

Qwen3-Coder-480B(595GB+ 다운로드, 8x H100 전체 필요)를 바로 시도하기 전에,
`docker-compose → vLLM → OpenAI 호환 API` 전체 흐름이 이 서버에서 정상 동작하는지
가볍고 빠른 모델로 먼저 검증하기 위한 구성.

## 구성

- 모델: `Qwen/Qwen2.5-Coder-32B-Instruct` (공개, 비양자화 BF16, 다운로드 ~65GB)
- GPU: 2장(tensor-parallel-size 2)
- 포트: 8123
- 상세 설정은 [`./docker-compose.yml`](./docker-compose.yml) 참고

## 검증 결과

- `curl http://localhost:<HOST_PORT>/v1/models` → `qwen-coder-32b` 정상 등록
- `curl http://localhost:<HOST_PORT>/v1/chat/completions` → 피보나치 메모이제이션 함수 코드 정상 생성

이 검증을 통과한 뒤 본 작업 대상인 [`../qwen3-coder-480b`](../qwen3-coder-480b/README.md)로 진행했다.

## 실행

```bash
cp .env.example .env   # 필요 시 값 수정
docker compose up -d
```

## 현재 상태

검증 완료 후 컨테이너 상태 미확인 — 재사용 전 `docker ps -a`로 확인 필요.
