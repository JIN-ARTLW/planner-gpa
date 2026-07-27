# 🎓 하루학점 — Daily GPA Planner

플래너 기록(기상·수면·일정·투두·식단·운동·NOT TO-DO)을 **대학교 학점(A+~F, 4.5 만점)** 으로 환산해주는 1인용 웹앱.
빌드 없이 `index.html` 한 파일로 동작하며 GitHub Pages로 호스팅합니다.

## 기능

- **기록 탭**: 날짜별로 몇 초 만에 입력
  - 기상/수면: 성공·실패 토글 (+ 실제 시각/시간 기록)
  - 일정·투두·식단·운동·NOT TO-DO: ＋/− 로 오늘의 개수를 정하고, 지킨 개수만큼 번호 탭
  - 몸무게·KPT 회고: 기록만 하고 채점 제외
- **성적표 탭**: 달력에 일별 등급, 월 평점(4.5 만점), 과목별 달성률, 몸무게 변화
- **저장**: 브라우저(localStorage) 자동 저장 + GitHub 저장소에 `YYYY-MM.md`로 커밋
  - .md 파일은 사람이 읽는 월간 성적표 + 하단에 복원용 데이터 블록(base64) 포함
  - 여러 기기에서 쓰면 날짜별 최신 수정본으로 병합

## 채점 방식

| 달성률 | 등급 | 평점 |
|---|---|---|
| 95%↑ | A+ | 4.5 |
| 90–94 | A0 | 4.0 |
| 85–89 | B+ | 3.5 |
| 80–84 | B0 | 3.0 |
| 75–79 | C+ | 2.5 |
| 70–74 | C0 | 2.0 |
| 65–69 | D+ | 1.5 |
| 60–64 | D0 | 1.0 |
| 60 미만 | F | 0.0 |

- 하루 점수 = 채점 대상 과목들의 달성률 평균 (기상·수면은 성공 100% / 실패 0%)
- 개수를 0으로 둔 과목은 그날 채점에서 제외 → 하루마다 과목 수가 달라도 됨
- 월 평점 = 채점된 날들의 평점 평균

## 호스팅 (GitHub Pages)

```bash
cd ~/code/planner-gpa
git init && git add -A && git commit -m "하루학점 첫 커밋"
gh repo create planner-gpa --public --source=. --push
```

그다음 GitHub 저장소 → **Settings → Pages → Branch: `main` / (root)** 저장.
잠시 후 `https://<username>.github.io/planner-gpa/` 에서 접속.

## 데이터 저장소 연결 (선택, 추천)

몸무게·회고가 공개되지 않도록 **데이터는 별도 비공개 저장소**에 저장하세요.

```bash
gh repo create planner-data --private --add-readme
```

1. GitHub → Settings → Developer settings → **Fine-grained personal access tokens**
2. Repository access: `planner-data` 1개만 선택
3. Permissions → **Contents: Read and write**
4. 앱의 **설정 탭**에 `username/planner-data` 와 토큰 입력 → 연결 테스트

토큰은 브라우저 localStorage에만 저장되며 어디에도 전송되지 않습니다(GitHub API 호출 제외).
"자동 업로드"를 켜면 기록을 고칠 때마다 4초 뒤 자동 커밋됩니다.

## 백업

- 성적표/설정 탭에서 **이번 달 .md 내려받기** 또는 **전체 백업(.json)**
- `.md` / `.json` 파일을 설정 탭에서 다시 가져오기 가능
