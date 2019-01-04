---
title: "python notes"
date: 2013-07-23
lastmod: 2019-01-04
draft: false
tags: ["tech", "python", "notes"]
categories: ["tech"]
description: "python学习点滴"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# enviroment
## virtualenv
``` shell
virtualenv myenv
source .myenv/bin/activate
deactivate
```


# Core type
  - integer, string, tuple is immutable, can't by change in-place
  - keep in mind that the mutable type is share reference, if you need to change
    them
  - You can copy sequence type by slice, but it only copy top-level, if you want
    to completely copy the object, use deepcopy
    #+BEGIN_SRC python
    >>>x = [1, 2, [3, 4]]
    >>>y = ['aa', x[:]]
    >>>x[0] = 11
    >>>x[2][0] = 33
    >>>x
    [11, 2, [33, 4]]
    >>>y
    ['aa', [1, 2, [33, 4]]
    #+END_SRC
  - comprehensions
    : [x*2 for x in range(10)]
  - sorted
    sorted dict by value
    #+BEGIN_SRC python
    #method 1
    sorted(d.iteritems(), key=lambda x: x[1])   #d.items() return list
    #method 2
    import operator
    x = {1: 2, 3: 4, 4:3, 2:1, 0:0}
    sorted_x = sorted(x.iteritems(), key=operator.itemgetter(1))
    #+END_SRC
  - pickle & struct
    object serialization & C structs represented as Python strings


# String
## python中的字符串
   首先明确一下，下面讲的是Python2，在Python2中的string分为两个类型:str类型和unicode类型. str类型其实就是一个字节数组,
   做下面一个简单测试就知道了：
   #+BEGIN_SRC python
   In [176]: s = '测试'

   In [177]: s
   Out[177]: '\xe6\xb5\x8b\xe8\xaf\x95'

   In [178]: s[0]
   Out[178]: '\xe6'

   In [179]: len(s)
   Out[179]: 6

   In [180]: type(s)
   Out[180]: str
   #+END_SRC
   从上面可以看到，'测试'这个String其实表达的是一个6字节的数组, 类型是str, 可以
   直接通过下标访问每个字节, 它的长度是6.

   Unicode类型是真正语义上的字符串, 继续做下面一个测试:
   #+BEGIN_SRC python
   In [181]: u = u'测试'

   In [182]: len(u)
   Out[182]: 2

   In [183]: type(u)
   Out[183]: unicode

   In [184]: u[0]
   Out[184]: u'\u6d4b'

   In [185]: u
   Out[185]: u'\u6d4b\u8bd5'
   #+END_SRC

   看到不一样了吗, 你通过u'这个标示告诉Python这是一个Unicode字符串, Unicode
   字符串长度是2, 类型是Unicode. 这里的长度不代表字节数(6d4b无法用一个字节表示), 而是代表字符个数, 这和str类型的string不一样.
   要搞明白这两个字符串类型的关系, 我们需要先来搞明白unicode和encode之间的关系.

## unicode & encode
这里两个概念不要搞混淆了，unicode是用一组code points表示字符序列，codepoints就
是一个整形，比如26446，代表‘李’，在unicode中是唯一的。

但是由于26446这个数字不能使用一个字节表示，那么他在内存、持久化、传输的时候，就必须定义一些标准, 这样对端才能正确理解这个bytes序列的具体含义. 对端也才能重新解码回来，这样就有了字符编码规范 - encode. 常见的encode包括ascii、iso-8859-1、utf-8
下面是我在理解过程中，几篇重要的文章：
  - [[http://nedbatchelder.com/text/unipain.html][Pragmatic Unicode]]
  - [[http://www.joelonsoftware.com/articles/Unicode.html][Joel About Unicode and Character Sets]]
  - [[http://docs.python.org/2/howto/unicode.html][Unicode HOWTO]]

理解这两个概念对于搞明白Python的字符串非常重要. 说白了, unicode字符串按照某种编码规范(例如utf-8)编码之后, 就变成了str类型的字节数组, 反之亦然.

下面我们就来看看最常用的utf-8编码

## utf-8 encode
最常用的编码之一，python建议采用的编码格式，采用变长方式编码:
|        + | +        | +                                                     |
|  Hex Min | Hex Max  | Byte Sequence in Binary                               |
| 00000000 | 0000007f | 0vvvvvvv                                              |
| 00000080 | 000007ff | 110vvvvv 10vvvvvv                                     |
| 00000800 | 0000ffff | 1110vvvv 10vvvvvv 10vvvvvv                            |
| 00010000 | 001fffff | 11110vvv 10vvvvvv 10vvvvvv 10vvvvvv                   |
| 00200000 | 03ffffff | 111110vv 10vvvvvv 10vvvvvv 10vvvvvv 10vvvvvv          |
| 04000000 | 7fffffff | 1111110v 10vvvvvv 10vvvvvv 10vvvvvv 10vvvvvv 10vvvvvv |
|          |          |                                                       |
如果字节是10开头，说明他不是一个字符，而是字符的组成字节；字节如果不是0开头，表
示该字符是多字节字符，有几个1，该字符就由几个字节组成，比如'\xc2'，表示该字符由
两字节组成。另外需要注意的是，很多字节是非法的，比如'\xc0\x80'，这个字节数组
在utf8中就是非法的，因为按照编码规则，他代表0，所以假如你做如下操作：
: '\xc0\x80'.encode('utf-8')
会报0xc0为非法字符，无法转换，'0x80'在utf8中表示为'\xc2\x80'

## encoding[fn:3]
   有了前面的铺垫,我们再来看看Python中的字符串编码支持.

   python内部都用unicode做中介。当你定义一个字符串时, 如果没有明确的用u'标示,
   那么他就是一个str类型的字符串数组. 当你需要对一个字符串转换编码时，比如你要把
   一个utf-8编码的字符串转换成gbk编码, python会先该字符串转换成unicode字符，然
   后再转成gbk. 转换成unicode的过程, 可以是隐式的也可以是显式的. 如果是隐式的,
   就像下面示例这样:
   #+BEGIN_SRC python
   # -*- coding: utf-8 -*-
   name = '斯坦福'
   #+END_SRC

   ``` shell
   >name.encode('gbk')
   UnicodeDecodeError: 'ascii' codec can't decode byte 0xe6 in position 0:
   ordinal not in range(128)
   ```

   Python会抛UnicodeDecodeError异常. 导致该结果的原因是, Python隐式转码的过程用
   的缺省编码是ASCII, 但是'斯坦福'这个字符串无法用ASCII编码格式进行编码(ascii只
   能处理0-127的字符).

   要解决上面的问题, 我们需要显式的对name进行转码, 先手动把name解码成unicode字符
   串, 然后再用gbk去做编码, 这样Python才能正常工作:
   ``` shell
   >uname = unicode(name, 'utf-8')    #或者也可以uname = name.decode('utf-8')
   >uname.encode('gb18030')
   '\xcb\xb9\xcc\xb9\xb8\xa3'
   ```

   另外需要注意的是, 我们怎么才能知道Python缺省使用的字符串编码呢? 例如上面赋值
   语句:
   #+BEGIN_SRC python
   name = '斯坦福'
   #+END_SRC

   Python缺省使用什么编码格式来对name进行编码? 在Python编码规范中, 明确要求你指
   定源代码编码[fn:2]， 例如你指定coding为”# -*- coding: utf-8 -*-“，那么
   Python就知道用utf-8对name进行编码, 你在对name做unicode()的时候就不能用gbk
   去做，会出错：
   ``` shell
   >unicode(name, 'gb18030')
   UnicodeDecodeError: 'gb18030' codec can't decode byte 0x8f in position 8:
   incomplete multibyte sequence
   ```
   如果你没有申明编码格式, 那么Python就会用系统的缺省编码, 比如在中文windows系统中, 可能就是cp936.

   转换一个str成为url也必须小心，如果你代码的编码是utf8，而目标网站编码是gbk，
   你在转换的时候需要先转换str编码为gbk，然后做quote:
   ``` shell
   >uname = name.decode('utf-8')
   >urllib2.quote(uname.encode('gb18030'))
   '%CB%B9%CC%B9%B8%A3'
   ```

   搞明白了上面这些东西, 基本你就能解决UnicodeDecodeError问题了.

## 关于print和重定向
这里需要注意，如果你的代码明确了编码，例如:“# -*- coding: utf-8 -*-”，那么你在中文的windows平台上执行print时需要小心，由于缺省用的是utf-8编码, 字符串在输出的时候需要注意, 需要先解码成unicode, 然后让Python自己去编码成console stdout制定的编码格式. 如果你不显示解码成unicode, 就会发生隐式转换, 导致UnicodeDecodeError异常.
``` shell
a = '测试'
print a.decode('utf-8')
```

但是如果你把上面的代码输出重定向到一个文件，无法执行成功，会出现编码错误。出现这个问题的原因是由于重定向之后，python无法知道stdout的encoding，只能使用ASCII来进行转码输出[fn:10]，可以用下面方法解决：
``` shell
import sys, codecs, locale
sys.stdout = codecs.getwriter(locale.getpreferredencoding())(sys.stdout)
a = '测试'
print a.decode('utf-8')
```

## p2k & p3k
上面我们说的都是p2k, 可以看到p2k对于字符串编码的处理非常麻烦, 这点在p3k之后有了很大的改进.

两个版本的python中，string的实现有很大区别:
- p2k中，String分为str类型和unicode类型, str类型其实是字节数组，而unicode类型才是真正语义上的字符串(计算字符串长度的时候就能看出明显区别), 另外p2k中也有byte类型, 但它和str是同义词;
- 而在p3k中，对String进行了统一, 只有unicode类型的string，p2k中的str类型字符直接用byte类型表示，bytearray为字节数组, 也就是说p3k中, str全部用unicode表示, byte类型表示编码过后的字符, 使用中要求你明确，不能混用。转换同样通过decode和encode实现，你可以把str encode成byte，也能把byte decode成str。

这样看来, p3k对于编码的处理方便多了~

当然, 早期的p3k把所有的字符都表示成unicode, 这样对于ascii字符会占用更多的内存. 在3.3之后, Python对ascii str进行了优化，ascii字符采用单字节表示，减少转换成本和内存占用。


# Regex[fn:1]
## MyDemo
   #+BEGIN_SRC python
     str = 'atestaa,btestbb,'
     #不匹配，match是从第一个字符开始匹配
     rs = re.match(r'test.*', str)
     #匹配，rs.group()输出 'atestaa,btestbb,'
     rs = re.match(r'.test.*', str)
     #匹配，注意patten后面多了一个逗号字符，但是匹配结果同上，因缺省是贪吃模式,
     #原则是最长匹配
     rs = re.match(r'.test.*,', str)
     #匹配，注意patten中多了个问号，匹配结果变成了'atestaa,',改变模式为最短匹配
     rs = re.match(r'.test.*?,', str)
     #匹配，group模式，patten中多了一对括号，匹配结果同上，只是括号中表达式匹
     #配的字符会单独存放在re.group(1)中，本例中为'aa'
     rs = re.match(r'.test(.*?),', str)
     #匹配，多了一个re.group(2)，本例中存放'bb'
     rs = re.match(r'.test(.*?),.test(.*?),', str)
     #使用方式同match，唯一区别就是search从任意位置开始匹配，而match必须从第一个
     #字符开始匹配，另外match和search都是一次匹配
     rs = re.search
     #匹配，多次匹配模式，rs输出['aa', 'bb']，return list
     rs = re.findall(r'test(.*?),', str)
   #+END_SRC

   复杂的例子：
   #+BEGIN_SRC python
     #这里匹配xx，xy，yx，yy，但是如果你只想匹配xy和yx的时候怎么办呢
     re = re.search(r'(x|y)(x|y)', 'xy')

     #匹配xy或者yx，不匹配xx，yy，解释一下，第一个大括号里面是表示当找到x的时
     #候，存入group（1）里面，如果是y则不存入group（1），后面一个括号里面表示如
     #果group（1）存在，则继续匹配y，否则匹配x
     rs = re.search(r'(?:(x)|y)(?(1)y|x)', 'xy')

     #OR关系匹配
     #假如你要匹配的内容前面的限定符有变化，比如下面例子，
     s = '''<div class="data">12-11<a href="http://www.google.com"></a></div>
            <div class ="phase">1102<a href="http://www.yahoo.com"></a></div>'''
     rs = re.findall(r'class="(?:(?:data)|(?:phase))">([\d-]+)<a.*?"(.*?)">', s)

   #+END_SRC

## findall group  - return tuple
   #+BEGIN_SRC python
     str = 'purple alice@google.com, blah monkey bob@abc.com blah dishwasher'
     tuples = re.findall(r'([\w\.-]+)@([\w\.-]+)', str)
     print tuples  ## [('alice', 'google.com'), ('bob', 'abc.com')]
     for tuple in tuples:
       print tuple[0]  ## username
       print tuple[1]  ## host
   #+END_SRC

## Options
   The re functions take options to modify the behavior of the pattern match.
   The option flag is added as an extra argument to the search() or findall()
   etc., e.g. re.search(pat, str, re.IGNORECASE).

   - IGNORECASE -- ignore upper/lowercase differences for matching, so 'a'
     matches both 'a' and 'A'.
   - DOTALL -- allow dot (.) to match newline -- normally it matches anything
     but newline. This can trip you up -- you think .* matches everything, but
     by default it does not go past the end of a line. Note that \s (whitespace)
     includes newlines, so if you want to match a run of whitespace that may
     include a newline, you can just use \s*
   - MULTILINE -- Within a string made of many lines, allow ^ and $ to match the
     start and end of each line. Normally ^/$ would just match the start and end
     of the whole string.

## Substitution
   The re.sub(pat, replacement, str) function searches for all the instances of
   pattern in the given string, and replaces them. The replacement string can
   include '\1', '\2' which refer to the text from group(1), group(2), and so on
   from the original matching text.

   Here's an example which searches for all the email addresses, and changes
   them to keep the user (\1) but have yo-yo-dyne.com as the host.
   #+BEGIN_SRC python
     str = 'purple alice@google.com, blah monkey bob@abc.com blah dishwasher'
     ## re.sub(pat, replacement, str) -returns new string with all replacements,
     ## \1 is group(1), \2 group(2) in the replacement
     print re.sub(r'([\w\.-]+)@([\w\.-]+)', r'\1@yo-yo-dyne.com', str)
     ## purple alice@yo-yo-dyne.com, blah monkey bob@yo-yo-dyne.com blah
     ##dishwasher
   #+END_SRC


# function & method
简单来说，function可以通过def和lambda定义，当function定义在class内部时，它通过
descriptor protocol转换成method（这么说也不准确，理论上你依然可以通过__dict__方
式绕开descriptor protocol访问到function）。

同时method又分为unbound method和bound method[fn:9]，当我们直接通过class访问method时，
它是unbound method；当我们通过该class的instance访问method时，他是bound method。
#+BEGIN_SRC python
def foo(x):
    print 'call foo', x

class MyClass(object):
    def m1(x):
        print 'call m1', x

    def m2(self, y):
        print 'call m2', y

In [259]: foo
Out[259]: <function __main__.foo>

In [260]: MyClass.m1
Out[260]: <unbound method MyClass.m1>

In [261]: m = MyClass()

In [264]: m.m1
Out[264]: <bound method MyClass.m1 of <__main__.MyClass object at 0x1e49c10>>
#+END_SRC
疑问自然产生，为啥m1是method，明明也是一个function，怎么就变成method了呢，为啥
同样是method还有unbound和bound之分呢，他们之间到底是如何转换的。

其实背后的魔法都来自于descriptor protocol，在解释descriptor之前，我们先来看看
方法m1到底是啥?
#+BEGIN_SRC python
In [281]: foo
Out[281]: <function __main__.foo>

In [282]: MyClass.__dict__.get('m1')
Out[284]: <function __main__.m1>

In [285]: MyClass.m1
Out[285]: <unbound method MyClass.m1>
#+END_SRC
参考上面的输出，你可以发现m1依然是一个function，你可以通__dict__访问到它，那为
啥通过MyClass.m1访问时，返回的又是method呢？

其实问题就在于方法m1是descriptor，要理解这个得费点功夫，建议参考Peter Inglesby
的Discovering Descriptors[fn:8]。我这里简单解释一下，首先你需要知道的是，Python
中一切都是对象，m1是一个对象，foo是一个对象，MyClass和m都是对象，而m1是一个实现
了descriptor protocol的对象(实现了__get__,__set__,__del__三个方法中的任意一个）
，当访问descriptor对象时，直接调用__get__方法返回了一个method。
#+BEGIN_SRC python
In [328]: '#'.join(dir(MyClass.m1))
Out[330]: '__call__#__class__#__cmp__#__delattr__#__doc__#__format__#__func__#
__get__#__getattribute__#__hash__#__init__#__new__#__reduce__#__reduce_ex__#
__repr__#__self__#__setattr__#__sizeof__#__str__#__subclasshook__#im_class#
im_func#im_self'
#+END_SRC
当你通过class访问m1的时候，Python其实是调用MyClass.m1.__get__(None, MyClass)，
当你通过instance访问m1的是，调用MyClass.m1.__get__(m, MyClass)。
#+BEGIN_SRC python
In [346]: MyClass.m1.__get__(None, MyClass)
Out[346]: <unbound method MyClass.m1>

In [347]: MyClass.m1.__get__(m, MyClass)
Out[354]: <bound method MyClass.m1 of <__main__.MyClass object at 0x1e49c10>>
#+END_SRC

另外，如果你想调用m1，你会发现你总是失败。
#+BEGIN_SRC python
In [357]: m.m1(1)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-357-329146df192e> in <module>()
----> 1 m.m1(1)

TypeError: m1() takes exactly 1 argument (2 given)

In [358]: MyClass.m1(1)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-360-f35a4576c1b2> in <module>()
----> 1 MyClass.m1(1)

TypeError: unbound method m1() must be called with MyClass instance as first
argument (got int instance instead)
#+END_SRC
当你需要访问一个method，必须满足两个条件，首先必须bound，另外必须在定义的时候
明确定义self参数。Python在调用method的时候会隐式传递instance作为参数，如果你没
有明确声明self参数，调用就无法成功。如果你需要调用没有声明self参数的方法，参考
如下方式。
#+BEGIN_SRC python
In [382]: MyClass.__dict__.get('m1')(1)
call m1 1

In [393]: m.m2(1)
call m2 1

In [394]: m.__class__.__dict__.get('m1')(1)
call m1 1

In [25]: m.m1.im_func(1)
call m1 1

In [26]: MyClass.m1.im_func(1)
call m1 1
#+END_SRC
其他的方法是通过把m1声明为staticmethod或classmethod方法实现。


# descriptor protocol
TODO & staticmethod & classmethod & instancemethod & super


# scopes & closure
先看下面的代码片段
#+BEGIN_SRC python
funcs = []
def outer():
    for i in range(4):
        def inner():
            print i
        funcs.append(inner)
    return funcs

for f in outer():
    f()

In [126]: 3
3
3
3
#+END_SRC
看到这个结果，是不是有点惊讶，我们期望的输出是“0，2，3，4”，这里输出的确全是3，
其要理解这个首先你必须明白变量的范围和查找规则，变量可以定义在module、function
中，另外就是Python内建功能内，module对应的是global，function对应的是local，而内
建功能对应的是builtin。Python的查找规则简单来说叫LEGB，依次是
local-》enclosing-》global-》builtin。这里奇怪的冒出来一个enclosing，enclosing
这个概念其实是由于Python支持nest function，enclosing module内的变量对于nest
function就是enclosing的概念。

回到上面的例子，funcs是global，outer是global，i对于outer是local（不准确说法），
但是对于inner来说是enclosing，当执行‘print i‘的时候，inner先找自己的local，没有
i定义，然后找enclosing，发现i，就终止查找，使用enclosing中的i。

但是一番解释之后，依然没有回答我们上面的问题，虽然找到了i，但是照理来说，i只有在
outter运行时才存在，而现在outer已经执行结束了，为啥inner还能使用i的值，并且都是3
呢？这就涉及到Python的另一个概念，closure：nest function会保存enclosing变量，这
就解释了为啥他依然能找到i了。至于为啥打印的都是3，这是因为，它只知道i这个变量，
对于i值的变化，他并不关心，他只能保存i的最终状态，当outter执行完的时候i的值已经
是3了，所以inner执行的时候答应的都是3。

避免上面类似的问题，你可以通过显示声明nest function变量
#+BEGIN_SRC python
funcs = []
def outer():
    for i in range(4):
        def inner(i=i):
            print i
        funcs.append(inner)
    return funcs

for f in outer():
    f()

In [127]: 0
1
2
3
#+END_SRC
这样对于inner来说，他有了一个local的i，当他执行的时候，就不再找enclosing了。

我们还需要注意的是global，看下面代码：
#+BEGIN_SRC python
demo_global.py
# -*- coding: utf-8 -*-
x = 10
def f1():
    def inner():
        print x
    return inner

demo_test.py
# -*- coding: utf-8 -*-
import demo_global

x = 30
f2 = demo_global.f1()
f2()
#+END_SRC
这里的输出是10，注意，global的范围就是module，所以在这里我们可以看到对于inner来
说，他的global限制在demo_global.py里面，如果你没有在demo_global中定义x，程序执
行会出错。

另外，Python是先编译再执行，如果你在一个function里面定义了一个变量，不论是在什
么位置，他都被申明为本地变量，下面情况需要注意
#+BEGIN_SRC python
x = 10

def f():
  print x
  x = x + 1
#+END_SRC
如果你调用f，系统会告诉你x是一个没有赋值的本地变量，调用失败


# Management Attribute
python的attribute管理比较复杂，涉及到不同的方法，包括下面4类：
  - "__getattr__ / __getattribute__"
  - "__setattr__ / __delattr__"
  - properties
  - descriptors

其中前两者一般情况下用于代理实现，比如delegation proxy，影响到每个attribute的
行为，而后两者应用于特定的attribute。

"__getattr__"和"__getattribute__"的主要区别在于，前者主要针对未定义的attribute
，而后者针对所有attribute（包括未定义的），使用后者的时候必须小心死循环，因为
你对所有attribute的fetch操作都会触发它的执行，一般情况下如果你需要在它里面获取
attribute的时候，一般可以通过使用superclass的方法实现，比如：
: object.__getattribute__(self, attr)

需要特别注意的是，隐式调用在new class里面是不会触发这两个方法的，比如说你在代码
里面定义了方法"__add__(self, value)"，当你执行"instance + value"时候就不会触发。
old class中任何attribute fetch操作都会触发"__getattr__"，没有__getattribute__。
#+BEGIN_SRC python
# -*- coding: utf-8 -*-
class GetAttr(object):
    eggs = 88
    def __init__(self):
        self.spam = 77
    def __len__(self):
        print('__len__:42')
        return 42
    def __getattr__(self, attr):
        print('getattr:' + attr)
        if attr == '__str__':
            return lambda *args: '[Getattr str]'
        else:
            return lambda *args: None


class GetAttrOld:
    eggs = 88
    def __init__(self):
        self.spam = 77
    def __len__(self):
        print('__len__:42')
        return 42
    def __getattr__(self, attr):
        print('getattr:' + attr)
        if attr == '__str__':
            return lambda *args: '[Getattr str]'
        else:
            return lambda *args: None


class GetAttribute(object):
    eggs = 88
    def __init__(self):
        self.spam = 77
    def __len__(self):
        print('__len__:42')
        return 42
    def __getattribute__(self, attr):
        print('getattribute:' + attr)
        if attr == '__str__':
            return lambda *args: '[GetAttribute str]'
        else:
            return lambda *args: None


for Class in GetAttr, GetAttrOld, GetAttribute:
    print('\n' + Class.__name__.ljust(50, '='))
    x = Class()
    x.eggs
    x.spam
    x.other
    len(x)

    try: x[0]
    except: print('fail []')

    try: x + 99
    except: print('fail +')

    try: x()
    except: print ('fail ()')

    x.b()
    x.__call__()
    print(x.__str__())
    print(x)
#+END_SRC

``` shell
In [168]:
GetAttr===========================================
getattr:other
__len__:42
fail []
fail +
fail ()
getattr:b
getattr:__call__
<__main__.GetAttr object at 0x7fe770053550>
<__main__.GetAttr object at 0x7fe770053550>

GetAttrOld========================================
getattr:other
__len__:42
getattr:__getitem__
getattr:__coerce__
getattr:__add__
getattr:__call__
getattr:b
getattr:__call__
getattr:__str__
[Getattr str]
getattr:__str__
[Getattr str]

GetAttribute======================================
getattribute:eggs
getattribute:spam
getattribute:other
__len__:42
fail []
fail +
fail ()
getattribute:b
getattribute:__call__
getattribute:__str__
[GetAttribute str]
<__main__.GetAttribute object at 0x7fe770053550>
```
关键要注意一点，比如"x.b()"这行代码，在python中是先运行"x.b"这个表达式，注意是
表达式，然后才是执行调用，它会触发__getattr__和__getattribute__方法。

"__setattr__"和"__delattr__"影响所有的attribute赋值和删除操作，也必须注意死循环

properties其实底层就是通过descriptors实现，properties对于简单的attribute get/set
操作处理起来比较方便，不需要实现其他的类，descriptors需要自定义类实现（也可以是
嵌套类），它的强大在于他可以维护client和descriptors两者的状态。一个类实现了三个
方法中的任意一个("__set__(self, instance, value)/__get__(self, instance, owner)
/__delete__(self, instance)，即代表它是descriptors，self为descriptors，instance
为client instance，owner为client class。另外需要注意的是在client申明descriptors
类型的attribute，必须是class attribute。


# Python分支
## Stackless Python[fn:4]
   先看看wikipedia上的解释：\\
   "Stackless microthreads are managed by the language interpreter itself,
   not the operating system kernel—context switching and task scheduling is
   done purely in the interpreter (these are thus also regarded as a form of
   green thread). This avoids many of the overheads of threads, because no mode
   switching between user mode and kernel mode needs to be done, and can
   significantly reduce CPU load in some high-concurrency situations.Due to the
   considerable number of changes in the source, Stackless Python cannot be
   installed on a preexisting Python installation as an extension or library.
   It is instead a complete Python distribution in itself. The majority of
   Stackless's features have also been implemented in PyPy, a self-hosting
   Python interpreter and JIT compiler." \\
   stackless是一个python的分支实现，主要的优势就是microthread，其实就是协程
   （coroutines / green threads），轻量级别的伪线程，采用栈方式实现协程切换，不
   会带来ContextSwitch，缺点是需要自己管理协程调用栈，stackless能实现10，0000级
   别的连接.

   ``` shell
   from greenlet import greenlet
   def test1():
       print 12
       gr2.switch()
       print 34

   def test2():
       print 56
       gr1.switch()
       print 78   #not to be printed

   gr1 = greenlet(test1)
   gr2 = greenlet(test2)
   gr1.switch()
   ```

## PyPy
   实现了一个JIT，运行时转换python代码到机器码，主要关注在运行效率的提升，和原生
   python兼容，采用RPython（python的一个子集）实现，目前好像还没有看到那里有大规
   模的使用。关于PyPy和CPython的性能讨论可以看这里[fn:5]


# Django & Flask
  Django是一个整体解决方案，Python Web framework，实现快速开发网站，你不用做什么
  其他技术选择，比如templage engine、ORM等，社区很活跃。
  Flask是一个microframework，基于Werkzeug和Jinja2，Flash核心非常简单，不像Django
  帮你做了所有技术决策，缺省template engine用的是jinja。


# Twisted & Tornado & gevent & eventlet
  Twisted是一个经典的非阻塞I/O框架，采用Callback方式实现异步调用，用户最广
  gevent和eventlet都是基于协程的异步框架（底下是greenlet），gevent是eventlet之后
  的一个项目，作者当时的一些需求不能被eventlet满足而重新设计的[fn:6]，gevent底下
  的I/O异步框架是libevent/livev（epoll on linux / kqueue on FreeBSD / iocp on
  windows)，evenlet自己实现的。

# requests
支持sockets需要版本2.10, 同时urllib3需要1.14以上版本, 同时需要安装PySocks包

# tor
需要安装stem, 测试代码:

``` python
from stem import Signal
from stem.control import Controller
import requests

# signal TOR for a new connection
def renew_connection():
    with Controller.from_port(port = 9051) as controller:
        controller.authenticate(password="lichunlongasd12288")
        controller.signal(Signal.NEWNYM)
renew_connection()
proxies = {
  'http': 'socks5://localhost:9050',
  'https': 'sockets5://localhost:9050',
}
print requests.session().get("http://httpbin.org/ip",proxies=proxies).content
```

# mysql
插入空到字段用`None`, 或者`''`, 不是`"null"`


# datetime
[How to work with dates and time with Python](https://opensource.com/article/17/5/understanding-datetime-python-primer)


# Emacs & Python
  设置python-shell-extra-pythonpaths变量来增加额外的PYTHONPATH环境变量
## ipdb
   emacs里面的ipython需要运行%pdb打开断点模式，否则好像即使设置了断点也进不去，
   不清楚为什么，另外一些简单的命令记录一下：
   - s: step into
   - n: step over
   - c: continue to next breakpoint
   - l: some more context
   - a: all argument
   - ?s: help for command s


# 一些必读的文章
   - [[http://docs.python.org/2/reference/index.html][The Python Language Reference]]
   - [[http://docs.python-guide.org/en/latest/][The Hitchhiker’s Guide to Python!]]
   - [[http://stackoverflow.com/questions/231767/the-python-yield-keyword-explained][The Python yield keyword explained]]
   - [[http://stackoverflow.com/questions/739654/how-can-i-make-a-chain-of-function-decorators-in-python][How can I make a chain of function decorators in Python?]]
   - [[http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python][What is a metaclass in Python?]]
   - [[http://nbviewer.ipython.org/urls/gist.github.com/ChrisBeaumont/5758381/raw/descriptor_writeup.ipynb][Python Descriptors Demystified]]
   - [[http://stackoverflow.com/questions/101268/hidden-features-of-python][Hidden features of Python]]
   - [[http://google-styleguide.googlecode.com/svn/trunk/pyguide.html][Google Python Style Guide]]
   - [[http://stackoverflow.com/questions/2709821/python-self-explained][Python 'self' explained]]
   - [[http://docs.python.org/release/2.5/whatsnew/pep-343.html][PEP 343: The 'with' statement]]


# 自己看的书
建议还是系统的看一本书，自己感觉看书能比较系统的了解一门语言，相比边学边用或者
碰到问题再去找答案，前者效率会更高
   - Learning Python
   - Python Essential Reference


# Footnotes
[fn:1] [[https://developers.google.com/edu/python/regular-expressions][Google Python Regex]]

[fn:2] [[http://www.python.org/dev/peps/pep-0263/][PEP 0263]]

[fn:3] [[http://docs.python.org/2/howto/unicode.html][Unicode HOWTO]]

[fn:4] [[http://en.wikipedia.org/wiki/Stackless_Python][Stackless]]

[fn:5] [[http://stackoverflow.com/questions/2591879/pypy-how-can-it-possibly-beat-cpython][Pypy - How can it possibly beat CPython]]

[fn:6] [[http://blog.gevent.org/2010/02/27/why-gevent/][Comparing gevent to eventlet]]

[fn:7] [[http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python][What is a metaclass in Python]]

[fn:8] [[https://ep2012.europython.eu/conference/talks/discovering-descriptors][Discovering Descriptors]]

[fn:9] [[http://stackoverflow.com/questions/114214/class-method-differences-in-python-bound-unbound-and-static][Class method differences in Python: bound, unbound and static]]

[fn:10] [[https://wiki.python.org/moin/PrintFails][PrintFails]]
