# PRD — BBG Bridge (가칭) v1.0

> 338 · 2026-10-14 · 작성: Claude Code(Opus 5.5) · 상태: 설계안(코드 없음)
> 터미널 PC에서만 도는 **개인 연구용** Bloomberg × Claude 도우미

## 1. 배경과 문제
- 학생 질문: Bloomberg 데이터를 AI(Claude)로 분석하고 싶은데, Bloomberg가 안내한 AI 연결(MCP) 상품은 비싸다. Claude API를 내려받아 로컬 네트워크로 MCP 없이 붙일 수 있나?
- 사실 확인(2026-10-14 웹 조사):
  - Bloomberg는 **2026-09-29 Enterprise MCP**를 발표했다. Data License Plus(DL+)용 AI 접근 계층이며, 기업 데이터 라이선스로 판매한다(터미널 부가 상품이 아님). [S1][S2]
  - **터미널 구독자용 MCP는 "출시 예정"**으로만 발표됐다(날짜 미발표). [S1][S2]
  - Desktop API(BLPAPI)는 **같은 PC에서 터미널이 실행·로그인된 상태**여야 동작하고(localhost:8194), Bloomberg 공식 Python 패키지로 설치한다. macOS ARM·Linux 라이브러리는 터미널 사용자용이 아니다 → **Windows 터미널 PC 전제**. [S3][S4]
  - 대학 도서관 안내 예시: Excel API 데이터는 내려받은 터미널에서만 사용, 원자료·파생자료 모두 반출 금지. [S5] → **AI 외부 전송 허용 여부는 계약서 확인 전까지 미정**.
- 결론: 인증 우회는 불가·금지. 대신 **터미널 구독에 딸린 Desktop API + Claude tool use(또는 직접 만든 로컬 MCP 서버)**로 만들 수 있으나, Live 사용은 약관 확인이 선행 조건.

## 2. 목표 / 비목표
**목표**
1. 터미널 PC에서 자연어 질문 → Claude가 필요한 데이터를 도구로 요청 → BLPAPI 조회 → 분석 답변
2. 약관 위험을 줄이는 **정책 가드**를 기본 탑재(한도·허용 필드·원자료 미저장·집계만 전송)
3. 터미널 없이 개발·시연 가능한 **Mock 모드**(가상 데이터, 화면에 항상 MOCK 표시)

**비목표**: 인증·라이선스 우회, 다중 사용자 서버, 데이터 재배포·공유, 자동 매매, 투자 조언

## 3. 사용자
| 페르소나 | 설명 | 핵심 요구 |
|---|---|---|
| 학생(주 사용자) | 경영·경제 전공, Python 초급, 학교 터미널 사용 | 쉬운 설치, 자연어 분석, 규칙 위반 걱정 없음 |
| 지도교수(점검자) | 사용 범위 확인 | 무엇이 외부로 나갔는지 기록 열람 |

## 4. 기능 요구 (MVP)
| ID | 기능 | 요구 사항 | 우선 |
|---|---|---|---|
| F1 | 대화형 분석 | 채팅 입력 → Claude Messages API tool use 루프(최대 반복 N회) → 답변·도구 호출 내역 표시 | P0 |
| F2 | 데이터 도구 4종 | `get_reference`(BDP 현재값), `get_history`(BDH 시계열), `get_bulk`(BDS 목록형), `search_ticker`(종목 검색) — 입력 JSON 스키마 고정 | P0 |
| F3 | 정책 가드 | 1회 최대 종목 수·기간·필드 수, 허용 필드 목록, 일일 조회 한도, "집계만 전송"(평균·변동성·수익률 등 요약만 Claude로) — 위반 시 차단하고 이유를 Claude·사용자에게 반환 | P0 |
| F4 | 모드 전환 | Live(BLPAPI) / Mock(결정적 가상 데이터, 시드 고정) — 같은 인터페이스(`DataSource`) | P0 |
| F5 | 감사 로그 | 도구·종목·필드·기간·행 수·전송 여부·차단 사유 기록. **시세 값은 저장하지 않음** | P0 |
| F6 | 관심종목·보고서 | 관심종목 목록, 대화 결과를 Markdown 보고서로 저장(기본: 원자료 표 제외) | P1 |
| F7 | MCP 서버 모드 | 같은 도구·가드를 로컬 stdio MCP 서버로 노출 → Claude Desktop/Claude Code에서 사용 | P1 |
| F8 | 데이터 탐색기 | Claude 없이 수동 조회 → 표/차트(외부 전송 없음) | P2 |

