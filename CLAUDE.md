# CLAUDE.md

이 저장소는 [wkd-gh.github.io](https://wkd-gh.github.io) — Jekyll + [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 기반 개인 기술/일상 블로그입니다. GitHub Pages로 배포되며, repo는 `wkd-gh/wkd-gh.github.io`, 브랜치는 `main` 하나만 씁니다.

이 문서는 이 블로그를 운영하면서 실제로 겪었던 삽질과 그 결과 정해진 규칙들을 정리한 것입니다. 여기 적힌 "함정" 항목들은 전부 한 번씩 실제로 터졌던 문제라, 다시 반복하지 않도록 특히 주의해서 지켜주세요.

## 사이트 언어 설정 — 건드리지 말 것

`_config.yml`의 `lang: ko`는 **의도된 설정**입니다. Chirpy의 한국어 로케일 파일명은 `ko-KR`이라 `ko`와 매칭되지 않고, 그 결과 UI 문구(버튼/툴팁/패널 제목 등)가 영어(`en`)로 폴백됩니다. 사용자가 이 영어 UI를 선호해서 일부러 이렇게 둔 상태입니다. **버그처럼 보인다고 `ko-KR`로 고치자고 제안하지 마세요.**

## 테마 오버라이드 구조 (중요)

gem 기반 테마라서 사이트 소스에 gem과 **동일한 상대경로**로 파일을 두면 로컬 파일이 gem의 기본 파일을 덮어씁니다 (`_sass`, `_layouts`, `_includes`, `assets` 전부 해당). gem 자체 파일은 직접 수정하지 않습니다 — 항상 로컬에 오버라이드 파일을 만드세요.

- gem 원본 위치 확인: `bundle show jekyll-theme-chirpy`

현재 로컬 오버라이드 목록과 이유:

| 파일 | 이유 |
|---|---|
| `_includes/favicons.html` | 파비콘 세트를 커스텀 아이콘으로 교체 (기본값은 무당벌레 캐릭터) |
| `_includes/footer.html` | 개인정보처리방침 링크 + GoatCounter 방문자 수 추가, "Some rights reserved." / "Using the Chirpy theme for Jekyll." 문구 제거 |
| `assets/css/jekyll-theme-chirpy.scss` | 이전/다음 게시물 버튼(`.post-navigation .btn`) hover 시 전체가 파란색으로 칠해지는 기본 스타일을 옅은 틴트 배경으로 완화 |

## 글 작성 규칙 (`_posts/*.md`)

### 프론트매터 템플릿

```yaml
---
title: 
date: YYYY-MM-DD HH:MM:SS +0900
categories: [...]   # 비워달라는 요청이 있으면 categories: [] 로 실제로 비울 것
tags: [...]
image: thumbnail.png            # 썸네일이 있을 때만
media_subpath: /assets/img/posts/<post-slug>/   # 본문에 이미지를 쓰면 필수
---
```

- **`date`는 반드시 "지금"보다 이전 시각으로.** Jekyll은 `future: false`가 기본값이라 미래 시각 글은 빌드에서 통째로 제외됩니다 ("글을 분명히 만들었는데 사이트에 안 뜬다"는 문제로 바로 이어짐). 글을 만들기 전에 `date` 셸 커맨드로 현재 시각을 확인하고, 그보다 확실히 이전 시각으로 설정하세요.
- 같은 날 여러 편을 올릴 때는 시간을 30분~1시간 단위로 띄워서 정렬 순서를 통제합니다 (예: 12:00, 13:00, 14:00…).

### 본문 구조

- 섹션 제목은 **`###`(h3)**를 씁니다. `####`(h4)를 쓰면 안 됩니다 — Chirpy의 우측 목차(TOC)는 본문에 `h2` 또는 `h3`가 있어야 활성화되고, 활성화되면 `h2/h3/h4`까지 스캔해서 목차를 만듭니다. 즉 최상위 섹션은 `###`, 그 아래 하위 소제목이 필요하면 `####`를 씁니다.
- 섹션 제목 스타일: `### <i class="fas fa-아이콘 fa-fw"></i> **제목**` — Font Awesome Free 아이콘 + 볼드. 마무리 섹션은 관례적으로 `fa-flag-checkered`를 씁니다.
- 강조 인용구는 kramdown IAL로: `{: .prompt-tip }`, `{: .prompt-warning }`, `{: .prompt-info }`, `{: .prompt-danger }`
- 유튜브 임베드는 raw `<iframe>` 대신 `{% include embed/youtube.html id='VIDEO_ID' %}` 사용 (반응형 16:9 자동 처리, 테마 기본 스타일 적용됨).
- 글 톤: 1인칭, 담백한 회고체("~했다", "~인 것 같다"). 원본 자료(메모, 마케팅 카피 등)를 그대로 복붙하기보다 항상 본인 목소리로 재정리합니다.

## 이미지 규칙

- 경로: `assets/img/posts/<post-slug>/`. 썸네일은 `thumbnail.png`, 본문 이미지는 의미를 알 수 있는 영문 스네이크/케밥 케이스 파일명.
- 마크다운에서 이미지는 **항상 크기를 지정**합니다: `![alt](file.png){: width="W" height="H" }`.
  - **SVG는 특히 필수입니다.** `<svg>` 루트 태그 자체에도 `width`/`height` 속성을 직접 넣어야 하며, `viewBox`만으로는 부족합니다. 테마 CSS가 `img { max-width:100%; height:auto }`라서, 크기 정보가 없는 SVG는 브라우저가 렌더링 크기를 못 잡고 찌그러들어 아예 안 보이는 버그가 실제로 있었습니다.
- 이미지 캐시 등에서 복사해온 새 파일은 권한이 `rw-------`로 오는 경우가 있어 웹서버가 못 읽을 수 있습니다 — `chmod 644`로 확인하세요.
- 다이어그램이 필요한데 스크린샷이 없는 경우: 이 환경엔 matplotlib/PIL이 기본 설치되어 있지 않습니다 (필요하면 `pip3 install --user`로 설치 가능하지만, 매번 설치하기보다) **SVG를 직접 손으로 그려서** 쓰는 편이 낫습니다. 원본 저작물이라 저작권 문제도 없습니다.

## PDF/메모 기반 포스팅 워크플로우

사용자가 노션 내보내기 PDF나 메모 PDF를 주고 "정리해서 블로그에 올려줘"라고 요청하는 패턴이 자주 있습니다. 이때 순서:

1. `Read` 도구로 PDF 전체를 읽고 내용을 파악합니다.
2. **민감정보 스크리닝은 필수입니다.** API 키, Access Token, Webhook URL, 개인 이메일, GitHub PAT 등이 메모에 평문으로 남아있는 경우가 실제로 있었습니다. 절대 본문에 옮기지 말고, 살아있는 토큰처럼 보이면 로테이션을 권고하세요.
3. 회사명 등 특정 가능한 정보는 사용자가 원하면 "M사(MSP 업체)"처럼 일반화합니다. 요청이 없어도 민감해 보이면 먼저 확인하세요.
4. PDF에 슬라이드 사진 등 이미지가 포함돼 있으면, PyMuPDF(`pip3 install --user pymupdf`)로 일정 크기 이상인 이미지만 추출해서(`page.get_images(full=True)` + 작은 아이콘/로고 필터링) 관련 섹션에 배치하면 글이 훨씬 풍부해집니다.
5. `categories`/`image`를 비워달라는 요청이 있으면 정말 비우고, 나머지 필드는 최대한 채워서 사용자가 최소한만 손보면 되게 합니다.

## 빌드 확인

```bash
bundle exec jekyll build --destination /tmp/_site_check
bundle exec jekyll serve   # 로컬에서 직접 눈으로 확인하고 싶을 때
```

- **새 글 작성, 구조 변경, 새 이미지 포맷, 새로운 Liquid/HTML 문법을 쓸 때만** 빌드 검증을 실행하세요.
- **프론트매터 한 줄 수정, 썸네일 교체 같은 기계적인 수정 후에는 빌드 검증을 생략**하세요. 반복적인 검증 호출은 사용자가 명시적으로 거부한 적이 있습니다 — 그냥 뭘 했는지 보고하고 끝내면 됩니다.

## Git / 배포

- **커밋과 push는 명시적으로 요청받았을 때만** 합니다. 작업을 다 끝내고도 커밋 안 된 상태로 남겨두는 것 자체는 문제 없습니다 — push 여부는 먼저 물어보세요.
- 커밋 메시지는 `Fix:`, `Update:`, `Add:`, `Chore:` 접두사 + 한국어 설명 (기존 로그 참고).

## 알려진 함정 (재발 방지)

- **미래 시각 글이 사이트에 안 보임** → `future: false` 기본값 때문. `date` 필드부터 확인.
- **SVG 이미지가 안 보임** → `<svg>`에 `width`/`height`가 없어서. `viewBox`만으로는 부족.
- **파비콘이 옛날 아이콘(무당벌레)으로 계속 나옴** → 파비콘은 `.ico` 파일 하나가 아니라 `.ico` / `favicon-96x96.png` / `apple-touch-icon.png` / PWA용 `web-app-manifest-{192,512}.png` 세트입니다. 하나만 바꾸면 나머지는 테마 기본값이 계속 노출됩니다. `_includes/favicons.html`은 실제로 로컬에 존재하는 파일만 링크하도록 관리되어 있습니다.
- **`lang: ko`인데 UI가 영어로 나옴** → 버그 아님, 의도된 상태입니다 (위 "사이트 언어 설정" 항목 참고).

## 현재 켜진 외부 서비스

`privacy.md`(개인정보처리방침)를 갱신할 때 기준이 되는 실제 상태입니다. 새 서비스를 추가/제거하면 `privacy.md`도 같이 업데이트하세요.

- **댓글**: Giscus (`wkd-gh/wkd-gh.github.io` repo, GitHub Discussions 기반, `Announcements` 카테고리)
- **방문 통계**: GoatCounter (site code: `wkd-gh`). `_config.yml`의 `analytics.goatcounter.id`와 `pageviews.provider: goatcounter`에 연결되어 footer 총 방문자 수 + 게시글별 조회수로 쓰입니다. 쿠키/IP를 저장하지 않는 프라이버시 친화적 서비스입니다.
- **호스팅**: GitHub Pages (GitHub, Inc.)
- Google Analytics 등 `_config.yml`의 나머지 `analytics.*` 프로바이더는 전부 미사용(`id` 빈 값)입니다.
