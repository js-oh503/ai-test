---
name: code-audit-fix
description: >
  프로젝트 소스코드 전체를 분석해 버그·오류를 찾고,
  GitHub 이슈로 등록 → 수정 계획 댓글 → 코드 수정 → 커밋/푸시 → 이슈 닫기를
  한 번에 순서대로 실행한다.
  "이슈 찾아서 고쳐줘", "오류 분석하고 수정해줘", "코드 전체 점검해줘",
  "/code-audit-fix" 요청에 사용한다.
when_to_use: >
  사용자가 코드베이스 전체의 버그·오류를 한꺼번에 찾아 수정하고 싶을 때 호출한다.
  특정 이슈 번호 없이 "문제를 찾아서 고쳐줘" 형태로 요청하는 경우에 사용한다.
  fix-issue(특정 이슈 번호 지정)나 issue-runner(기존 이슈 처리)와 다르게,
  이 스킬은 이슈 자체를 새로 발굴하는 것부터 시작한다.
argument-hint: "[repo] [files...]"
arguments: [repo, files]
allowed-tools: Read, Edit, Grep, Glob, Bash, PowerShell, Agent
---

## 역할

코드 감사(audit) + 자동 수정 담당자.  
소스코드를 직접 읽고 버그를 발굴 → GitHub 이슈 등록 → 수정 계획 댓글 →
코드 수정 → 커밋/푸시 → 이슈 닫기까지 전 사이클을 완주한다.

---

## 실행 절차

### 0단계: 준비

```powershell
# gh CLI PATH 갱신
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
```

`$repo` 인자가 없으면 현재 디렉터리의 git remote에서 추출:
```powershell
git remote get-url origin
# 결과 예: https://github.com/DSR-family/sales-analysis.git
# → repo = "DSR-family/sales-analysis"
```

`$files` 인자가 없으면 주요 소스 파일을 자동 감지:
- `*.py` — Python 스크립트
- `*.html` — HTML/JS 대시보드·UI
- `*.ts`, `*.tsx` — TypeScript/React
- `*.js` (node_modules 제외)

---

### 1단계: 코드 감사 (버그 발굴)

분석 대상 파일을 Read/Grep으로 읽고 아래 항목을 중심으로 점검한다.

#### 런타임 오류 (최우선)
- `ZeroDivisionError` / `NaN` 전파 — 0으로 나누는 분기, `sum()==0` 미처리
- `KeyError` / `undefined` 참조 — dict/객체 키 존재 확인 없이 접근
- `TypeError` — 타입 불일치 (str vs int, null vs 숫자 등)
- 반복문 탈출 로직 오류 — `break`가 의도한 루프를 탈출하지 않는 경우
- 누락된 `await`, 비동기 처리 오류

#### 데이터 처리 오류
- 필터링 시 누락 행 발생 — `NaN` 비교, 빈 문자열 처리 미흡
- 집계 불일치 — 한쪽에서 제외한 행이 다른 집계에는 포함
- 타입 혼재 — `toFixed()` 결과(문자열)와 숫자 비교

#### UI/표시 오류 (HTML·JS)
- 데이터 없을 때 빈 차트·빈 테이블 무음 실패
- 단일 데이터셋일 때 전년대비 계산 오류
- 차트가 데모 모드에서만 동작하고 실파일 모드에서 미작동

#### 의존성 오류
- `requirements.txt` / `package.json`에 실제 사용 모듈 누락
- 환경에 따라 다른 결과를 내는 경로 하드코딩

파일이 크거나 여러 개이면 **Agent(subagent_type=Explore)**로 병렬 분석한다.

---

### 2단계: 이슈 분류 및 우선순위 결정

발견한 문제를 아래 기준으로 분류한다:

| 등급 | 기준 | 처리 방향 |
|------|------|-----------|
| 🔴 심각 | 프로그램이 실행 자체가 안 됨 / 핵심 기능 완전 불작동 | 즉시 수정 |
| 🟠 중간 | 특정 조건에서 오류 발생 / 결과값 왜곡 | 이슈 등록 후 수정 |
| 🟡 낮음 | 엣지 케이스, UX 미흡, 경고 미표시 | 이슈 등록 (수정은 선택) |

**오탐 검증**: 이슈 등록 전 반드시 해당 코드를 Read로 재확인해 실제 문제인지 검증한다.

---

### 3단계: GitHub 이슈 등록

심각·중간 등급 문제를 이슈로 등록한다.

