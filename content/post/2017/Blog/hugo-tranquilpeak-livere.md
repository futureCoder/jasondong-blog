---
title: "Hugo博客tranquilpeak主题使用LiveRe评论系统"
date: 2017-08-30T08:22:31+08:00
draft: true
metaAlignment: center
clearReading: false
categories:
- 分类
- 技术文章
tags:
- Hugo
- LiveRe
---

Hugo内置了Disqus评论系统，Disqus虽然好用，但在大中华局域网下经常加载不出或者干脆不显示，多说也关了，准备用[搜狐畅言](https://changyan.kuaizhan.com/),但注册信息时需要网站的备案号，否则只能用15天，无奈放弃。后来无意在电影天堂还是啥网站看到他评论系统还不错，查了一下，是韩国的牌子[LiveRe(来必力)](https://livere.com/)，于是小折腾一下。

### 注册LiveRe

略过，将网站注册完成后到{{< hl-text cyan>}}管理页面->代码管理{{< /hl-text >}}中可以拿到{{< hl-text cyan>}}一般网站{{< /hl-text >}}的安装代码。里面有你的```data-uid```。

### 更改tranquilpeak主题文件
tranquilpeak原声支持Disqus，通过在```config.toml```中设置```disqusShortname```的方式可以方便的使用Disqus。所以这里也不添加新的config字段了，直接将```disqusShortname```设置为LiveRe的```data-uid```。但这样就完了么？答案是naive。我们还需要将主题中Disqus的安装代码替换掉，下面给出需要替换的文件内容。

#### layout\_default\single.html
评论部分的分区是在```<div id="post-footer">```中，其中有一段代码
```html
{{ if not (eq .Params.comments false) }}
    {{ if .Site.DisqusShortname }}
    <div id="lv-container" data-id="city" data-uid='{{ .Site.DisqusShortname }}'>
    {{ partial "post/disqus.html" . }}
    </div>
    {{ end }}
{{ end }}
```
这里是LiveRe插件是否启用的判断逻辑，如果设置了```disqusShortname```且开启了评论功能，那么就会加载LiveRe，```data-uid```即是配置在```config.toml```中的```disqusShortname```字段。

#### layout\partials\post\disqus.html

其实改无所谓，改只是为了显示起来更统一。
```html
<div id="livere_thread">
  <noscript>为正常使用来必力评论功能请激活JavaScript</noscript>
</div>
```

#### layout\partials\post\actions.html
这里面的```disqus_thread```也改为```livere_thread```，不然链接不到。

#### layout\partials\script.html
不好意思把重点放在了最后，这里才是加载```LiveRe```插件的脚本逻辑。
找到```{{ if .Site.DisqusShortname }}```,可以看到原来这里是配置Disqus的安装代码,我们需要做的就是把Disqus的代码替换为LiveRe的安装代码。注意不要把```<div id="lv-container" data-id="city" data-uid='{{ .Site.DisqusShortname }}'>```这行写在这里，不然界面布局会乱。改完后大概是这样
```html
{{ if .IsPage }}
  {{ if not (eq .Params.comments false) }}
    {{ if .Site.DisqusShortname }}
      <script type="text/javascript">
        (function(d, s) {
            var j, e = d.getElementsByTagName(s)[0];
            if (typeof LivereTower === 'function') { return; }
            j = d.createElement(s);
            j.src = 'https://cdn-city.livere.com/js/embed.dist.js';
            j.async = true;

            e.parentNode.insertBefore(j, e);
        })(document, 'script');
      </script>
    {{ end }}
  {{ end }}
{{ end }}
```
到这里，就已经把Disqus系统替换为LiveRe系统了。