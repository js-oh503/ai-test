# DSR Daily Brief

DSR 와이어로프·섬유로프 업계 동향을 매일 자동 수집해 한국어 HTML 보고서를 생성하는 프로그램.  
→ 프로젝트 배경과 철학은 [SOUL.md](SOUL.md) 참고.

---

## 보고서 구성

| 탭 | 내용 |
|----|------|
| 업계 동향 | 오프쇼어·해양·크레인·광산 관련 기사. **Top 5 핵심 기사** + 카테고리별 전체 목록 |
| 경쟁사 동향 | Kiswire·고려제강·Bridon 등 경쟁사 언급 기사 |

각 카드: **기사 제목(링크)** + **요약(자동 번역)** + **DSR 연관 이유** + **대응 방안**

---

## 사전 준비

**Python 3.13**이 설치되어 있어야 합니다.  
설치 경로: `C:\Users\Admin\AppData\Local\Programs\Python\Python313\`

```powershell
# 의존성 설치 (최초 1회)
pip install -r requirements.txt
```

---

## 수동 실행

```powershell
$env:PYTHONIOENCODING = "utf-8"
& "C:\Users\Admin\AppData\Local\Programs\Python\Python313\python.exe" shipbuilding_daily.py
```

`reports/DSR동향_YYYYMMDD.html` 파일이 생성되고 브라우저가 자동으로 열립니다.

---

## 자동 실행 (매일 07:00)

작업 스케줄러에 등록되어 있습니다. 확인 방법:

```powershell
schtasks /query /tn "DSR Daily Brief"
```

스케줄러가 없거나 재등록이 필요할 때:

```powershell
schtasks /create /tn "DSR Daily Brief" /tr `
  "\"C:\Users\Admin\AppData\Local\Programs\Python\Python313\python.exe\" C:\Users\Admin\Desktop\workspace\shipbuilding_daily.py" `
  /sc daily /st 07:00 /f
```

---

## 파일 구조

```
workspace/
├── shipbuilding_daily.py   ← 메인 프로그램 (수집·분석·HTML 생성)
├── requirements.txt         ← feedparser
├── README.md               ← 이 파일
├── SOUL.md                 ← 프로젝트 철학·배경
├── CLAUDE.md               ← AI(Claude Code) 작업 지침
├── .gitignore
├── .claude/
│   └── skills/             ← Claude Code 커스텀 스킬 4종
│       ├── report-audit/
│       ├── fix-issue/
│       ├── issue-write/
│       └── issue-runner/
└── reports/                ← 생성된 보고서 (git 추적 안 함)
    └── DSR동향_YYYYMMDD.html
```

---

## Claude Code 스킬

| 요청 예시 | 실행 내용 |
|----------|----------|
| "보고서 감사해줘" | 최신 보고서 품질 검토 → GitHub 이슈 등록 |
| "이슈 #N 수정해줘" | 이슈 분석 → 코드 수정 → 커밋 → 이슈 닫기 |
| "이슈 등록해줘" | 문제를 GitHub 이슈로 작성 |
| "이슈 러너" | 열린 이슈 전체 자동 처리 |

---

## 자주 묻는 질문

**Q: 기사가 0건 수집되어요**  
→ 인터넷 연결 확인. 일부 RSS 피드는 일시적으로 접근 불가할 수 있습니다.

**Q: 번역이 안 되어요**  
→ 브라우저가 translate.googleapis.com에 접근할 수 있어야 합니다. 사내 방화벽 확인.

**Q: 비용이 드나요?**  
→ 없습니다. Claude API 없이 동작하며, 번역도 무료 엔드포인트를 사용합니다.
