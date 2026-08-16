# wkd-gh.github.io

[![Gem Version](https://img.shields.io/gem/v/jekyll-theme-chirpy)][gem]&nbsp;
[![GitHub license](https://img.shields.io/github/license/cotes2020/chirpy-starter.svg?color=blue)][mit]

Data Analytics Engineer wkd-gh의 개인 블로그입니다. 업무 회고, 사이드 프로젝트, 스터디 기록을 정리합니다.

- **Live**: https://wkd-gh.github.io
- **Theme**: [jekyll-theme-chirpy][chirpy]

## Development

Ruby 버전은 `.ruby-version`(3.3.0)을 따릅니다.

```shell
bundle install
bundle exec jekyll serve
```

`http://localhost:4000`에서 확인할 수 있습니다.

## Deployment

`main` 브랜치에 push하면 GitHub Actions([.github/workflows/pages-deploy.yml](.github/workflows/pages-deploy.yml))가 자동으로 빌드 후 GitHub Pages에 배포합니다.

## Comments

댓글은 [giscus](https://giscus.app)(GitHub Discussions 기반)를 사용합니다.

## Credits

이 블로그는 [Chirpy Starter][chirpy-starter] 템플릿을 기반으로 만들어졌으며, [MIT][mit] 라이선스를 따릅니다.

[gem]: https://rubygems.org/gems/jekyll-theme-chirpy
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[chirpy-starter]: https://github.com/cotes2020/chirpy-starter
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE
