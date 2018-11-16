---
title: hexo集成评论系统Gitalk
tags: gitalk
categories: hexo
---

#   简述
当初搭建hexo的时候想要使用内置的一些聊天系统，比如畅言啊,disqus等，但是都不太理想，后来由于不怎么更新就不在意了。  
最近又回来了，打算以后多写一些碰到的问题，学到的东西，或者生活~  所以打算接一个评论系统，考虑再三决定使用Gitalk。

<!--more-->

#   为什么使用gitalk

*   简单

#   怎么加入

*   js引入
    首先在github新建一个Application，即在github头像下的setting-> developer settings->register a new application

    Application name： comment //随便写
    Homepage URL：http://blog.duanshaojie.cn //博客地址
    Application description： 博客评论 //
    Authorization callback URL：http://blog.duanshaojie.cn

*   获取了Client ID和Client Secret之后，开始引入  
    先添加配置，我的主题是themes,所以在这下面的配置文件中添加如下,
    这里需要注意以下，我前面一直把repo填成上面新建的application name的名字，导致Error: NOT Found，这里应该填为你博客的仓库
    ```
    #gitalk settings
    gitalk:
    enable: true 
    #github 用户名吧,
    owner: duanshaojie
    #hexo在github的仓库
    repo: github.duanshaojie.io 
    #Github 用户名,
    admin: duanshaojie 
    #`Github Application clientID`
    clientID: 83135870a07a6d067622 
    #`Github Application clientSecret`
    clientSecret: eaefd8f7b31afd0d58b8439ea9d1f10ced93278b
    ```
    然后将下面代码放入themes\next\layout\_partials\comments.swig里面
    ```
    {% if theme.gitalk.enable %}
        <link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
        <div id="gitalk-container"></div>
        <script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>
        <script type="text/javascript">
            var gitalk = new Gitalk({
                clientID: '{{ theme.gitalk.clientID }}',
                clientSecret: '{{ theme.gitalk.clientSecret }}',
                id: window.location.pathname,
                repo: '{{ theme.gitalk.repo }}',
                owner: '{{ theme.gitalk.owner }}',
                admin: '{{ theme.gitalk.admin }}'
            })
            gitalk.render('gitalk-container')
        </script>
    {% endif %}
    ```

#   最后
*   没有最后咯，hexo g -d 去感受下把

#   错误解决 
*   Error: Not Found
    弄完以上之后，我进入评论的时候一直报这个错误，找来找去没发现原因，在一个偶然的地方看到貌似是说repo错误，后来改成了博客的仓库名就好了
