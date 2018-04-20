---
title: "Github pages 구성 - jekyll"
---

* github.io
* jekyll 사용
* 마크다운 : http://www.hakawati.co.kr/405
* 참고
  * https://themeforest.net/category/static-site-generators/jekyll?_ga=2.125497034.1303046295.1523921610-1226921710.1523921610
  * http://tech.kakao.com/2016/07/07/tech-blog-story/
  * liquid template : https://shopify.github.io/liquid/
  * 테마 사이트 : http://themes.jekyllrc.org/
  * 설정법 : https://junhobaik.github.io/jekyll-apply-theme/
* 적용 테마
  * http://lanyon.getpoole.com/
  * https://github.com/poole/lanyon

### ruby 버전 확인
~~~bash
$ ruby --version
ruby 2.3.3p222 (2016-11-21 revision 56859) [universal.x86_64-darwin17]
~~~

### Jekyll 설치
~~~bash
$ sudo gem install jekyll bundler
~~~

### Github Page clone
~~~bash
git clone https://github.com/getto215/getto215.github.io.git
jekyll new . --force  

# http://localhost:4000/ 으로 확인
jekyll serve
~~~

## Lanyon 템플릿 설정
로컬에 템플릿을 다운받아서 세팅을 진행함

### Lanyon clone
* http://lanyon.getpoole.com/
~~~bash
git clone https://github.com/poole/lanyon.git
cp -r * ../getto215.github.io.git/
~~~

### _config.yml 수정
~~~yml
# 추가
paginate_path: "page:num"

# 주석 처리
#relative_permalinks: true

# 추가
gems: [jekyll-paginate]
~~~

## Github pages에 적용 
```bash
 $ git init
 $ git add --all
 $ git commit -m "Initial Commit"
 $ git remote add origin "https://github.com/getto215/getto215.github.io.git"
 $ git push origin master
```

### git 상태 확인
~~~bash
$ git status
~~~

### 포스트 기본 형식
~~~yml
---
layout: post
title:  "Welcome to Jekyll!"
date:   2017-05-06 13:45:35 +0900
categories: jekyll update
---
~~~


## To do
* page list
* ~~code highlights 내에 스크롤 생기도록~~ 필요없을듯 그냥 줄바꿈
* tag 적용
* 통계
* ~~한글글꼴~~
* 이미지 리사이징
* ~~컨텐츠 분할~~
* ~~날짜 형식 변환~~
* 이미지 로고
* 이미지 업로드 테스트