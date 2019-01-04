---
title: "用emacs写blog"
date: 2012-08-15
lastmod: 2012-08-15
draft: false
tags: ["tools", "emacs"]
categories: ["tech"]
description: "一篇杂记，简单介绍了如何基于github和jekyll搭建一个简单的blog，同时通过emacs和org来写blog，我个人认为这个认为这事情还是比较复杂，适合用emacs做编辑器的人和所有爱折腾的人民。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

断断续续写过一些blog，很多原因没有继续下去，当然自己懒是最主要的。总觉得写blog是一件很麻烦的事情：首先要找到合适的服务提供商，很多人都找一个免费的，不需要折腾嘛，直接注册就可以开写了，当然这也是有代价的，一般的免费服务即慢且广告到处漂，看的人大倒胃口，写起来也麻烦，如果纯文字的blog当然也无所谓，但咱们干技术的，多少还是需要贴些代码、图啥的，还需要在线去拷贝、粘贴，非常麻烦！当然你能在线写，但是那些个富文本框都能让人发狂，稍有不胜，前功尽弃，很打击人不是吗；另外如果我希望写东西不用一蹴而就，想写的时候就写点，这样编辑blog的功能就显得非常重要了，但是这类的功能你很难看到，即使有也很难用；当然有时候你对版面啥的不满意，希望整整，这同样也是一件X任务。

当然有些人喜欢自己折腾，自己买host，自己注册域名，用wordpress类似的产品搭建属于自己的blog，这个可控程度高点，一切都能自己控制，但是这个挺费精力的，你想我也就想写点东西，费力吧唧的折腾那些个乱七八糟的技术，不是自己的本意。当然，如果你想有一个自己专属的空间，这点代价还是值得的。

用上emacs之后，就开始寻思怎么用emacs来写blog，最好能在emacs里面写好，然后发布到blog空间，最好空间能有版本管理功能，我可以随时通过emacs对blog进行update，这样我就可以想到哪写到哪，在emacswiki上翻了翻，找到了orgmode和jekyll，翻了翻网上的文档，基本上确定就是他了，说起jekyll，这玩意还挺好玩，他就是个渲染引擎，自动把你的blog转换为html，写这东西的人是Tom Preston-Werner，这哥们同时也是github的创始人，他写了一篇blog，专门来说这事，为啥他要写jekyll，[[http://tom.preston-werner.com/2008/11/17/blogging-like-a-hacker.html][blogging like a hacker]]，感兴趣的建议看看。既然是自家人，jekyll和github集成自然就没问题了，这个就解决了blog的版本管理问题，非常不错吧。在github上搭建一个blog系统，广告自然是没有了，挺美的。参考了一些文章，最终实现了在emacs用orgmode写blog，然后通过magit git到github，在github上部署jekyll，这样一个个人weblog就完成了。

具体实现基本参考[[http://orgmode.org/worg/org-tutorials/org-jekyll.html][Using org to Blog with Jekyll]]

基本的流程如下：

先在自己电脑上建一个目录，比如就叫myblog，目录结构如下：

``` shell
---myblog
  ---org
     ---index.org
     ---_posts
        ---2012-07-06-title-name.org
  ---jekyll
```

jekyll目录直接从jekyllbootstrap check out下来的，你写文档就直接在/org/_posts写就可以了，写完之后可以通过M-x org-publish-all发布到jekyll目录下的_posts目录，然后按下面程序提交：

`M-x magit-status`

s键添加提交对象，之后通过c键写注释，C-c C-c提交，P键push，更新就提交到github了。

如果你碰到提交的页面没有生效， 一般都是在github上page build失败导致的，你可以在本地调试，关闭_config.yml中的auto，设置为false，然后就可以在本地看到page build失败的异常，修复即可。

另外，jekyll只能渲染body，不能包括head，所以我们在配置的时候需要设置orgmode只能包括body：
`body-only t`
但是这样的话，orgmode就不能导出toc和章节编号，我自己对lisp不熟悉，就粗暴的修改了org-html.el：
先编辑下面行：
`       (if (and org-export-with-toc (not body-only))`
修改为：
`      (if (and org-export-with-toc)`
然后修改下面行：
` 	(if (and num (not body-only))`
修改为：
`	(if (and num)`

下面是我的emacs配置，供感兴趣的同学参考：

``` emacs-lisp
;;org-mode
(add-to-list 'load-path "~/.emacs.d/lisp/org-7.8.11/lisp")
(require 'org-install)
(add-to-list 'auto-mode-alist '("\\.org$" . org-mode))
(define-key global-map "\C-cl" 'org-store-link)
(define-key global-map "\C-ca" 'org-agenda)
(global-set-key "\C-cb" 'org-iswitchb)
(global-font-lock-mode 1)                     ; for all buffers
(add-hook 'org-mode-hook 'turn-on-font-lock)  ; Org buffers only
(setq org-startup-indented t)
(setq org-log-done t)
(require 'org-publish)
(setq org-publish-project-alist '(
  ("org-myblog"
          ;; Path to your org files.
          :base-directory "~/myblog/org/"
          :base-extension "org"

          ;; Path to your Jekyll project.
          :publishing-directory "~/myblog/jekyll/"
          :recursive t
          :publishing-function org-publish-org-to-html
          :headline-levels 4
          :html-extension "html"
          :body-only t ;; Only export section between <body> </body>
    )

    ("org-static-myblog"
          :base-directory "~/myblog/org/"
          :base-extension "css\\|js\\|png\\|jpg\\|gif\\|pdf\\|mp3\\|ogg\\|swf\\|php"
          :publishing-directory "~/myblog/jekyll/"
          :recursive t
          :publishing-function org-publish-attachment)

    ("myblog" :components ("org-myblog" "org-static-myblog"))
))

;; turn on soft wrapping mode for org mode
(add-hook 'org-mode-hook 'turn-on-visual-line-mode)
(add-hook 'message-mode-hook 'turn-on-orgstruct)
(add-hook 'message-mode-hook 'turn-on-orgstruct++)

(add-to-list 'load-path "~/.emacs.d/lisp/magit")
(require 'magit)
```
