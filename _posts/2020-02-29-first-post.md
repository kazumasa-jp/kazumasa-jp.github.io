---
title: First Post
author: Kazumasa
layout: post
tags: [Jekyll]
---
静的サイトジェネレータの勉強を兼ねて、GitHub Pagesを使うことにした。まずはGitHub Pagesと相性が良いという話のJekyllを使ってみる。

テーマにはjekyll-theme-prologueを選んだ。デザインはシンプルで好ましいが、フッターにJekyllへ移植した人の名前が書いてあるのが少し引っかかる。リンクも切れているし、消したほうがいいかなという気もする。使わせてもらっておきながら、こういうことを考えるのも失礼な話だが。

コンテンツは、技術メモのようなもののリンクになるだろう。メモの内容は改定するかもしれない。
Blogページのほうは多分作業記録になると思う。

そこで、今日の作業だが、Blogの要約がうまくいっていなかったので、jekyll-theme-prologue-0.3.3/_layouts/blog.htmlを/_layouts/blog.htmlにコピーし、20行目のtruncatewordsをtruncateに変更した。
{% highlight html %}
{% raw %}
{%- capture _excerpt -%}<p>{{- _post.excerpt | strip_html | truncate: 100 -}}</p>{%- endcapture -%}
{% endraw %}
{% endhighlight %}
