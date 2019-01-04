---
title: "百度在线输入法Python代码"
date: 2014-01-18
lastmod: 2014-01-18
draft: false
tags: ["tech", "python", "xmbc"]
categories: ["tech"]
description: "写了一段python代码，用于xbmc，通过云输入方式支持xbmc中文输入"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

代码里面有一些xbmc的语句，可以忽略，脚本只是提供了一个基本逻辑，供参考。

``` python
    def __init__( self, *args, **kwargs ):
        self.totalpage = 0
        self.nowpage = 0
        self.words = ''
        self.py = ''
        self.bg = 0
        self.ed = 20
        self.wordpgs = []   #word page metadata
        self.inputString = kwargs.get("default") or ""
        self.heading = kwargs.get("heading") or ""
        self.fetchCookie()
        self.conn = httplib.HTTPConnection('olime.baidu.com')
        xbmcgui.WindowXMLDialog.__init__(self)

    def fetchCookie(self):
        req = urllib2.Request('http://ime.baidu.com/online.html')
        req.add_header('User-Agent', UserAgent)
        response = urllib2.urlopen(req)
        response.close()
        cdata = response.info().get('Set-Cookie')
        self.cookie = cdata.split(';')[0]
        if re.search('BAIDUID',self.cookie) is None: # just in case
            self.cookie='BAIDUID=5FF5CF0348C2CD89C15799855BBCB09A:FG=1'

    def changepages (self):
        self.getControl(CTRL_ID_HZLIST).setLabel('')
        hzlist = ''
        if not self.wordpgs: return
        if self.nowpage >= self.totalpage-1:
            self.bg += 20
            self.ed += 20
            self.getChineseWord(self.py, self.bg, self.ed)
            return
        curwpg = self.wordpgs[self.nowpage]
        for i, w in enumerate(self.words[curwpg[0]:curwpg[1]]):
            hzlist = hzlist + str(i) + '.' + w +' '
        if self.nowpage > 0:
            hzlist = '<' + hzlist
        if self.nowpage < self.totalpage-1:
            hzlist = hzlist + '>'
        self.getControl(CTRL_ID_HZLIST).setLabel(hzlist)

    def getChineseWord (self, py, bg=0, ed=20):
        self.getControl(CTRL_ID_HZLIST).setLabel('')
        if py=='': return
        if not bg:
            self.nowpage = 0
            self.totalpage = 0
            self.wordpgs = []
            self.words= []
            self.py = py
            self.bg = 0
            self.ed = 20
        wres = self.getwords(py, bg, ed)
        if wres:
            self.words.extend(wres)
            self.wordpgs = []
            inum = 0
            for s, w in enumerate(self.words):
                if len(''.join(self.words[inum:s+1]).decode('utf-8')) + (
                        s+1-inum)*2 >30:
                    self.wordpgs.append((inum, s))
                    inum = s
            if len(self.words) > inum:
                self.wordpgs.append((inum, len(self.words)))
            self.totalpage = len(self.wordpgs)
        else:
            self.nowpage = self.totalpage - 1

        hzlist = ''
        curwpg = self.wordpgs[self.nowpage]
        for i, w in enumerate(self.words[curwpg[0]:curwpg[1]]):
            hzlist = hzlist + str(i) + '.' + w +' '
        if self.nowpage > 0:
            hzlist = '<' + hzlist
        if self.nowpage < self.totalpage-1:
            hzlist = hzlist + '>'
        self.getControl(CTRL_ID_HZLIST).setLabel(hzlist)

    def getwords(self, py, bg, ed):
        t = time.time()
        urlsuf = '/py?input={0}&inputtype=py&bg={1}&ed={2}&{3}'.format(
            py, bg, ed,
            'result=hanzi&resultcoding=unicode&ch_en=0&clientinfo=web')
        try:
            self.conn.request('GET', urlsuf, headers={'Cookies':self.cookie,})
        except Exception as e:
            self.conn = httplib.HTTPConnection('olime.baidu.com')
            self.conn.request('GET', urlsuf, headers={'Cookies':self.cookie,})
        httpdata = self.conn.getresponse().read()
        words = []
        try:
            jsondata = simplejson.loads(httpdata)
        except ValueError:
            return None
        for word in jsondata[0]:
            words.append(word[0].encode('utf-8'))
        return words
```
