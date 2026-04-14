# Operational Feedback Log

skill-security-audit 운영 중 관찰된 실제 사례를 누적 기록하는 로그입니다. 다음 리뷰 라운드의 1차 입력 자료로 사용됩니다.

문서가 목적이 아니라 **데이터 파이프라인**이 목적입니다. 완벽한 기록보다 빠른 기록이 우선입니다.

---

## 기록 대상

| 카테고리 | 정의 | 예시 |
|---|---|---|
| **FP** (False Positive) | BLOCK/WARN 판정이지만 실제로는 안전했던 케이스 | ` curl \| bash`가 README 안의 문자열로 인용되어 SEC-003이 오탐 |
| **FN** (False Negative) | 통과시켰으나 사후에 위험이 드러난 케이스 | zero-width unicode로 난독화한 exfil 코드가 META-002를 피해 PASS |
| **Bypass** | 규칙을 의도적으로 우회한 시도 (성공/실패 무관) | 런타임 `eval(base64.decode(...))` 구성으로 SEC-041 우회 |
| **Friction** | 제출자/리뷰어 불만, 오해, 프로세스 마찰 | audit-ignore 작성법이 모호하다는 피드백 |
| **Near-miss** | 거의 놓칠 뻔했거나 인접 규칙이 우연히 잡은 케이스 | SEC-001이 놓쳤으나 DST-007이 부수적으로 BLOCK |

---

## 기록 포맷

1줄이면 충분합니다. PR/issue 링크가 있으면 같이 둡니다.

```
YYYY-MM-DD | CATEGORY | RULE-ID | 1줄 설명 | 링크
```

복잡한 분석은 별도 issue로 추출. 이 파일은 **수집 layer**만 담당합니다.

---

## Log

<!-- 새 엔트리는 이 주석 바로 아래에 추가 -->

_(아직 비어있음. 첫 사례 기록 시 이 섹션이 시작됨)_

---

## Phase 2 종합 리뷰 진입 heuristic

다음 중 **하나라도** 충족되면 종합 리뷰(아키텍처/거버넌스 재검토) 고려:

- FP가 분기당 **5건 이상** 누적 → adoption 저해
- FN **1건이라도** 기록 → 규칙 커버리지 재검토
- Bypass **3건 이상** 기록 → 탐지 메커니즘의 구조적 약점
- Friction **2건 이상 반복** 발생 → 프로세스/문서 개선 필요

heuristic이지 gate는 아닙니다. 맥락상 필요하면 기준 이전에도 리뷰 가능.

---

## 관련 문서

- [pending-requirements.md](../devflow-docs/pending-requirements.md) — 운영 중 들어온 요구사항 수집
- [ci-integration-guide.md](ci-integration-guide.md) — 감사 워크플로우 구조
