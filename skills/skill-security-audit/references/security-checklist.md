# Security Checklist

> Phase 1+2 보안 규칙. 자격증명 노출, 원격 코드 실행, 셸 인젝션, 민감 경로 접근, 네트워크 접근 + OWASP AST10 확장 규칙을 검사한다.
> 이 파일은 SKILL.md에서 참조하는 규칙 정의 문서이며, 여기에 포함된 패턴 예시는 **설명 텍스트**이다.

---

## 분석 전략 참조

| 파일 유형 | 전략 |
|-----------|------|
| `*.sh`, `*.py`, `*.js`, `*.ts`, `Makefile`, `package.json` scripts | 패턴 매칭으로 즉시 판정 |
| `*.md` (SKILL.md, references/) | LLM 문맥 분류 후 판정 (directive → 규칙 적용, descriptive → 탐지 안 함, 애매 → MEDIUM) |
| `*.json`, `*.yaml` (assets/) | 패턴 매칭 (자격증명 중심) |

---

## 1. 자격증명 노출 (Credential Exposure)

### SEC-010 — 하드코딩된 API 키
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `ghp_[A-Za-z0-9]{36}` — GitHub Personal Access Token
  - `sk-[A-Za-z0-9]{20,}` — OpenAI / Anthropic API Key
  - `xoxb-[0-9]{10,}` — Slack Bot Token
  - `xoxp-[0-9]{10,}` — Slack User Token
  - `AKIA[0-9A-Z]{16}` — AWS Access Key ID
  - `AIza[0-9A-Za-z\-_]{35}` — Google API Key
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴 — Markdown 문맥 분류 불필요)
- **수정 제안**: 자격증명을 파일에 직접 포함하지 마세요. 환경변수 또는 시크릿 매니저를 사용하세요.

### SEC-011 — 프라이빗 키
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `BEGIN PRIVATE KEY`
  - `BEGIN RSA PRIVATE KEY`
  - `BEGIN EC PRIVATE KEY`
  - `BEGIN DSA PRIVATE KEY`
  - `BEGIN OPENSSH PRIVATE KEY`
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴)
- **수정 제안**: 프라이빗 키를 저장소에 포함하지 마세요. 시크릿 매니저 또는 키 볼트를 사용하세요.

### SEC-013 — process.env 전체 덤프 또는 외부 전송
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `process.env` 전체 참조 + 외부 전송 함수 조합 (예: `fetch(url, { body: JSON.stringify(process.env) })`)
  - `os.environ` 전체 덤프 + 외부 전송
  - `ENV` 전체를 `curl`, `WebFetch`, `http.post` 등에 전달
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 (process.env/os.environ + 네트워크 함수 조합 탐지)
  - Markdown: LLM 문맥 분류 — directive이면 CRITICAL, 애매하면 MEDIUM
- **수정 제안**: 환경변수 전체를 외부로 전송하지 마세요. 필요한 변수만 명시적으로 참조하세요.

### SEC-012 — 민감 설정 파일 직접 참조
- **심각도**: MEDIUM
- **탐지 패턴**:
  - `.env` 파일 참조 (Read, cat 등으로 내용 읽기)
  - `credentials.json`, `token.json`, `client_secret_*.json` 참조
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 즉시 판정
  - Markdown: LLM 문맥 분류
- **수정 제안**: 민감 설정 파일을 직접 참조하는 대신, 환경변수를 통해 값을 전달하세요.

---

## 2. 원격 코드 실행 (Remote Code Execution)

### SEC-003 — 파이프를 통한 원격 실행
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `curl ... | bash`, `curl ... | sh`
  - `wget ... | bash`, `wget ... | sh`
  - `curl ... | python`, `curl ... | node`
  - 변형: `curl -s`, `curl -fsSL` 등 플래그 포함
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴)
- **수정 제안**: 원격 스크립트를 파이프로 직접 실행하지 마세요. 먼저 다운로드한 후 내용을 검토하고 실행하세요.

### SEC-030 — Base64 디코딩 후 실행
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `base64 -d | bash`, `base64 --decode | sh`
  - `base64 -d | python`, `base64 -d | node`
  - `echo ... | base64 -d | ...` 체인
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴)
- **수정 제안**: Base64로 인코딩된 내용을 디코딩하여 직접 실행하지 마세요. 코드를 평문으로 포함하세요.

