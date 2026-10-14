# ERD — BBG Bridge (가칭) v1.0

> 338 · 2026-10-14 · SQLite 단일 파일 `bbg_bridge.db`
> **원칙: 조회한 시세 값 자체는 저장하지 않는다.** 요청 메타·행 수·전송 여부만 남긴다. 캐시는 메모리·TTL ≤ 5분.

```mermaid
erDiagram
  SESSION ||--o{ MESSAGE : has
  MESSAGE ||--o{ TOOL_CALL : triggers
  TOOL_CALL ||--o| DATA_REQUEST : executes
  TOOL_CALL ||--o{ AUDIT_LOG : records
  POLICY_RULE ||--o{ TOOL_CALL : blocks
  SESSION ||--o{ REPORT : produces
  WATCHLIST ||--o{ WATCHLIST_ITEM : contains

  SESSION {
    int id PK
    text title
    text mode "LIVE or MOCK"
    text model
    datetime created_at
  }
  MESSAGE {
    int id PK
    int session_id FK
    text role "user, assistant, tool"
    text content
    int tokens_in
    int tokens_out
    datetime created_at
  }
  TOOL_CALL {
    int id PK
    int message_id FK
    text tool_use_id "Claude tool_use id"
    text tool_name
    text args_json
    text status "ok, blocked, error"
    int blocked_by_rule_id FK
    text blocked_reason
    int latency_ms
  }
  DATA_REQUEST {
    int id PK
    int tool_call_id FK "NULL if explorer"
    text tickers_json
    text fields_json
    date start_date
    date end_date
    int row_count
    text source "LIVE or MOCK"
    bool sent_to_llm
    text payload_shape "raw or aggregate"
    datetime created_at
  }
  POLICY_RULE {
    int id PK
    text kind "max_tickers, max_days, allowed_field, daily_quota, aggregate_only"
    text value
    bool active
  }
  AUDIT_LOG {
    int id PK
    int tool_call_id FK
    text event "requested, blocked, fetched, sent"
    text detail
    datetime at
  }
  REPORT {
    int id PK
    int session_id FK
    text title
    text body_md
    bool includes_raw
    datetime created_at
  }
  WATCHLIST {
    int id PK
    text name
  }
  WATCHLIST_ITEM {
    int id PK
    int watchlist_id FK
    text ticker
    text note
  }
  SETTING {
    text key PK
    text value
  }
```

## 테이블 설명
| 테이블 | 용도 | 비고 |
|---|---|---|
| SESSION | 대화 1건 | 모드·모델을 대화 시작 시 고정 |
| MESSAGE | 대화 메시지 | role=tool은 요약만(원표 대신 "N행 전송" 문구) |
| TOOL_CALL | Claude가 요청한 도구 호출 | 차단이면 어떤 규칙이 막았는지 연결 |
| DATA_REQUEST | 실제 조회 메타 | 탐색기 수동 조회는 tool_call_id NULL·sent_to_llm=false |
| POLICY_RULE | 정책 가드 규칙 | 설정 화면에서 편집 |
| AUDIT_LOG | 사건 이력 | 교수 점검용, 삭제 UI 없음 |
| REPORT | Markdown 보고서 | includes_raw 기본 false |
| WATCHLIST(_ITEM) | 관심종목 | |
| SETTING | 키-값 설정 | API 키는 저장하지 않음(.env/키체인) |

## 기본 정책 값(초안, PI 조정)
| kind | value |
|---|---|
| max_tickers | 10 |
| max_days | 1825 (5년) |
| allowed_field | PX_LAST, PX_VOLUME, CUR_MKT_CAP, PE_RATIO, BEST_EPS 등 허용 목록 |
| daily_quota | 200 요청/일 |
| aggregate_only | true (Live 기본값) |
