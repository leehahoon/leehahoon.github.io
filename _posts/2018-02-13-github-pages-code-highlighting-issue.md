---
layout: post
title: "Github pages code highlighting"
date: 2018-02-13 23:00:13 -0400
categories: development
---

### Github Pages
원래 네이버 블로그에서 글을 작성하다가 깃헙 페이지로 옮겨봤는데 괜찮아서 계속 하려고 했다. 근데 마크다운 형식으로 깃헙에 커밋을 했는데 정작 깃헙 블로그에선 code syntax highliting이 적용되지 않은 소스코드가 출력됐다. 처음엔 \_config.yml 문제인 줄 알았지만 사용한 jekyll 테마가 코드 하이라이팅을 지원하지 않았다. 그래서 어떻게 이 문제를 해결했는지 내가 사용한 방법을 적으려 한다.

1. kramdown, rouge 설치
```bash
gem install kramdown rouge
```  

2. \_config.yml 수정
```
markdown: kramdown
kramdown:
	input: GFM
	syntax_highlighter: rouge
```

3. CSS 파일 수정
```
rougify style monokai >> _sass/style.scss
rougify style monokai >> assets/main.scss
```

나의 경우, "Start Bootstrap - Clean Blog" 테마를 사용했다. 이 테마의 경우 css 파일이 \_sass/style.scss 였다. 그리고 assets/main.scss도 관련있을 것 같아서 코드 하이라이팅에 관련된 css 코드를 추가했다. 그 후, markdown 파일에서 \`\`\`{code}\`\`\` 같은 형식으로 입력하면 Syntax Highlighting이 적용된 결과물을 확인할 수 있다.

> monokai 말고도 rougify help style 를 통해 다양한 코드 하이라이팅 테마를 확인할 수 있습니다.  

> https://benhur07b.github.io/2017/03/25/add-syntax-highlighting-to-your-jekyll-site-with-rouge.html  