# Context Window Guardrails (Simplified)

이 문서는 Codex가 “컨텍스트 창을 거의 다 썼다”라는 경고를 어떻게 계산하는지, 그리고 한도 직전에 어떤 조치를 취하는지를 단순화한 요약이다. 모든 수치는 기존 코드에서 그대로 가져왔다.

## 고정 상수

- **유효 컨텍스트 창 비율 95 %** – `ModelFamily` 기본값 (`core/src/model_family.rs:87`). 모델에서 공지한 전체 창(`context_window`)에 95 %를 곱해 “실제로 입력에 쓸 수 있는” 마지노선을 만든다. 예: GPT‑4.1의 창은 `1_047_576` 토큰(`core/src/openai_model_info.rs:25`), 따라서 Codex는 `994_197` 토큰(= `1_047_576 * 95 / 100`)까지만 입력 데이터에 사용한다. 나머지 5 %는 시스템 프롬프트, 툴 오버헤드, 모델 출력에 대비한 헤드룸이다.
- **자동 요약 트리거 90 %** – `ModelInfo::default_auto_compact_limit`이 `context_window * 9 / 10`으로 정의되어 있다 (`core/src/openai_model_info.rs:30-35`). 즉, 모델이 “이번 턴에서 사용한 총 토큰”을 보고했을 때 90 % 이상이면 즉시 요약(compaction) 플로우를 돌린다.
- **툴 출력 포맷 제한** – 모델에 재전송되는 각 함수/툴 출력은 `10 KiB` (`MODEL_FORMAT_MAX_BYTES`, `core/src/context_manager/truncate.rs:7`) 또는 `256`줄 (`MODEL_FORMAT_MAX_LINES`, 같은 파일)까지만 허용한다. 히스토리에 넣기 직전에 1.1배 버퍼(`CONTEXT_WINDOW_HARD_LIMIT_FACTOR`, `core/src/context_manager/history.rs:13-18`)를 적용해 최대 `11 264`바이트, `281`줄까지만 유지한다.
- **중앙 절단 마커** – `truncate_middle` (`core/src/truncate.rs`)는 길이가 너무 큰 문자열을 앞/뒤 절반만 남기고 가운데에 `…N tokens truncated…` 마커를 삽입한다. 토큰 수는 로컬 토크나이저(`codex_utils_tokenizer`)로 계산한다.

## 단순화한 히스토리 알고리즘

```text
effective_window = floor(model_context_window * 0.95)
auto_compact_limit = floor(model_context_window * 0.90)

build_turn():
  1. History snapshot = initial context + 모든 API 가시 아이템 (`ContextManager::get_history_for_prompt`)
  2. 각 tool output은 10KiB/256줄 한도로 포맷팅 (`format_output_for_model_body`)
  3. snapshot을 그대로 Responses API에 보냄
  4. turn 종료 후 모델이 보고한 TokenUsage.tokens_in_context_window를 체크
     - 사용량 ≥ auto_compact_limit → run_inline_auto_compact_task()

run_inline_auto_compact_task():
  1. 현재 History를 복사하고 Summarization Prompt를 히스토리 끝에 덧붙임
  2. Summarization Prompt가 95 % 창을 넘으면 가장 오래된 ResponseItem부터 하나씩 drop (`history.remove_first_item`)
  3. 모델이 요약을 완성하면:
     - 최신 assistant message를 summary_text로 저장
     - 과거 user message만 추린 뒤, Askama 템플릿으로 “history bridge” 생성
  4. History를 `[initial_context, bridge]`로 교체

turn_loop():
  while true:
    build_turn()
    stream model
    if tokens >= auto_compact_limit:
       if 이전 턴도 요약 시도였으면 에러 리턴
       else run_inline_auto_compact_task() 후 같은 턴을 다시 시도
    else break
```

위 의사코드는 `core/src/codex.rs:1866-1896`, `docs/conversation-compaction.md`, `core/src/codex/compact.rs` 흐름을 그대로 요약한 것이다. 핵심은 “모델에 요청을 보내기 전에는 히스토리를 가능하면 그대로 유지하되, 요청 직후 받은 token usage가 90 %를 넘으면 자동 요약으로 페이징한다”는 점이다.

## 왜 퍼센트가 들쑥날쑥해 보이나?

1. 히스토리가 매 턴 누적되므로 단순히 “이전 턴 출력”을 붙여도 95 % → 98 %처럼 즉시 변한다.
2. 명령 실행 로그는 `truncate_middle`로 잘라 붙이지만, diff/파일 결과가 몇 줄만 늘어나도 토큰이 수백 개 증가할 수 있다.
3. 요약이 성공하면 토큰 사용량이 크게 떨어진다. 이후 새 로그가 다시 붙으면 퍼센트가 재상승한다.

이 문서를 저장해 두면 “컨텍스트 창 경고가 어떻게 계산되는지”를 빠르게 복기할 수 있다.