---

## 3. 셸 인젝션 (Shell Injection)

### Trusted / Untrusted Input 정의

| 구분 | 예시 |
|------|------|
| **Untrusted** (검증 필요) | CLI argument, prompt input, PR 내용, repository content, 파일 읽기 결과, 환경변수(`process.env`, `$ENV`), 외부 API 응답 |
| **Trusted** (검증 불필요) | 코드 내부 상수(하드코딩된 값), allowlist로 제한된 값, 검증된 enum/whitelist 값 |

### SEC-001 — 셸 인젝션
- **심각도**: CRITICAL 또는 HIGH (조건부)
- **CRITICAL 조건** (하나라도 만족):
  - `sh -c`, `bash -c`, `subprocess(..., shell=True)` 등 셸 실행 경로에 untrusted input 직접 삽입
  - 입력 검증 / allowlist / escaping 없이 untrusted input으로 명령 문자열 직접 구성
  - untrusted input이 검증 없이 명령 실행에 연결
- **HIGH 조건**: 위 불충족 (단순 변수 할당, trusted input만 사용)
- **탐지 패턴**:
  - `bash -c "$VARIABLE"`, `sh -c "${input}"`
  - `subprocess(f"... {user_input} ...", shell=True)`
  - `exec("... " + variable + " ...")`
  - Bash tool에 변수 직접 삽입 지시
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 → trusted/untrusted 판단 → CRITICAL 또는 HIGH
  - Markdown: LLM 문맥 분류 → directive이면 규칙 적용, 애매하면 MEDIUM
- **수정 제안**: 사용자 입력을 셸 명령에 직접 삽입하지 마세요. 인자를 배열로 전달하거나, allowlist로 검증하세요.

### SEC-002 — 문자열 기반 동적 코드 실행
- **심각도**: HIGH
- **탐지 패턴**:
  - `eval(...)` (JavaScript, Python)
  - `exec(string)` (Python)
  - `Function(string)` (JavaScript)
  - `new Function(...)` (JavaScript)
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 즉시 HIGH
  - Markdown: LLM 문맥 분류 → directive이면 HIGH, 애매하면 MEDIUM
- **수정 제안**: 문자열 기반 동적 코드 실행은 보안 위험이 큽니다. 정적 함수 호출이나 설정 파일로 대체하세요.

---

## 4. 민감 경로 접근 (Sensitive Path Access)

### SBX-003 — 경로 탈출
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `../` 가 2회 이상 반복 (예: `../../etc/passwd`)
  - `..` + 시스템 경로 조합 (`../../../etc`, `../../../usr`)
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 즉시 판정
  - Markdown: LLM 문맥 분류
- **수정 제안**: 상대 경로로 상위 디렉토리를 탐색하지 마세요. 스킬 디렉토리 내부 파일만 참조하세요.

### SBX-004 — 홈 디렉토리 민감 경로 참조
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `~/.ssh`, `$HOME/.ssh`
  - `~/.aws`, `$HOME/.aws`
  - `~/.gnupg`, `$HOME/.gnupg`
  - `~/.netrc`, `$HOME/.netrc`
  - `~/.config/gh`, `$HOME/.config/gh`
  - `~/.kube/config`, `$HOME/.kube/config`
  - `~/.docker/config.json`, `$HOME/.docker/config.json`
  - `~/.npmrc`, `$HOME/.npmrc`
  - `~/.pypirc`, `$HOME/.pypirc`
  - `~/.config/gcloud`, `$HOME/.config/gcloud`
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴)
- **수정 제안**: 사용자의 SSH 키, AWS 자격증명, GPG 키 등 민감 디렉토리에 접근하지 마세요.

### SBX-007 — 키체인·히스토리 읽기
- **심각도**: CRITICAL
- **탐지 패턴**:
  - `~/.bash_history`, `~/.zsh_history`
  - `security find-generic-password` (macOS Keychain)
  - `security find-internet-password`
  - `~/.password-store`
  - `~/.local/share/keyrings`
- **파일 유형별 처리**: 모든 파일에서 패턴 매칭 즉시 판정 (고신뢰 패턴)
- **수정 제안**: 셸 히스토리나 키체인에서 자격증명을 읽지 마세요.

---

## 5. 네트워크 접근 (Network Access)

