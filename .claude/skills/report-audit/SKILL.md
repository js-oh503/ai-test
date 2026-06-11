---
name: report-audit
description: >
  생성된 보고서(HTML)를 검토해서 품질 문제를 찾고 GitHub 이슈로 등록한다.
  "보고서 검토해줘", "브리핑 점검해줘", "이슈 등록해줘" 같은 요청에 사용한다.
when_to_use: >
  사용자가 보고서·브리핑·뉴스 수집 결과물의 품질을 검토하거나
  발견된 문제를 GitHub 이슈로 등록하고 싶을 때 호출한다.
argument-hint: "[repo] [report-path]"
arguments: [repo, report_path]
allowed-tools: Read, Grep, Bash, PowerShell
---

## 역할

보고서 품질 감사관. 수집된 보고서를 읽고 아래 4가지 기준으로 문제를 찾아
GitHub 이슈로 등록한다.

## 감사 기준

1. **관련성** — 업무·도메인과 무관한 기사가 포함되어 있는가?
2. **시의성** — 기준 기간(최근 24시간 또는 이번 주) 밖의 오래된 자료가 있는가?
3. **중복** — 동일 사건이 여러 출처에서 중복 수집되었는가?
4. **수치 출처** — 구체적인 수치(%, 금액, 건수)에 출처 표기가 있는가?

## 실행 절차

### 1단계: 보고서 파일 읽기
- `$report_path` 가 주어지면 해당 파일을 읽는다.
- 주어지지 않으면 `reports/` 폴더에서 가장 최근 HTML 파일을 찾는다.

### 2단계: 4가지 기준으로 분석
각 기준별로 문제 항목을 목록화한다.
문제 없으면 "이상 없음"으로 표기한다.

### 3단계: 분석 결과를 사용자에게 표로 보고
| 기준 | 발견 건수 | 내용 요약 |
형식으로 먼저 보고한다.

### 4단계: GitHub 이슈 등록
- `$repo` 가 주어지면 해당 레포에, 없으면 현재 디렉토리의 git remote origin에 등록한다.
- 버그(코드 결함)는 `bug` 라벨, 개선(필터·로직)은 `enhancement` 라벨을 붙인다.
- 이슈 본문은 임시 파일을 경유해 등록한다 (PowerShell 인코딩 문제 방지):
  ```
  $tmpFile = [System.IO.Path]::GetTempFileName()
  [System.IO.File]::WriteAllText($tmpFile, $body, [System.Text.Encoding]::UTF8)
  gh issue create --repo $repo --title "..." --body-file $tmpFile --label "..."
  Remove-Item $tmpFile
  ```
- 이슈 등록 후 URL 목록을 사용자에게 알린다.

## 출력 형식

```
## 감사 결과 요약
| 기준     | 건수 | 내용 |
|----------|------|------|
| 관련성   | N건  | ...  |
| 시의성   | N건  | ...  |
| 중복     | N건  | ...  |
| 수치출처 | N건  | ...  |

## 등록된 GitHub 이슈
- #N [제목](URL) — bug/enhancement
```
