---
title: "emacs notes"
date: 2013-09-28
lastmod: 2019-01-04
draft: false
tags: ["tech", "emacs", "notes"]
categories: ["tech"]
description: "一些使用的小技巧，碰到的一些问题解决方法记录，方便查阅"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# EMACS
有任何操作问题，建议参考[Xha Emacs](http://ergoemacs.org/index.html)，我有操作问题一般都能在这里找到答案。

# Display
## too long line highlight

``` shell
   (setq whitespace-style '(lines))
   (setq whitespace-line-column 78)
   (global-whitespace-mode 1)
```

# Macro

``` shell
| key sequence | comment                       |
|--------------+-------------------------------|
| F3           | start record macro            |
| F4           | end record macro              |
| C-x (        | start record macro            |
| C-x )        | end record macro              |
| C-x e        | execute last macro            |
| C-u 10 C-x e | execute last macro repeat 100 |
|              |                               |
```


# Navigate

``` shell
| key sequence | comment               |
|--------------+-----------------------|
| C-x r m      | add file to bookmarks |
| C-x r l      | open bookmarks        |
|              |                       |
```

# Copy & Paste
| key sequence  | comment           |
|---------------+-------------------|
| C-S-Backspace | delete whole line |
|               |                   |

# Coding

``` shell

| key sequence | comment        |
|--------------+----------------|
| M-;          | comment coding |
|              |                |
```

# Replace
替换字符串有很灵活的方式，比如我需要替换如下字符串组用序列数字：
``` shell
str01
str01
str01
str01
```
替换成
``` shell
str1
str2
str3
str4
```
执行如下键序列：
:  :M-x replace-regexp
:  在"Replace regexp:"提示符下输入“01”
:  在"Replace regexp 01 with:"提示符下输入“\,(1+ \#)”

# Common #
## LISP
[Read Lisp, Tweak Emacs: How to read Emacs Lisp so that you can customize Emacs](http://emacslife.com/how-to-read-emacs-lisp.html)
### 常见错误
1. `Lisp error: (void-function …)`, 调用了没有定义的function
2. `Symbol's value as variable is void: _`, 变量没有定义
## Emacs Daemon ##

### 通过systemd启动一个emacs daemon ###

1.  在你的Home目录下创建文件: `~/.config/systemd/user/emacs.service`, 注意这里的Environment, 如果不设置的话, 中文输入法不能激活(这几个环境变量一般我们是写在xprofile里面, 系统登入之后才生效, 之前启动的程序无法看到)

    ```
    [Unit]
    Description=Emacs: the extensible, self-documenting text editor

    [Service]
    Type=forking
    Environment=GTK_IM_MODULE=ibus XMODIFIERS=@im=ibus QT_IM_MODULE=ibus
    ExecStart=/usr/bin/emacs --daemon
    ExecStop=/usr/bin/emacsclient --eval "(kill-emacs)"
    Restart=always

    [Install]
    WantedBy=default.target
    ```

2.  Enable该服务: `systemctl --user enable emacs`

3.  启动该服务: `systemctl --user start emacs`

### 启动Emacs ###

通过命令启动emacs: `emacsclient -c`

注意，通过这种方式启动Emacs，字体需要通过default-frame-list设置：
`(add-to-list 'default-frame-alist '(font . "Inconsolata-12"))`

## themes ##
### powerline ##
(require 'powerline)
(powerline-default-theme)

## Package Manager / ELPA
### 参考
http://ergoemacs.org/emacs/emacs_package_system.html
### 升级包
M-x package-list-packages RET U x
## Edit ##
### abbrev

```
;; turn on abbrev mode globally
(setq-default abbrev-mode t)
```

abbrev允许你自定义字符串缩写, 例如: 我的Kubernetes项目GOPATH非常长, 输入很痛苦, 我希望减少输入, 通过`C-x a g`定义global abbrev, 这样下次就能通过`M-/`自动扩展:

Define Abbrev

Suppose you want to define “bg” → “background”.

Type “background”.
call add-global-abbrev 【Ctrl+x a g】, in the prompt, type “bg”.
Now, when you type “bg” followed by a space or return, it'll expand to “background”.


If the expanded text is more than one word, for example, suppose you want to define “faq” → “frequently asked questions”.

Type “frequently asked questions”.
Select the text.
Call universal-argument 【Ctrl+u】, type “0”.
Call add-global-abbrev 【Ctrl+x a g】, in the prompt, type “faq”.


```
/home/tacy/workspace/go/kubernetes/src/k8s.io/kubernetes/Godeps/_workspace/:/home/tacy/workspace/go/kubernetes/src/
C-x a g
kroot

M-x setenv RET GOPATH RET kroot M-/
```

列出已经定义的abbrev: M-x list-abbrev RET

其他相关参考:[https://www.emacswiki.org/emacs/AbbrevMode](Abbrev Mode)


### Common ###
 - diff buffer & file: `M-x diff-buffer-with-file` or `C-x s d`
 - reload buffer: `M-x revert-buffer`
 - convert tab to whitespace: 选择range, 然后执行`M-x untabify`, 相反操作执行`tabify`.  [Converting between tabs and whitespace](https://www.masteringemacs.org/article/converting-tabs-whitespace)

### Region ###
激活Region通过: `C-SPC`

```
Kill it with C-w (see Killing).
Copy it to the kill ring with M-w (see Yanking).
Convert case with C-x C-l or C-x C-u (see Case).
Undo changes within it using C-u C-/ (see Undo).
Replace text within it using M-% (see Query Replace).
Indent it with C-x TAB or C-M-\ (see Indentation).
Fill it as text with M-x fill-region (see Filling).
Check the spelling of words within it with M-$ (see Spelling).
Evaluate it as Lisp code with M-x eval-region (see Lisp Eval).
Save it in a register with C-x r s (see Registers).
Save it in a buffer or a file (see Accumulating Text).
```
### Rectangles ###
激活Retangles通过: `C-x SPC`

```
C-x r o
Insert blank space to fill the space of the region-rectangle (open-rectangle). This pushes the previous contents of the region-rectangle to the right.
C-x r N
Insert line numbers along the left edge of the region-rectangle (rectangle-number-lines). This pushes the previous contents of the region-rectangle to the right.
C-x r c
Clear the region-rectangle by replacing all of its contents with spaces (clear-rectangle).
M-x delete-whitespace-rectangle
Delete whitespace in each of the lines on the specified rectangle, starting from the left edge column of the rectangle.
C-x r t string RET
Replace rectangle contents with string on each line (string-rectangle).
M-x string-insert-rectangle RET string RET
Insert string on each line of the rectangle.

Kill a rectangle: kill-rectangle ‘C-x r k’
Yank the last rectangle killing: yank-rectangle ‘C-x r y’
Insert a text rectangle: string-insert-rectangle (unbound)
Insert a text to replace rectangle region: string-rectangle ‘C-x r t’
Delete the selected rectangle: delete-rectangle ‘C-x r d’
Insert a whitespace rectangle into the region: open-rectangle ‘C-x r o’
Number lines in rectangle: number-to-register ‘C-x r N’. This command also inserts a column of spaces after numbers.
```

### Simpleclip ###

插件提供Copy/Paste功能，具体参考[CopyAndPaste](http://www.emacswiki.org/emacs/CopyAndPaste#toc4)
该插件提供和系统Clipboard之间的交互，不影响Emacs内部Kill/Yank行为

Copy to Clipboard: `S-c`, Paste from Clipboard: `S-v`

## Navigation ##
### Dired ###
```
^            dired-up-directory
<remap>      dired-toggle-read-only                  切换到编辑模式后，可以对文件名做批量操作，C-c C-c提交
%-m          dired-mark-files-regexp                 mark file by regex
Q            dired-do-find-regexp-and-replace        在所有mark文件里面，做查找替换
```

### Helm ###
[http://tuhdo.github.io/helm-intro.html](helm intro)
缺省前缀`C-x c`, 导航代码利器.

### helm-find-file
查找文件, 我绑定在`C-x C-f`, 进入helm find file session之后, 你能做很多复杂的事情:
### recurive find file
`C-u C-x C-f`
#### grep
grep当前buffer目录下的文件: `C-s`会对当前buffer进行grep, `. C-s`会对当前目录进行匹配, 注意不会查找子目录, `C-u C-s`会对子目录也匹配
##### edit grep result
先保存grep result, by `C-x C-s`, 然后通过wgrep插件的`C-c C-p`进入编辑模式, `C-c C-e`应用修改, 注意buffer并没有保存到磁盘.

### Registor ###

```
C-x r SPC r
Record the position of point and the current buffer in register r (point-to-register).
C-x r j r
Jump to the position and buffer saved in register r (jump-to-register).
```

### Bookmark ###

```
C-x r l    bookmark menu list
C-x r m    bookmark mark
C-x r b    bookmark jump
```

### Neotree ###

插件提供树状目录浏览文件。

打开目录树通过neotree实现：`C-c n`
显示隐藏文件：`A`
指定根目录：`M-x neotree-dir`

# Dev #

## Go ##
扩展Emacs支持golang开发, 实现语法高亮, 代码提示, 代码跳转, 语法检查和代码风格检查, 代码接口实现查找, 代码调试.

主模式采用[go-mode](https://github.com/dominikh/go-mode.el)

代码跳转依赖godef, 代码提示依赖gocode(company-mode+company-go), 代码接口实现查找依赖guru(go-guru), 代码调试依赖dlv(go-dlv), 代码风格和语法检查依赖flycheck

### Config ###
首先你需要安装go-mode: `(require 'go-mode)`.

同时安装godef, 代码跳转依赖它: `go get -u github.com/rogpeppe/godef`

#### auto complete ####

首先说说gocode, 因为代码提示功能由后台运行的进程gocode实现, 你需要先安装gocode: `go get -u github.com/nsf/gocode`.

需要特别注意, gocode只会分析编译之后的代码(pkg目录下的包), 不会分析source code, 所以你确保代码被编译过才会提示

gocode采用server/client模式, gocode会启动一个后台进程监听client请求, 可以手动启动gocode来做调试: `gocode -s -debug`.

默认情况下gocode只会搜索$GOROOT/pkg和$GOPATH/pkg两个地方, 如果你的编译结果不在这两个地方, 需要手动添加, 例如我为kubernetes项目设置自定义lib-path:

`gocode set lib-path "/home/tacy/workspace/go/kubernetes/src/k8s.io/kubernetes/Godeps/_workspace:/home/tacy/workspace/go/kubernetes/src/k8s.io/kubernetes/_output/local/go"`

注意, 路径不要包括pkg这一层.

gocode的配置文件存放在: `~/.config/gocode/config.json`, 可以通过`gocode set`查看所有配置项.

gocode支持当代码修改后, 自动编译, 获取最新的代码提示, 这个功能处于实验阶段:

```
gocode set autobuild true
```

配置插件company-mode + company-go:

```
;; autocompletion with company-mode
(add-hook 'go-mode-hook 'company-mode)
(add-hook 'go-mode-hook (lambda ()
                          (set (make-local-variable 'company-backends)
                               '(company-go))
                          (company-mode)))
```


#### syntax & style check ####
flycheck缺省支持go, 提供下面几个checker:

```
go-gofmt (syntax check with gofmt)
go-golint (coding style with Golint)
go-vet (check for suspicious code with go tool vet)
go-build or go-test (syntax and type check with Go, for source and tests respectively)
go-errcheck (check for unhandled error returns with errcheck)
```

lint, errcheck需要安装:

```
go get -u github.com/kisielk/errcheck
go get -u github.com/golang/lint/golint
```

然后激活flycheck即可: `(add-hook 'go-mode-hook 'flycheck-mode)`

但是需要注意, go-build和go-test这两个checker对于大项目不适用, 一个`go-build `就直接歇菜了, 何况实时检测.

简单的解决方法就是禁用go-build和go-test, 另外我也建议禁用errcheck(或者直接不安装errcheck这个包):

```
;; (add-hook 'go-mode-hook 'flycheck-mode)
(add-hook 'go-mode-hook (lambda ()
                          (setq flycheck-disabled-checkers
                               '(go-build go-test go-errcheck))
                          (flycheck-mode)))
```


#### oracle ####

* oracle 已经废弃, 使用guru

此oracle非彼oracle, 这是go的一个语法分析工具, 强大到没朋友, 下面是它提供的功能:

```
C-c C-o <       go-oracle-callers
C-c C-o >       go-oracle-callees
C-c C-o c       go-oracle-peers
C-c C-o d       go-oracle-definition
C-c C-o f       go-oracle-freevars
C-c C-o g       go-oracle-callgraph
C-c C-o i       go-oracle-implements
C-c C-o p       go-oracle-pointsto
C-c C-o r       go-oracle-referrers
C-c C-o s       go-oracle-callstack
C-c C-o t       go-oracle-describe
```

用它来阅读代码非常方便, 简直就是go版的cscope, 配置也很简单, 首先安装oracle: `go get -u golang.org/x/tools/cmd/oracle`, 然后配置emacs:
```
(Load-file "/Users/tacy/workspace/go/go-tools/src/golang.org/x/tools/cmd/oracle/oracle.el")
(setq exec-path (cons "/Users/tacy/workspace/go/go-tools/bin" exec-path))
(add-to-list 'exec-path "/Users/tacy/workspace/go/kubernetes/bin")
```

#### guru ####

```
       (load-file "/home/tacy/workspace/go/go-tools/src/golang.org/x/tools/cmd/guru/go-guru.el")
       (setq exec-path (cons "/home/tacy/workspace/go/go-tools/bin" exec-path))
       (setenv "PATH" (concat "/home/tacy/workspace/go/go-tools/bin:" (getenv "PATH")))
```

M-x go-guru-set-scope k8s.io/heapster/metrics/heapster


### Usage ###

#### Navigation ####

```
M-. (godef-jump)
M-* (pop-tag-map)
C-c C-d (godef-describe)
M-x compile
```

#### godep ####

使用godep的项目在操作的时候需要在前面加godep, 例如`go build` -> `godep go build`, 你也可以修改GOPATH: `M-x setenv RET GOPATH RET ``godep path``:$GOPATH`, 这样就不用在go前面加godep了.

#### govendor ####
添加依赖包, 如果包里面有多个文件(注意后面的三个点):

```
govendor add golang.org/x/crypto/acme/autocert/...

go get -v -u golang.org/x/sys/...
```


#### oracle ####

强大的Go代码分析工具，首先设置分析范围：`M-x go-oracle-set-scope`, 例如`k8s.io/kubernetes/cmd/kubelet`(特别注意, 分析范围最好指向含main方法的类. 否则很多功能没法使用, 例如: go-oracle-callstack, go-oracle-callers, go-oracle-callees等), 然后参考帮助操作。

```
<f5>            go-oracle-describe
<f6>            go-oracle-referrers
```

需要注意godep带来的问题.

下面举个例子， 我希望分析kubelet这个包的代码, 首先我要做的是指定GOPATH:

`M-x setenv RET GOPATH RET /home/workspace/go/kubernetes/src/k8s.io/kubernetes/Godeps/_workspace:/home/workspace/go/kubernetes`

然后是指定oracle的分析范围:

`M-x go-oracle-set-scope RET k8s.io/kubernetes/cmd/kubelet`

查看interface方法实现: `C-c C-o i`, 其他都在`C-c C-o`这个组里面定义.

需要特别注意的是, 如果需要分析别的包, 务必要设置scope.

#### Playground ####

```
`go-play-buffer` and `go-play-region` send code to the Playground
`go-download-play` to download a Playground entry into a new buffer
```

## Python ##
emacs默认自带python-mode. 但是功能有限, 可以采用elpy实现扩展, 实现一些ide功能, 主要包括代码提示(autocomplete / company+jedi), 语法检查(flymake / flycheck), 代码规范(autopep8 / yapf)

我选用的组合是elpy+company+jedi+flycheck+autopep8, 本来打算用yapf替代autopep8, 但是elpy当前的版本(1.10)还不支持, 需要等1.11 release.

### Config ###

#### Elpy config ####

可以通过`M-x elpy-config`查看elpy配置情况, elpy需要一些依赖包支持, 包括jedi, importmagic, autopep8, flakes8. 确认这些都已经安装.

参考[Elpy](https://elpy.readthedocs.org/en/latest/ide.html)

#### flake8 ####
elpy缺省语法检查用的是flymake, 你需要安装flycheck插件, 并用下面配置替换flymake:

```
;; use flycheck not flymake
(when (require 'flycheck nil t)
  (setq elpy-modules
        (delq 'elpy-module-flymake elpy-modules))
  (add-hook 'elpy-mode-hook 'flycheck-mode))
```

flake8行为可以通过配置文件自定义:
```
$cat ~/.config/flake8
[flake8]
ignore = E221,E501,E203,E202,E272,E251,E211,E222,E701
max-line-length = 160
exclude = tests/*
```

#### company + jedi ####
安装company插件和company-jedi插件, 通过下面配置激活:

```
;; autocompletion with company-mode
(add-hook 'python-mode-hook 'company-mode)
(add-hook 'python-mode-hook (lambda ()
                              (set (make-local-variable 'company-backends)
                                   '(company-jedi company-yasnippet))
                          (company-mode)))
```
这里的company backend只用了company-jedi和company-yasnippet, 其他可根据自己喜好添加, 具体可以参考[company-mode](http://company-mode.github.io/).


### Usage ###

#### Virtualenv ####

```
M-x pyven-activate
M-x pyven-deactivate
```

#### Navigation ####

```
M-.             (elpy-goto-definition)
M-*             (pop-tag-mark)
C-c C-d         (elpy-doc)
C-c C-j         imenu
```

#### Project ####

```
M-x elpy-set-project-root
C-c C-f (elpy-find-file)
C-c C-s (elpy-rgrep-symbol)
```

#### Edit ####

```
C-c <           python-indent-shift-left
C-c >           python-indent-shift-right
M-;             comment region
```

#### Refactoring ####

```
C-c C-e (elpy-multiedit-python-symbol-at-point)
C-c C-r f (elpy-format-code)
C-c C-r i (elpy-importmagic-fixup)
C-c C-r r (elpy-refactor)
```


## Markdown ##
### Edit ###
[markdown](https://github.com/jrblevin/markdown-mode)的帮助写的很好, 建议仔细阅读, 如果不知道怎么用, 直接`C-c C-h`. 比如说`C-c C-a`这个prefix的绑定都是和链接相关, 对应到html的a标签, 很好理解.

`C-c C-s P`插入代码块

### 预览 ###

直接调用浏览器预览结果通过`C-c C-c P`实现

注意需要的是, 缺省markdown-command使用的命令是markdown, 该命令无法正确显示GFM的md: 例如代码块显示等，可以使用[marked](https://github.com/chjj/marked).

`[tacy@tacyArch ~]$ npm install -g marked`

然后需要配置[markdown mode](bhttps://github.com/jrblevin/markdown-mode)的参数markdown-command：

`(setq markdown-command "marked")`

### Tips ###

内嵌代码通过`C-c C-s P`, 可以指定语言
Promote subtree: `C-M-LEFT`


## JavaScript ##
使用js2-mode, 语法校验用flycheck, 代码格式化使用js-beautify, 需要全局安装组件: `npm install -g eslint babel-eslint eslint-cli js-beautify`

需要的emacs lisp: js2-mode, web-beautify
### 配置 ###

``` emacs-lisp
(add-to-list 'auto-mode-alist '("\\.js$" . js2-mode))

;; js2-mode

;; Change some defaults: customize them to override
(setq-default js2-basic-offset 2
	          js2-highlight-level 3
              js2-bounce-indent-p nil)

(add-hook 'js2-mode-hook (lambda ()
			   (setq-default js2-mode-show-parse-errors nil
					 js2-mode-show-strict-warnings nil)
			   (flycheck-mode)
			   (js2-imenu-extras-setup)
			   ))

;; js-mode
(setq-default js-indent-level 2)      ;;缩进
```


## VIM
从桌面拷贝内容到VI，会出现indent错误（缩进错误），需要设置`:set noai`

# Orgmode
## add footnotes
使用快捷键C-c C-x f添加注脚，使用C-c C-l添加URL链接，使用C-c C-c在footnotes和
正文之间跳转，使用C-c C-o在浏览器打开URL连接
## 单行注释
: : 注视内容
## 添加注释块
: <s <TAB>