### SEC-020 — 외부 HTTP 요청 (기본)
- **심각도**: MEDIUM
- **탐지 패턴**:
  - `WebFetch`, `fetch(`, `axios`, `http.get`, `http.post`
  - `curl`, `wget` (파이프 없이 단독 사용)
  - `requests.get`, `requests.post` (Python)
  - `urllib`, `httpx`
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 즉시 판정
  - Markdown: LLM 문맥 분류
- **수정 제안**: 외부 HTTP 요청이 감지되었습니다. 스킬의 핵심 기능에 필요한 경우 의도된 동작으로 무시해도 됩니다.

### SEC-020H — 외부 HTTP 요청 + 민감 데이터 페이로드
- **심각도**: HIGH
- **탐지 패턴**: SEC-020 패턴 + 다음 페이로드 조합:
  - `process.env`, `os.environ` 참조를 요청 본문에 포함
  - API 키 변수를 헤더/본문에 포함
  - 파일 읽기 결과를 외부로 전송
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 조합 매칭
  - Markdown: LLM 문맥 분류 — 민감 데이터 + 외부 전송 지시 조합 탐지
- **수정 제안**: 자격증명이나 환경변수를 외부 서버로 전송하지 마세요. 필요하면 audit-ignore로 예외 선언하세요.

### SEC-022 — 직접 네트워크 도구
- **심각도**: HIGH
- **탐지 패턴** (단어 경계 매칭 필수 — `\b` 또는 독립 토큰으로 판정):
  - `ssh`, `scp`, `sftp` — 단어 경계 매칭. `sshd_config`, `ssh-keygen` 같은 복합어 내 출현은 문맥 확인 필요
  - `nc`, `netcat`, `ncat` — 단어 경계 매칭. `disconnect`, `once` 등 일반 단어 내 `nc`는 오탐이므로 독립 토큰만 탐지
  - `socat`
  - `telnet`
- **파일 유형별 처리**:
  - 구조화 코드: 패턴 매칭 즉시 판정 (단어 경계 적용)
  - Markdown: LLM 문맥 분류
- **수정 제안**: 직접 네트워크 도구 사용이 감지되었습니다. 의도된 동작이면 audit-ignore로 예외 선언하세요.

---

## 6. 외부 디렉토리 쓰기

### SBX-001 — 스킬 디렉토리 외부 파일 쓰기
- **심각도**: HIGH
- **탐지 패턴**:
  - Write/Edit tool에 스킬 디렉토리 외부 경로 지시
  - `>`, `>>` 리다이렉션으로 외부 경로에 쓰기
  - `cp`, `mv`로 외부 경로에 파일 복사/이동
  - `tee` 명령으로 외부 경로에 쓰기
- **파일 유형별 처리**:
  - 구조화 코드: 절대경로 + 쓰기 명령 조합 탐지
  - Markdown: LLM 문맥 분류 — 스킬 외부 경로에 쓰기 지시인지 판단
- **수정 제안**: 스킬 디렉토리 외부에 파일을 쓰지 마세요. 출력은 stdout 또는 스킬 디렉토리 내부로 제한하세요.

---

### OWASP AST10 확장 (v2)

> Phase 2에서 추가된 보안 규칙. OWASP Agentic Skills Top 10 기반.

### SEC-040 — 안전하지 않은 YAML 로더 (Unsafe YAML Loader)
- **심각도**: CRITICAL
- **OWASP**: AST05 5.1
- **범위**: scripts/ 내 모든 .py 파일, 모든 .yaml/.yml 파일
- **탐지 패턴**:
  - `yaml.load(` — `Loader=yaml.SafeLoader` 또는 `Loader=SafeLoader` 없이 사용
  - `yaml.unsafe_load(`
  - .yaml/.yml 파일 내 `!!python/object` 태그
  - .yaml/.yml 파일 내 `!!python/apply` 태그
- **파일 유형별 처리**: 패턴 매칭으로 즉시 판정
- **메시지**: "Unsafe YAML deserialization at {file}:{line}. Use yaml.safe_load() instead (OWASP AST05 5.1)."
- **수정 제안**: `yaml.load(data)` 를 `yaml.safe_load(data)` 로 교체하세요. `!!python/object`, `!!python/apply` 태그는 임의 코드 실행이 가능하므로 제거하세요.