## 5. 비기능 요구
- 로컬 전용: 웹 UI는 127.0.0.1 바인딩, 외부 포트 개방 금지
- 비밀: Anthropic API 키는 `.env`(git 제외) 또는 OS 키체인
- 환경: Windows 터미널 PC, Python 3.11~3.12(BLPAPI 지원 범위), 단일 실행 명령(`bbg-bridge run`)
- 저장: SQLite 단일 파일, 시세 값 미저장, 메모리 캐시 TTL ≤ 5분
- 비용 표시: 대화별 입력·출력 토큰 수

## 6. 시스템 구성
```
[브라우저 UI] ⇄ [FastAPI 앱 (127.0.0.1)]
                    ├─ Chat Orchestrator ── Anthropic Messages API (tool use)
                    ├─ Tool Layer (4 tools) ── Policy Guard ── DataSource
                    │                                          ├─ LiveSource: blpapi → 터미널(localhost:8194)
                    │                                          └─ MockSource: 시드 고정 가상 데이터
                    ├─ Audit Logger ── SQLite
                    └─ (옵션) MCP Server(stdio) ── 같은 Tool Layer 재사용
```
핵심: 도구·가드·데이터원을 한 계층으로 두고, 웹 채팅과 MCP 서버가 **같은 코드**를 쓴다.

## 7. 기술 스택
Python · `blpapi`(Bloomberg 공식 인덱스) 또는 `xbbg` · `anthropic` SDK · `mcp` Python SDK · FastAPI + Jinja/HTMX(또는 Streamlit, PI 결정) · SQLite · pytest

## 8. 성공 기준
- Mock 모드 질문 시나리오 10개 전부 통과(예: "삼성전자와 SK하이닉스 1년 수익률·변동성 비교")
- 정책 위반 요청 테스트 100% 차단(한도 초과·금지 필드·과다 기간)
- 감사 로그에 시세 값이 0건 저장됨을 테스트로 확인
- 터미널 PC에서 Live 스모크 테스트 1회 통과(학생 수행, **약관 확인 후**)

## 9. 리스크
| 리스크 | 영향 | 대응 |
|---|---|---|
| 약관상 외부 AI 전송 불가 | Live 사용 불가 | 학교 도서관·Bloomberg 담당자 서면 확인 선행. 불가 시 "집계만 전송" 또는 Mock·공개데이터 전용으로 운영 |
| 학교 터미널에 프로그램 설치 제한 | 설치 불가 | 전산 담당 확인, 휴대용 Python(설치 없는 방식) 검토 |
| Bloomberg 공식 터미널용 MCP 출시 | 자체 개발 가치 감소 | 출시 시 그쪽으로 전환, 본 프로젝트는 학습용으로 유지 |
| 커뮤니티 Bloomberg MCP 사용 유혹 | 유지보수·약관 불명 | 참고만, 코드 검토 없이 쓰지 않음 |
| Claude가 잘못된 수치 해석 | 오분석 | 도구 결과 원표를 화면에 함께 표시, 투자 조언 금지 시스템 프롬프트 |

## 10. 마일스톤
| M | 범위 | 완료 조건 |
|---|---|---|
| M1 | MockSource + 도구 4종 + tool use 루프(CLI) | 시나리오 10개 통과 |
| M2 | 정책 가드 + 감사 로그 + SQLite | 위반 차단 테스트 통과 |
| M3 | 웹 UI(채팅·보고서·로그·설정) | IA 6화면 동작 |
| M4 | MCP 서버 모드 | Claude Desktop에서 도구 호출 확인 |
| M5 | LiveSource + 학생 스모크 테스트 | 약관 확인서 + 1회 통과 |

## 출처 (Sources)
- [S1] Bloomberg Enterprise MCP 발표 기사 — https://cyprusshippingnews.com/2026/10/06/bloomberg-launches-enterprise-mcp-to-seamlessly-connect-bloomberg-data-with-clients-enterprise-ai-applications
- [S2] Bloomberg Enterprise MCP 해설 — https://mcpplaygroundonline.com/blog/bloomberg-enterprise-mcp
- [S3] Bloomberg API Library — https://www.bloomberg.com/professional/support/api-library
- [S4] Fetching data from Bloomberg (Desktop API는 터미널 로그인 필요) — https://merton.readthedocs.io/en/latest/cookbook/bloomberg.html
- [S5] Harvard Business School Library, Bloomberg 이용 안내 — https://www.library.hbs.edu/Find/Databases/Bloomberg
- [S6] 커뮤니티 Bloomberg MCP(비공식) — https://mcp.so/server/bloomberg-mcp

> 면책: 약관 해석은 법률 자문이 아니다. 실제 허용 범위는 학교와 Bloomberg 계약서·담당자 확인을 따른다.
