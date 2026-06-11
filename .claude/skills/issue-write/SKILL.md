---
name: issue-write
description: >
  발견된 문제·개선점을 GitHub 이슈로 등록한다.
  "이슈 등록해줘", "이슈 써줘", "이슈라이트", "/issue-write" 요청에 사용한다.
when_to_use: >
  사용자가 특정 문제나 개선사항을 GitHub 이슈로 등록하고 싶을 때 호출한다.
  문제 내용이 대화에 있거나, 파일·코드를 분석해서 문제를 발견했을 때 사용한다.
argument-hint: "[repo] [제목 또는 내용 설명]"
arguments: [repo, content]
allowed-tools: Read, Grep, Bash, PowerShell
---

## 역할

GitHub 이슈 작성 전문가. 사용자가 전달한 문제·개선사항을 분석해
명확하고 실행 가능한 GitHub 이슈를 작성하고 등록한다.

## 실행 절차

### 1단계: 입력 파악
- `$content` 가 있으면 해당 내용을 기반으로 이슈 작성
- 없으면 대화 맥락에서 등록할 문제를 파악
- `$repo` 가 없으면 현재 디렉토리의 git remote origin 사용:
  ```powershell
  git remote get-url origin
  ```
  URL에서 `github.com/owner/repo` 형태로 추출

### 2단계: 이슈 분류 판단
| 유형 | 라벨 | 판단 기준 |
|------|------|-----------|
| 버그 | `bug` | 코드 오류, 예외 발생, 잘못된 동작 |
| 개선 | `enhancement` | 기능 추가, 성능 개선, UX 향상 |
| 질문 | `question` | 동작 불명확, 스펙 확인 필요 |

### 3단계: 이슈 본문 작성
아래 구조로 작성한다:

```
## 문제 (버그) 또는 ## 목적 (개선)
무엇이 잘못되었는지 / 무엇을 달성하려는지 1~3줄

## 재현 방법 (버그일 때)
1. ...
2. ...
3. 결과: ...

## 기대 동작
...

## 수정 방법 (알고 있을 때)
...
```

### 4단계: 이슈 등록
한글 인코딩 문제를 방지하기 위해 반드시 임시 파일 경유:

```powershell
$tmpFile = [System.IO.Path]::GetTempFileName()
[System.IO.File]::WriteAllText($tmpFile, $body, [System.Text.Encoding]::UTF8)
gh issue create --repo $repo --title "제목" --body-file $tmpFile --label "라벨"
Remove-Item $tmpFile
```

### 5단계: 결과 보고
등록된 이슈 URL과 번호를 사용자에게 알린다.

## 주의사항
- 제목 앞에 분류 태그 붙이기: `[버그]`, `[개선]`, `[질문]`
- gh CLI PATH 갱신:
  `$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")`
- 한 번에 여러 이슈를 등록할 때는 순서대로 처리하고 각각 URL을 보고한다