*   Error: Validation Failed
    这里是因为提交评价的时候，github限制了labal参数的长度不能超过50，这个错误网上资料很多，我看到一个比较简单的说说
    1.创建一个js文件（mad.js）
    ```
        ! function(n) {
        "use strict";
        function t(n, t) {
            var r = (65535 & n) + (65535 & t);
            return (n >> 16) + (t >> 16) + (r >> 16) << 16 | 65535 & r
        }
        function r(n, t) {
            return n << t | n >>> 32 - t
        }
        function e(n, e, o, u, c, f) {
            return t(r(t(t(e, n), t(u, f)), c), o)
        }
        function o(n, t, r, o, u, c, f) {
            return e(t & r | ~t & o, n, t, u, c, f)
        }
        function u(n, t, r, o, u, c, f) {
            return e(t & o | r & ~o, n, t, u, c, f)
        }
        function c(n, t, r, o, u, c, f) {
            return e(t ^ r ^ o, n, t, u, c, f)
        }
        function f(n, t, r, o, u, c, f) {
            return e(r ^ (t | ~o), n, t, u, c, f)
        }
        function i(n, r) {
            n[r >> 5] |= 128 << r % 32, n[14 + (r + 64 >>> 9 << 4)] = r;
            var e, i, a, d, h, l = 1732584193,
                g = -271733879,
                v = -1732584194,
                m = 271733878;
            for (e = 0; e < n.length; e += 16) i = l, a = g, d = v, h = m, g = f(g = f(g = f(g = f(g = c(g = c(g = c(g = c(g = u(g = u(g = u(g = u(g = o(g = o(g = o(g = o(g, v = o(v, m = o(m, l = o(l, g, v, m, n[e], 7, -680876936), g, v, n[e + 1], 12, -389564586), l, g, n[e + 2], 17, 606105819), m, l, n[e + 3], 22, -1044525330), v = o(v, m = o(m, l = o(l, g, v, m, n[e + 4], 7, -176418897), g, v, n[e + 5], 12, 1200080426), l, g, n[e + 6], 17, -1473231341), m, l, n[e + 7], 22, -45705983), v = o(v, m = o(m, l = o(l, g, v, m, n[e + 8], 7, 1770035416), g, v, n[e + 9], 12, -1958414417), l, g, n[e + 10], 17, -42063), m, l, n[e + 11], 22, -1990404162), v = o(v, m = o(m, l = o(l, g, v, m, n[e + 12], 7, 1804603682), g, v, n[e + 13], 12, -40341101), l, g, n[e + 14], 17, -1502002290), m, l, n[e + 15], 22, 1236535329), v = u(v, m = u(m, l = u(l, g, v, m, n[e + 1], 5, -165796510), g, v, n[e + 6], 9, -1069501632), l, g, n[e + 11], 14, 643717713), m, l, n[e], 20, -373897302), v = u(v, m = u(m, l = u(l, g, v, m, n[e + 5], 5, -701558691), g, v, n[e + 10], 9, 38016083), l, g, n[e + 15], 14, -660478335), m, l, n[e + 4], 20, -405537848), v = u(v, m = u(m, l = u(l, g, v, m, n[e + 9], 5, 568446438), g, v, n[e + 14], 9, -1019803690), l, g, n[e + 3], 14, -187363961), m, l, n[e + 8], 20, 1163531501), v = u(v, m = u(m, l = u(l, g, v, m, n[e + 13], 5, -1444681467), g, v, n[e + 2], 9, -51403784), l, g, n[e + 7], 14, 1735328473), m, l, n[e + 12], 20, -1926607734), v = c(v, m = c(m, l = c(l, g, v, m, n[e + 5], 4, -378558), g, v, n[e + 8], 11, -2022574463), l, g, n[e + 11], 16, 1839030562), m, l, n[e + 14], 23, -35309556), v = c(v, m = c(m, l = c(l, g, v, m, n[e + 1], 4, -1530992060), g, v, n[e + 4], 11, 1272893353), l, g, n[e + 7], 16, -155497632), m, l, n[e + 10], 23, -1094730640), v = c(v, m = c(m, l = c(l, g, v, m, n[e + 13], 4, 681279174), g, v, n[e], 11, -358537222), l, g, n[e + 3], 16, -722521979), m, l, n[e + 6], 23, 76029189), v = c(v, m = c(m, l = c(l, g, v, m, n[e + 9], 4, -640364487), g, v, n[e + 12], 11, -421815835), l, g, n[e + 15], 16, 530742520), m, l, n[e + 2], 23, -995338651), v = f(v, m = f(m, l = f(l, g, v, m, n[e], 6, -198630844), g, v, n[e + 7], 10, 1126891415), l, g, n[e + 14], 15, -1416354905), m, l, n[e + 5], 21, -57434055), v = f(v, m = f(m, l = f(l, g, v, m, n[e + 12], 6, 1700485571), g, v, n[e + 3], 10, -1894986606), l, g, n[e + 10], 15, -1051523), m, l, n[e + 1], 21, -2054922799), v = f(v, m = f(m, l = f(l, g, v, m, n[e + 8], 6, 1873313359), g, v, n[e + 15], 10, -30611744), l, g, n[e + 6], 15, -1560198380), m, l, n[e + 13], 21, 1309151649), v = f(v, m = f(m, l = f(l, g, v, m, n[e + 4], 6, -145523070), g, v, n[e + 11], 10, -1120210379), l, g, n[e + 2], 15, 718787259), m, l, n[e + 9], 21, -343485551), l = t(l, i), g = t(g, a), v = t(v, d), m = t(m, h);
            return [l, g, v, m]
        }

        function a(n) {
            var t, r = "",
                e = 32 * n.length;
            for (t = 0; t < e; t += 8) r += String.fromCharCode(n[t >> 5] >>> t % 32 & 255);
            return r
        }

        function d(n) {
            var t, r = [];
            for (r[(n.length >> 2) - 1] = void 0, t = 0; t < r.length; t += 1) r[t] = 0;
            var e = 8 * n.length;
            for (t = 0; t < e; t += 8) r[t >> 5] |= (255 & n.charCodeAt(t / 8)) << t % 32;
            return r
        }

        function h(n) {
            return a(i(d(n), 8 * n.length))
        }

        function l(n, t) {
            var r, e, o = d(n),
                u = [],
                c = [];
            for (u[15] = c[15] = void 0, o.length > 16 && (o = i(o, 8 * n.length)), r = 0; r < 16; r += 1) u[r] = 909522486 ^ o[r], c[r] = 1549556828 ^ o[r];
            return e = i(u.concat(d(t)), 512 + 8 * t.length), a(i(c.concat(e), 640))
        }

        function g(n) {
            var t, r, e = "";
            for (r = 0; r < n.length; r += 1) t = n.charCodeAt(r), e += "0123456789abcdef".charAt(t >>> 4 & 15) + "0123456789abcdef".charAt(15 & t);
            return e
        }
        function v(n) {
            return unescape(encodeURIComponent(n))
        }
        function m(n) {
            return h(v(n))
        }
        function p(n) {
            return g(m(n))
        }
        function s(n, t) {
            return l(v(n), v(t))
        }
        function C(n, t) {
            return g(s(n, t))
        }
        function A(n, t, r) {
            return t ? r ? s(t, n) : C(t, n) : r ? m(n) : p(n)
        }
        "function" == typeof define && define.amd ? define(function() {
            return A
        }) : "object" == typeof module && module.exports ? module.exports = A : n.md5 = A
    }(this);
    ```
    2.然后将上面的comments中的内容，增加mda的引用，将id进行md5减小长度，然后再发布试试？
    ```
    {% if theme.gitalk.enable %}
        <link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
        <div id="gitalk-container"></div>
        <script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>
        <script src="/js/src/md5.js"></script>
        <script type="text/javascript">
            var gitalk = new Gitalk({
                clientID: '{{ theme.gitalk.clientID }}',
                clientSecret: '{{ theme.gitalk.clientSecret }}',
                <!-- id: window.location.pathname, -->
                id: md5(location.pathname),
                repo: '{{ theme.gitalk.repo }}',
                owner: '{{ theme.gitalk.owner }}',
                admin: '{{ theme.gitalk.admin }}'
            })
            gitalk.render('gitalk-container')
        </script>
    {% endif %}
    ```

#   扩展留言薄
*   在主题下的_comfig.yml下找到如下菜单信息，增加 guestbook: /guestbook
```
menu:
  home: /
  categories: /categories/
  about: /about/
  archives: /archives/
  tags: /tags/
  guestbook: /guestbook
```

*   给你增加的菜单添加名称，修改主题下的languages\zh-Hans.yml,增加guestbook的名称
```
menu:
  home: 首页
  archives: 归档
  categories: 分类
  tags: 标签
  about: 关于
  search: 搜索
  schedule: 日程表
  sitemap: 站点地图
  commonweal: 公益404
  guestbook: 留言
```

*   最后在根目录下的source下git bash here,新建page
```
hexo new page "guestbook"
```

*   再部署下试试？