### SEC-041 — 스크립트 내 위험한 코드 실행 (Dangerous Code Execution in Scripts)
- **심각도**: CRITICAL
- **OWASP**: AST01 1.4
- **범위**: scripts/ 내 모든 파일
- **탐지 패턴**:
  - Python: `eval(`, `exec(`, `compile(`, `os.system(`, `os.popen(`, `subprocess.call(.*shell=True`, `subprocess.Popen(.*shell=True`, `__import__(`, `importlib.import_module(`
  - JavaScript: `eval(`, `new Function(`, `child_process.exec(`, `child_process.execSync(`
  - Shell: `eval `, 알 수 없는 URL 소싱
- **파일 유형별 처리**: 패턴 매칭으로 즉시 판정 (기존 SEC-001의 scripts/ 디렉토리 확장)
- **우선순위**: `scripts/` 내 파일에서 SEC-041과 SEC-002가 동일 패턴(예: `eval(`)에 동시 매칭되면, **SEC-041이 우선 적용**되고 SEC-002는 중복 보고하지 않는다. SEC-041은 scripts/ 전용이므로 scripts/ 외부 파일에서는 SEC-002가 적용된다.
- **메시지**: "Dangerous code execution pattern [{pattern}] at {file}:{line} (OWASP AST01 1.4)."
- **수정 제안**: `eval()` 대신 명시적 파싱을 사용하세요. `subprocess.call(cmd, shell=True)` 대신 `subprocess.call(cmd_list)` 처럼 리스트를 전달하세요.

### SBX-010 — 무제한 셸 접근 (Wildcard Shell Access)
- **심각도**: HIGH
- **OWASP**: AST03 3.3
- **범위**: SKILL.md frontmatter `allowed-tools` 필드
- **탐지 패턴**:
  - `Bash(*:*)` 또는 `Bash(*)`
  - `Shell(*:*)` 또는 `Shell(*)`
  - 모든 도구 선언에서 무제한 `*` 와일드카드
- **파일 유형별 처리**: frontmatter 파싱 후 패턴 매칭
- **메시지**: "Unrestricted shell access declared in allowed-tools: [{value}]. Scope to specific commands (OWASP AST03 3.3)."
- **수정 제안**: `Bash(*:*)` 대신 범위가 한정된 선언을 사용하세요. 예: `Bash(git:*)`, `Bash(npm:run)`.

### SBX-011 — 바이너리 네트워크 권한 (Binary Network Permission)
- **심각도**: HIGH
- **OWASP**: AST03 3.7
- **범위**: SKILL.md frontmatter, 매니페스트 파일, SKILL.md body
- **탐지 패턴**:
  - `network: true` — 도메인 allowlist 없이 사용
  - SKILL.md body에서 "requires network access", "needs internet" — 특정 도메인 명시 없이
- **파일 유형별 처리**:
  - frontmatter: 키-값 파싱
  - SKILL.md body: 자연어 지시 탐지
- **메시지**: "Binary network permission without domain allowlist. Declare specific domains instead (OWASP AST03 3.7)."
- **수정 제안**: `network: true` 대신 도메인 allowlist를 metadata 또는 compatibility 필드에 선언하세요. 예: `compatibility: Requires network access to api.example.com`.

### SBX-012 — 광범위 파일 글로브 (Broad File Glob)
- **심각도**: HIGH
- **OWASP**: AST03 3.4
- **범위**: SKILL.md frontmatter `allowed-tools`, scripts/, SKILL.md body
- **탐지 패턴**:
  - `**/*` — 재귀 와일드카드
  - `~/*` 또는 `~/` — 홈 디렉토리
  - 루트 경로: `/etc/`, `/usr/`, `/bin/`, `/var/`
  - `Read(**/*:*)` 또는 `Write(**/*:*)` — allowed-tools 내
- **파일 유형별 처리**:
  - frontmatter: 패턴 매칭
  - scripts/: 패턴 매칭
  - SKILL.md body: 자연어 지시 탐지
- **메시지**: "Broad file access pattern [{pattern}] at {location}. Scope to specific directories (OWASP AST03 3.4)."
- **수정 제안**: `Read(**/*:*)` 대신 범위가 한정된 선언을 사용하세요. 예: `Read(src/**:*)`, `Read(references/*:*)`.
