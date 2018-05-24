---
title: "자주 사용하는 마크다운(markdown)-예제"
tags:
  - markdown
---
자주 사용하는 마크다운(markdown)을 정리하였습니다.

Text can be **bold**, _italic_, or ~~strikethrough~~.
~~~
Text can be **bold**, _italic_, or ~~strikethrough~~.
~~~

--------

[Link to another page](404.html).
~~~
[Link to another page](404.html).
~~~

--------

# Header 1
## Header 2
### Header 3
#### Header 4
##### Header 5

~~~
# Header 1
## Header 2
### Header 3
#### Header 4
##### Header 5
~~~

--------

> This is a blockquote following a header.
> When something is important enough, you do it even if the odds are not in your favor.

~~~
> This is a blockquote following a header.
> When something is important enough, you do it even if the odds are not in your favor.
~~~

--------

```js
// Javascript code with syntax highlighting.
var fun = function lang(l) {
  dateformat.i18n = require('./lang/' + l)
  return true;
}
```

~~~
```js
// Javascript code with syntax highlighting.
var fun = function lang(l) {
  dateformat.i18n = require('./lang/' + l)
  return true;
}
```
~~~

--------

* This is an unordered list.
* This is an unordered list.
* This is an unordered list.
  * This is an unordered list.

~~~
* This is an unordered list.
* This is an unordered list.
* This is an unordered list.
  * This is an unordered list.
~~~

--------

1. This is an ordered list
2. This is an ordered list
3. This is an ordered list
   1. This is an ordered list

~~~
1. This is an ordered list
2. This is an ordered list
3. This is an ordered list
   1. This is an ordered list
~~~

--------

| head1        | head two          | three |
|:-------------|:------------------|:------|
| ok           | good swedish fish | nice  |
| ok           | good `oreos`      | hmm   |

~~~
| head1        | head two          | three |
|:-------------|:------------------|:------|
| ok           | good swedish fish | nice  |
| ok           | good `oreos`      | hmm   |
~~~

--------

* * *

~~~
* * *
~~~

--------

![](https://assets-cdn.github.com/images/icons/emoji/octocat.png)

~~~
![Alt "Hello"](https://assets-cdn.github.com/images/icons/emoji/octocat.png)
~~~

--------

![](https://guides.github.com/activities/hello-world/branching.png)

~~~
![Alt "Hello"](https://guides.github.com/activities/hello-world/branching.png)
~~~