이슈 본문 구성:
```
## 문제
[한 문장으로 무엇이 잘못됐는지 설명]

## 재현 방법
[단계별 재현 조건]

## 위치
[파일명 N줄]

## 영향
[사용자에게 미치는 영향]
```

한글 이슈 등록 시 반드시 임시 파일 경유:
```powershell
$tmp = [System.IO.Path]::GetTempFileName()
[System.IO.File]::WriteAllText($tmp, $body, [System.Text.Encoding]::UTF8)
gh issue create --repo $repo --title "[BUG] 제목" --body-file $tmp
Remove-Item $tmp -ErrorAction SilentlyContinue
```

---

### 4단계: 수정 계획 댓글 등록 (코드 수정 전 필수)

각 이슈에 수정 계획 댓글을 먼저 남긴다.

댓글 내용 구성:
```
## 수정 계획

**원인:** [한 줄 설명]

**수정 내용 (파일명 N줄):**
- 변경 전: [코드]
- 변경 후: [코드]
```

```powershell
$tmp = [System.IO.Path]::GetTempFileName()
[System.IO.File]::WriteAllText($tmp, $comment, [System.Text.Encoding]::UTF8)
gh issue comment $issueNum --repo $repo --body-file $tmp
Remove-Item $tmp -ErrorAction SilentlyContinue
```

---

### 5단계: 코드 수정

이슈 번호 순서대로 한 건씩 수정한다.

- **Edit 도구**로 최소한의 변경만 적용 (관련 없는 코드 건드리지 않음)
- Python 파일이면 수정 후 실행해 오류 없음 확인:
  ```powershell
  $env:PYTHONIOENCODING = "utf-8"
  & "C:\Users\Admin\AppData\Local\Programs\Python\Python313\python.exe" [파일명]
  ```
- HTML/JS 파일이면 브라우저 실행으로 확인:
  ```powershell
  Start-Process "[파일명].html"
  ```
- 수정 중 기존 기능이 깨지지 않도록 주의한다.

---

### 6단계: 커밋 & 푸시

수정이 완료된 파일을 커밋한다.

커밋 메시지 형식:
```
fix: [수정 내용 한 줄 요약] (close #N, close #M, ...)
```

여러 이슈를 한 커밋에 묶을 수 있으면 묶는다. 파일이 달라 충돌 우려가 있으면 이슈별 커밋.

```powershell
git add [수정된 파일들]
git commit -m "fix: 버그 수정 N건 (close #N, close #M)"
git push origin [브랜치명]
```

---

### 7단계: 이슈 닫기 & 완료 보고

커밋 메시지의 `close #N` 키워드로 자동 닫히지 않은 이슈는 수동으로 닫는다:
```powershell
gh issue close $issueNum --repo $repo --comment "수정 완료. 커밋: [SHA]"
```

전체 완료 후 사용자에게 아래 형식으로 보고한다:

```
## 코드 감사 & 수정 완료

### 발견된 이슈
| # | 등급 | 내용 | 결과 |
|---|------|------|------|
| #N | 🔴 심각 | 내용 | ✅ 수정 완료 (커밋 SHA) |
| #N | 🟠 중간 | 내용 | ✅ 수정 완료 (커밋 SHA) |

### 요약
- 분석 파일: N개
- 발견 이슈: N건 (심각 N / 중간 N / 낮음 N)
- 수정 완료: N건
- 오탐 제외: N건
```

---

## 예외 처리

| 상황 | 대응 |
|------|------|
| 오탐으로 확인된 이슈 | 이슈를 즉시 닫고 "오탐" 댓글 남김 |
| 수정 방법 불명확 | 댓글에 "추가 분석 필요" 표시 후 다음 이슈로 |
| 수정 후 실행 오류 | `git checkout [파일]`로 롤백, 이슈에 실패 댓글 |
| 이슈 등록 실패 (라벨 없음 등) | `--label` 옵션 제거 후 재시도 |

---

## 주의사항

- PowerShell에서 `&&` 사용 불가 → `;` 또는 순차 명령 사용
- 한글 이슈·댓글은 반드시 임시 파일 경유 (`[System.IO.File]::WriteAllText`)
- `Remove-Item $tmp` 는 `-ErrorAction SilentlyContinue` 옵션 추가 (경로 오류 방지)
- 이슈 1개 처리가 완전히 끝난 후 다음 이슈로 진행 (병렬 수정 금지)
- 분석은 병렬(Agent 활용)로, 수정은 직렬로 진행
- git config `user.name` / `user.email` 미설정 시 커밋 전 확인 후 설정
