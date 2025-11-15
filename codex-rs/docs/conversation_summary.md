# Conversation Q&A Summary

이 문서는 2025-XX-XX에 진행된 질의응답 전체를 요약한다. 아래 순서는 실제 대화 흐름을 반영하며, 각 항목은 핵심 결론과 관련 파일/코드 위치를 함께 기록했다.

## 1. 클라이언트 LLM 호스트 설계 계획

- Codex Rust 워크스페이스 구조를 조사한 뒤, Codex 스타일의 클라이언트 LLM 호스트를 만들기 위한 단계별 계획서를 작성했다 (`docs/client_llm_host_plan.md`).
- 계획은 워크스페이스 스냅샷, 데이터/컨트롤 플로우, Phase 0~6(기초 → 대화 엔진 → 샌드박스 → UI → 통합 → 배포 → 하드닝)로 구성했다.

## 2. 컨텍스트 사용량/토큰 계산

- “컨텍스트 97 %”는 모델 컨텍스트 창 중 현재 turn에서 사용 중인 토큰 비율을 의미하며, Codex가 로컬 토크나이저(`utils/tokenizer`)로 계산한 값이다. 서버가 사전 계산해 주는 것이 아니다.
- 토큰 증감 이유: 히스토리 누적, 추가 로그/툴 출력, 모드별 지시문 차이, UI 갱신 타이밍 등. turn마다 히스토리를 재조립하므로 1–2 %씩 달라질 수 있다.
- Codex는 `TokenUsageInfo`를 유지하면서 모델이 보고한 사용량을 합산하고, UI 종료 시 `Token usage: total=… input=… (+ cached)…` 형태로 출력한다. `cached` 값은 Responses API가 돌려주는 `input_tokens_details.cached_tokens` 필드 그대로다 (`core/src/client.rs:582-608`, `protocol/src/protocol.rs:705-915`).

## 3. 토크나이저·라이선스 관련

- Codex 내장 토크나이저(`utils/tokenizer`)는 순수 Rust 구현이므로 로컬에서 사용 가능하며, truncate·context 계산에 그대로 쓰인다.
- Codex 전체가 MIT/Apache-2.0 듀얼 라이선스이므로 truncate/tokenizer 코드를 비공개 레포에서 재사용할 때도 LICENSE/NOTICE만 포함하면 된다. 외부 배포 시에는 라이선스 고지를 동봉해야 한다.

## 4. Truncate & Conversation Memory 흐름

- `ContextManager::record_items`가 ResponseItem을 저장할 때 툴/함수 출력은 `format_output_for_model_body` → `truncate_middle`를 통해 즉시 10 KiB/256줄 제한(head/tail + ellipsis)으로 잘린 사본을 히스토리에 넣는다 (`core/src/context_manager/history.rs:40-91`, `core/src/context_manager/truncate.rs`, `core/src/truncate.rs`).
- 스트리밍 중에는 UI와 모델이 **전체 출력**을 실시간으로 받는다. 히스토리에 저장될 때만 truncate가 적용된다.
- 전체 히스토리 압박은 토큰 사용량이 모델 창의 90 %(`auto_compact_limit`)를 넘을 때 `run_inline_auto_compact_task`가 실행되어 요약/브리지로 교체한다 (`docs/conversation-compaction.md`, `core/src/codex.rs:1828-1896`). 이를 요약한 문서를 `docs/context-window-guardrails.md`에 정리했다.

## 5. Streaming 툴 호출 프로토콜

- Responses API 스트림 한 턴 안에서 모델 ↔ Codex ↔ UI가 동일한 SSE 흐름을 공유한다. 모델이 `read_file`을 요청하면 Codex가 즉시 파일을 읽어 전체 내용을 같은 스트림으로 다시 보낸다 (`core/src/tools/handlers/read_file.rs`).
- 히스토리에 저장된 truncate 사본과 무관하게, 모델은 필요하면 다음 턴에서 read_file을 다시 호출해 전체 파일을 재요청할 수 있다.

## 6. 세션/히스토리 관리 요약

- `get_pending_input()`는 아직 turn에 반영되지 않은 사용자 입력 큐만 담는다. 툴 출력은 `process_items` 경로로 즉시 히스토리에 들어가므로 별도로 큐에 쌓이지 않는다.
- 히스토리는 `Session`이 들고 있는 `ContextManager.items: Vec<ResponseItem>`에 저장되며, truncate된 버전만 남는다. 동시에 롤아웃 로그에는 전체 스트림이 기록된다.
- turn 종료 시 `TokenUsageInfo`가 업데이트되고, 필요하면 auto-compaction이 실행된다. UI 종료 메시지는 `codex resume <session-id>` 안내와 함께 누적 토큰 사용량을 보여 준다.

## 7. 메신저형 AI Agent 앱으로 응용 시 고려사항

- Codex-core는 모델 선택/Provider 의존성을 헐겁게 유지하므로, 필요한 모듈(히스토리, truncate, tokenizer, compaction)만 포크하거나 전체 core를 라이브러리로 사용하고 새로운 UI(메신저 기반)를 입히는 방식이 현실적이다.
- `codex-tui` 자체를 변형하기보다는 core 로직을 재사용하고, 메신저 UX에 맞는 프론트엔드를 따로 만드는 편이 작업량 대비 효율이 높다.

이 요약은 대화 전체를 빠르게 복기하기 위한 문서로, 추가 논의가 생기면 항목을 이어서 업데이트한다.
