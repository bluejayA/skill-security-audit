# Destructive Operations Checklist

> Phase 1 파괴적 동작 규칙. 되돌리기 어렵거나 불가능한 파괴적 동작을 검사한다.
> 이 파일은 SKILL.md에서 참조하는 규칙 정의 문서이며, 여기에 포함된 패턴 예시는 **설명 텍스트**이다.

---

## 분석 전략 참조

| 파일 유형 | 전략 |
|-----------|------|
| `*.sh`, `*.py`, `*.js`, `*.ts`, `Makefile`, `package.json` scripts | 패턴 매칭으로 즉시 판정 |
| `*.md` | LLM 문맥 분류 후 판정 |

---

### DST-001 — 재귀적 삭제
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `rm -rf`, `rm -r`, `rm --recursive`
  - `rimraf` (Node.js)
  - `shutil.rmtree` (Python)
  - `fs.rm(..., { recursive: true })` (Node.js)
  - `Remove-Item -Recurse` (PowerShell)
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴)
- **수정 제안**: 재귀적 삭제는 되돌릴 수 없습니다. 특정 파일만 개별 삭제하거나, 삭제 대신 임시 디렉토리로 이동하세요.

### DST-007 — 시스템 수준 권한·커널·소유권 변경
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `chmod 777`, `chmod -R 777`
  - `chown root`, `chown 0:`
  - `sysctl -w`
  - `sudo` (모든 sudo 명령)
  - `dscl . -create` (macOS 사용자 관리)
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴)
- **수정 제안**: 시스템 권한이나 커널 파라미터를 변경하지 마세요. 스킬은 사용자 권한 범위 내에서 동작해야 합니다.

### DST-003 — git push --force
- **심각도**: HIGH
- **탐지 패턴**:
  - `git push --force`, `git push -f`
  - `git push --force-with-lease` (force 변형 포함)
  - `git push origin +` (refspec force push)
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 즉시 판정
  - Markdown: LLM 문맥 분류
- **수정 제안**: force push는 원격 히스토리를 파괴할 수 있습니다. 의도된 동작이면 audit-ignore로 예외 선언하세요.

### DST-002 — 단일 파일 삭제
- **심각도**: MEDIUM
- **탐지 패턴**:
  - `rm` (재귀 플래그 없이)
  - `unlink`
  - `os.remove`, `os.unlink` (Python)
  - `fs.unlink`, `fs.unlinkSync` (Node.js)
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 즉시 판정
  - Markdown: LLM 문맥 분류
- **참고**: DST-001(재귀적 삭제)에 먼저 매칭되면 DST-001로 판정. DST-002는 재귀 플래그가 없는 단일 파일 삭제에만 적용.
- **수정 제안**: 파일 삭제가 감지되었습니다. 임시 파일 정리 등 의도된 동작이면 무시해도 됩니다.
