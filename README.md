# swoobuntu.github.io

Cloud Engineer 최성우(Karfa Choi)의 포트폴리오 사이트입니다.

**🔗 https://swoobuntu.github.io**

## 구성

```
push (main)
   │
   ▼
GitHub Actions ─ Hugo 설치 → hugo --minify 빌드 → Pages 아티팩트 업로드
   │
   ▼
GitHub Pages ─ https://swoobuntu.github.io
```

| 경로 | 내용 |
|---|---|
| `data/*.yaml` | 프로필 · 경력 · 기술 · 자격증 · 학력 · 프로젝트 카드 |
| `content/projects/*.md` | 프로젝트 상세 페이지 |
| `layouts/` | 직접 만든 레이아웃 (외부 테마 없음) |
| `assets/css/main.css` | 스타일 (라이트 · 다크 모드, 모바일 대응) |
| `static/` | 링크 미리보기 이미지, 다이어그램 |
| `.github/workflows/deploy.yml` | 빌드 · 배포 파이프라인 |

## 수정하는 법

- 자격증 · 기술 · 경력 추가 → `data/` 아래 YAML에 항목만 추가
- 프로젝트 내용 수정 → `content/projects/` 아래 Markdown 수정
- `main`에 push하면 1~2분 뒤 사이트에 반영

## 로컬 미리보기

```bash
hugo server
```
