# Pending Requirements

운영 중 들어온 요구사항을 **action 이관 전**에 수집하는 저장소. 아이디어 박스가 아니라 "언젠가 처리할 약속 목록"입니다. 방치되면 stale 데이터가 되므로, 주기적으로(예: 월 1회) 상태를 점검합니다.

**이 파일의 역할:**
- incoming 요구사항을 잊지 않게 기록
- 실제 action으로 이관되면 PR/issue로 graduate
- 검토 후 하지 않기로 한 건은 사유를 남기고 declined로 이동

---

## Incoming

| 날짜 | 출처 | 요약 | 우선순위 | 메모 |
|---|---|---|---|---|
| 2026-04-14 | 내부 세션 | 룰셋 버전 업그레이드 시 기존 플러그인 자동 재감사 트리거 | P1 | v1.5→v2.0 전환처럼 major bump 시 수동 재감사 필요. 워크플로우 추가 검토 |
| 2026-04-14 | 내부 세션 | marketplace spec에 "shared code는 `skills/_*/` 하위" 컨벤션 명시 | P2 | diff-scoping fallback이 컨벤션에 의존. 문서 PR 1건 필요 |
| 2026-04-14 | 내부 세션 | `sync-marketplace.yml` — skill-security-audit 버전 bump 시 marketplace.json 자동 갱신 | P2 | 현재 수동. 운영 편의성 향상 항목 |

---

## Graduated (→ PR / Issue로 이관)

| 날짜 | 요약 | 이관처 |
|---|---|---|
| _(비어있음)_ | | |

---

## Declined

| 날짜 | 요약 | 거절 사유 |
|---|---|---|
| _(비어있음)_ | | |

---

## 상태 정의

- `incoming` — 수집 완료, 구현 여부 결정 전
- `graduated` — PR/issue 번호가 부여되어 별도 트래킹으로 이관
- `declined` — 검토 후 채택하지 않기로 결정. **사유 기록 필수**

## 우선순위 정의

- `P0` — 보안 직결 / 현재 차단 중 / 데이터 손실 리스크
- `P1` — 운영에 지속 영향 / 거버넌스 / 확장성
- `P2` — 개선 / 사용성
- `P3` — **operate-and-iterate** (운영 데이터 쌓인 후 재평가 대상)

---

## 관련 문서

- [operational-feedback.md](../docs/operational-feedback.md) — 관찰된 실제 사례 로그 (FP/FN/Bypass 등)
