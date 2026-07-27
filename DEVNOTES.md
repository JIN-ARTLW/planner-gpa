# DEVNOTES — 하루학점 내부 구조·수정 가이드

다른 환경에서 이어서 편집할 때 필요한 모든 것. (2026-07-28 기준, Claude Code로 제작)

## 배포

- GitHub Pages: `main` 브랜치 루트 → https://jin-artlw.github.io/planner-gpa/
- **푸시하면 끝.** 반영까지 1~2분, CDN 캐시가 최대 10분이라 확인할 땐 `?v=숫자`를 붙여 열 것
  (예: `.../planner-gpa/?v=12`) — 캐시 우회용일 뿐 코드와 무관

## index.html 구조 (단일 파일)

- `<style>`: iOS 다크/라이트 자동(`prefers-color-scheme`), CSS 변수 기반. 하단 탭바 `#tabbar`
- `<script>` 섹션 순서:
  1. 상수: `DEFAULT_REPO`(JIN-ARTLW/planner-data), `SITE_REPO`, `VAULT_ITER`(120만),
     `BINS`(기상/수면), `CATS`(schedule/todo/meal/exercise/nottodo), `WEIGHT_ORDER`, `GRADE_TABLE`
  2. 날짜 유틸 / 저장소(localStorage `pg:data:YYYY-MM`, `pg:settings`)
  3. 채점 `gradeOf()` — 아래 "채점 공식"
  4. GitHub 동기화(contents API) + **토큰 금고**(vault)
  5. 렌더링: `renderEntry` / `renderMonth` / `renderSettings` (innerHTML 재렌더 방식)
  6. 월간 이미지 `buildMonthCanvas()` (1080px 캔버스 → PNG)
  7. 이벤트: `#view` 클릭 위임(`data-act` 속성), 시작 시 `loadSiteVault()` + `initialSync()`

## 데이터 모델

```json
// localStorage["pg:data:2026-07"]
{ "days": { "2026-07-28": {
  "date": "2026-07-28",
  "wake": true,          // true=성공, false=실패, null=미기록
  "sleep": false,
  "cats": { "todo": {"done": 3, "total": 5}, ... },  // total=0이면 그날 채점 제외
  "updatedAt": 1785175987557   // 기기 간 병합 기준(최신 승리)
}}}
```

## 채점 공식 (gradeOf)

날짜 문자열을 FNV 해시 → mulberry32 PRNG 시드. **소비 순서 고정** (wake, sleep, schedule,
todo, meal, exercise, nottodo 가중치 7개 → curve → mixA → cutJit) — 순서 바꾸면 과거 점수가 전부 변함.

- 가중치 `w = 0.3 + rnd()*1.9` (0.3~2.2)
- 커브 `curve = 1.15 + rnd()*0.4` (1.15~1.55)
- 최약과목 반영 `mixA = 0.55 + rnd()*0.3`
- 등급컷 지터 `cutJit = 0~3점` (F 제외 전 등급 상향)
- `mixed = mixA*가중평균 + (1-mixA)*(0.35 + 0.65*최약과목비율)` ← 바닥 0.35
- `pct = mixed^curve * 100`
- **불변식**: 전부 지키면 mixed=1 → 정확히 100점 A+ 4.5. 모든 항 단조증가.

## GitHub 동기화

- 데이터: 비공개 `JIN-ARTLW/planner-data`의 `data/YYYY-MM.md` (월별 1파일)
- .md 구조: 사람이 읽는 월간 성적표 + 맨 아래 `<!-- planner-data:v1:BASE64 -->` 복원 블록
- 쓰기: 수정 1.5초 디바운스 자동 PUT + `visibilitychange/pagehide` 시 keepalive 즉시 PUT
  (`shaCache`에 파일 sha를 캐시해서 탭 닫힐 때 GET 없이 PUT 가능)
- 읽기: 앱 시작 시 이번 달+지난달 자동 pull, 날짜별 `updatedAt` 최신 승리로 병합

## 토큰 금고 (vault.json)

- 사이트 루트의 `vault.json` = fine-grained 토큰(planner-data 전용, Contents R/W)을
  **사용자 암호**로 PBKDF2(SHA-256, 120만회) + AES-GCM 암호화한 것. 공개돼도 암호 없인 못 엶
- 앱 시작 시 `fetch('vault.json')` → 토큰 없으면 기록 탭 위에 🔐 암호 칸 표시 →
  암호 입력 → 복호화 → settings 저장 → 자동 pull
- **토큰 교체 절차**(만료 1년): 새 토큰 발급 → 앱 설정 탭에 토큰 입력 → "🔐 금고 만들기" →
  암호 2회 입력 → 클립보드의 JSON을 GitHub 웹에서 `vault.json`에 덮어쓰기 커밋
- 대안 연결법: 설정 탭 "북마크 URL 복사" → `#t=토큰&r=repo` 해시 북마크 (해시는 서버 미전송)

## 아이콘

- `apple-touch-icon.png`(360px): 파란 배경 + 🎓. macOS에서 재생성하려면 osascript(JXA)로
  NSImage에 이모지 drawAtPoint 후 PNG 저장 (git log의 관련 커밋 참고)
- 파비콘은 `<head>`의 SVG 데이터 URI (🎓)

## 주의사항

- 시크릿모드 전용 사용자 — localStorage는 세션마다 사라짐. 영속성은 전적으로 GitHub 동기화
- `WEIGHT_ORDER`·PRNG·공식 파라미터를 바꾸면 **과거 날짜의 점수가 재계산되어 달라짐**
- 등급표(`GRADE_TABLE`)의 A+ 컷은 95 + 지터 최대 3 = 98 < 100 이어야 만점 보장이 유지됨
- `?v=N` 쿼리는 캐시 우회용 관례 (civic 프로젝트와 동일 패턴)
