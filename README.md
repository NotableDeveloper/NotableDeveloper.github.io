# Engineering Notes

Jekyll 기반 GitHub Pages 기술 블로그입니다.

## 시작하기

```bash
cd my-blog
bundle install
bundle exec jekyll serve --livereload
```

브라우저에서 `http://localhost:4000`을 엽니다.

## 설정할 항목

- `_config.yml`의 `title`, `description`, `url`, `baseurl`, `author`를 실제 블로그 정보로 변경합니다.
- 글은 `_posts/YYYY-MM-DD-title.md` 형식으로 추가합니다.
- GitHub 저장소의 Settings → Pages → Source를 **GitHub Actions**로 설정합니다.
- `main` 브랜치에 push하면 `.github/workflows/pages.yml`이 빌드와 배포를 수행합니다.
