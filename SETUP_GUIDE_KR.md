# 블로그 배포 가이드

## 1. GitHub 레포 만들기
1. GitHub에서 새 레포지토리 생성 → 이름을 **정확히** `wkd-gh.github.io` 로 지정 (본인 계정명.github.io)
2. Public으로 생성 (Private이면 GitHub Pages 무료 배포 안 됨, 단 Pro 계정이면 가능)

## 2. 이 폴더 내용을 push
로컬(본인 PC)에서 이 폴더를 열고:

```bash
cd jangho-blog
git init
git remote add origin https://github.com/wkd-gh/wkd-gh.github.io.git
git add .
git commit -m "chore: init blog"
git branch -M main
git push -u origin main
```

## 3. GitHub Pages 설정
1. 레포 → Settings → Pages
2. Build and deployment → Source를 **GitHub Actions**로 선택
   (이 템플릿은 `.github/workflows/pages-deploy.yml`이 이미 포함되어 있어서 push하면 자동 빌드됨)
3. 몇 분 후 `https://wkd-gh.github.io` 접속하면 블로그가 떠 있을 것

## 4. 커스터마이징 체크리스트
- `_config.yml`: email, LinkedIn 링크 등 채워넣기
- `_tabs/about.md`: 자기소개 페이지 작성
- `assets/img/`: 아바타 이미지 교체 (선택)
- `_posts/`: 여기에 `YYYY-MM-DD-제목.md` 형식으로 글 작성 → commit할 때마다 잔디 심어짐

## 5. 로컬에서 미리보기 (선택, Ruby 필요)
```bash
bundle install
bundle exec jekyll serve
# http://localhost:4000 에서 확인
```
Ruby/Jekyll 설치가 번거로우면 로컬 미리보기 없이 그냥 push해서 GitHub Actions 빌드 결과로 확인해도 됩니다.

## 글쓰기 워크플로우 예시
```bash
# 새 글 작성
touch _posts/$(date +%Y-%m-%d)-오늘의-회고.md
# 작성 후
git add . && git commit -m "post: 오늘의 회고" && git push
# → 자동 빌드/배포 + 잔디 심어짐
```
