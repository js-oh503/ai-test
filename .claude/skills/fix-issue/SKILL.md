---
name: fix-issue
description: >
  GitHub 이슈 번호를 받아 현재 코드 상태를 파악하고,
  이슈에 수정 계획 댓글을 남긴 뒤 코드를 수정·커밋·푸시하고 이슈를 닫는다.
  "이슈 #N 수정해줘", "#N 고쳐줘" 같은 요청에 사용한다.
when_to_use: >
  사용자가 특정 GitHub 이슈 번호를 언급하며 수정을 요청할 때 호출한다.
argument-hint: "[issue-number] [repo]"
arguments: [issue, repo]
allowed-tools: Read, Edit, Grep, Bash, PowerShell
---

## 역할

GitHub 이슈 기반 버그 수정 담당자.
이슈를 확인 → 댓글 작성 → 코드 수정 → 커밋/푸시 → 이슈 닫기를 순서대로 수행한다.

## 실행 절차

### 1단계: 이슈 내용 확인
```powershell
gh issue view $issue --repo $repo
```
- `$repo` 가 없으면 현재 디렉토리의 git remote origin을 사용한다.

### 2단계: 관련 코드 위치 파악
- 이슈에 언급된 함수명·파일명을 Grep으로 찾는다.
- 문제 라인을 Read로 정확히 확인한다.

### 3단계: 수정 계획 댓글 등록
댓글은 반드시 코드 수정 전에 등록한다.
내용 구성:
- 현재 상태 파악 (문제 위치, 라인 번호)
- 원인 분석
- 수정 계획 (변경 전/후 비교)

임시 파일 경유로 등록 (인코딩 문제 방지):
```powershell
$tmpFile = [System.IO.Path]::GetTempFileName()
[System.IO.File]::WriteAllText($tmpFile, $comment, [System.Text.Encoding]::UTF8)
gh issue comment $issue --repo $repo --body-file $tmpFile
Remove-Item $tmpFile
```

### 4단계: 코드 수정
- Edit 도구로 최소한의 변경만 적용한다.
- 수정 후 프로그램을 실행해 오류가 없는지 확인한다.

### 5단계: 커밋 & 푸시
커밋 메시지 형식: `fix: [수정 내용 한 줄 요약] (close #$issue)`
- `close #N` 키워드를 포함해 GitHub이 이슈를 자동으로 닫게 한다.

```powershell
git add [수정된 파일]
git commit -m "fix: ... (close #$issue)"
git push origin master
```

### 6단계: 완료 보고
사용자에게 아래 형식으로 보고한다:
- 수정 내용 1~2줄 요약
- 커밋 SHA
- 이슈 URL

## 주의사항

- PowerShell에서 `&&` 연산자 사용 불가 → `;` 또는 순차 명령 사용
- 한글 출력 시 `$env:PYTHONIOENCODING = "utf-8"` 설정
- gh CLI PATH 갱신 필요 시:
  `$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")`
