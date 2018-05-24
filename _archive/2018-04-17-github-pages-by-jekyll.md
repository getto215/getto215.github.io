---
title: "Github pages + Jekyll + Minimal mistakes"
tags:
  - github-pages
---
# Github pages + Jekyll + minimal mistakes

## 관련 사이트
* Github pages ([github.io](https://github.io))
* 설정 방법 : [https://junhobaik.github.io/jekyll-apply-theme/](https://junhobaik.github.io/jekyll-apply-theme/)
* Minimal-mistakes
  * 오피스 : [https://mademistakes.com/work/minimal-mistakes-jekyll-theme/](https://mademistakes.com/work/minimal-mistakes-jekyll-theme/)
  * 샘플 포스트 : [https://mmistakes.github.io/minimal-mistakes/year-archive/](https://mmistakes.github.io/minimal-mistakes/year-archive/)
  * 샘플 포스트 소스 : [https://github.com/mmistakes/minimal-mistakes/tree/master/docs](https://github.com/mmistakes/minimal-mistakes/tree/master/docs)

## 명령어
git push
~~~
git init
git add --all
git commit -m "Initial Commit"
git remote add origin "https://github.com/getto215/getto215.github.io.git"
git push origin master
~~~

포스트 기본 형식
~~~yaml
---
title: "자주 사용하는 마크다운(markdown)-예제"
tags:
  - markdown
---
...
~~~

jekyll 로컬 실행
~~~bash
$ jekyll serve
# jekyll serve --drafts
~~~

