# KAERI 연구 노트

사이트: https://seosangryeong.github.io/Kaeri_note/

로봇 프로젝트 기록 여섯 개를 한 사이트로 모은 저장소입니다. 왼쪽에는 프로젝트의 기록 목록, 오른쪽에는 프로젝트 목록이 있습니다.

기록 내용은 여기 없고, 각 프로젝트 저장소의 `notes/`, `assets/`를 빌드할 때 읽어 옵니다. 각 프로젝트 사이트는 따로 그대로 있습니다.

| 프로젝트 | 저장소 | 따로 보는 사이트 |
| --- | --- | --- |
| Kanake 뱀로봇 | [Kanake_note](https://github.com/seosangryeong/Kanake_note) | https://seosangryeong.github.io/Kanake_note/ |
| Teraffe 고소작업로봇 | [Teraffe_note](https://github.com/seosangryeong/Teraffe_note) | https://seosangryeong.github.io/Teraffe_note/ |
| PipeRobot 배관 로봇 | [piperobot_note](https://github.com/seosangryeong/piperobot_note) | https://seosangryeong.github.io/piperobot_note/ |
| FR3 로봇팔 | [fr3_note](https://github.com/seosangryeong/fr3_note) | https://seosangryeong.github.io/fr3_note/ |
| R-BIZ 챌린지 로버 | [rbiz_challenge_note](https://github.com/seosangryeong/rbiz_challenge_note) | https://seosangryeong.github.io/rbiz_challenge_note/ |
| 기타 프로젝트 | [other_projects](https://github.com/seosangryeong/other_projects) | https://seosangryeong.github.io/other_projects/ |

## 업데이트

- 프로젝트 저장소에 기록을 push하면, 이 사이트는 **매일 새벽 3시(KST)**에 자동으로 다시 빌드합니다.
- 바로 반영하려면 GitHub의 Actions → *Deploy KAERI notes* → **Run workflow**를 누릅니다.

## 미리 보기

프로젝트 폴더들이 이 폴더와 같은 곳(`~/kaeri_note/`)에 있으면 그대로 읽습니다.

```bash
python3 build.py && python3 check_site.py
python3 -m http.server -d _site 8080     # http://localhost:8080
```

## 프로젝트 추가

`build.py`의 `PROJECTS`에 한 줄, `.github/workflows/site.yml`에 checkout 한 단계를 추가합니다.

## 구성

| 파일 | 내용 |
| --- | --- |
| `home.md` | 첫 화면의 연구 소개 (`{{period:slug}}`는 `build.py`의 기간으로 바뀜) |
| `build.py` | 프로젝트 노트 → `_site/` (홈 + 프로젝트별 폴더), 프로젝트 순서·기간 |
| `styles.css` | 스타일 (각 프로젝트 사이트와 같은 모양 + 오른쪽 프로젝트 목록) |
| `check_site.py` | 깨진 링크 검사 |
| `.github/workflows/site.yml` | push·매일·수동 실행 시 빌드·검사·GitHub Pages 배포 |
