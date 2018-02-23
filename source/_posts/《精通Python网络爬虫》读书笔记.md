---
title: 《精通Python网络爬虫》读书笔记
tags: [otherTec]
---

> 虽然是JD打折的时候买的实体书，但是近300页的技术书最后基本能够看完还是感觉挺不容易的。本书还是比较合胃口的，有基础但不限于乏味的独立知识点，有框架介绍也有侧重点，而且从0开始实战什么的才是技术书读者的最爱。书可能看完了就丢在一边了，后面也很难再次翻起，这里记录一些point，也方便以后查阅。

<!-- more -->

#### 1.Urllib库与URLError异常处理

1.1 爬取网页，基本指令
导入
 `import urllib.request`
爬取一个网页
 `file = urllib.request.urlopen("http://www.baidu.com")`
打印输出
 `print(data)`
读取内容
 `file.read()` 读取所有内容，返回字符串
 `file.readlines()` 读取所有内容，返回列表变量
 `file.readline()` 读取一行内容
打开文件
 `fhandle = open(path, "wb")`
写入文件
 `fhandle .write(data)`
关闭文件
 `fhandle .close()`
举个栗子

```
import urllib.request
url = "http://blog.csdn.net/chunqiuwei/article/details/74079916"
data = urllib.request.urlopen(url)
fhandle = open("/Users/xunwang/Desktop/python/Demo1.2.html", "wb")
fhandle.write(data.read())
fhandle.close()
```
爬取网页直接写入文件
 `urllib.request.urlretrieve(url, path)`
清除urlretrieve产生的缓存
 `urllib.request.urlcleanup()`
返回相关信息
 `file.info()`
返回状态码
 `file.getcode()`
返回当前爬的url
 `file.geturl()`
文本编码
 `urllib.request.quote()`
文本解码
 `urllib.request.unquote()`

1.2 浏览器模拟，Headers属性
有时候爬虫会返回403错误，这时候可以尝试在header中设置User-Agent来假装是浏览器。
方法1，使用build_opener()修改报头
```
import urllib.request
url = "http://blog.csdn.net/chunqiuwei/article/details/74079916"
headers = ("User-Agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36")
opener = urllib.request.build_opener()
opener.addheaders = [headers]
data = opener.open(url).read()
print(data)
```

方法2，使用add_header()添加报头
```
import urllib.request
url = "http://blog.csdn.net/chunqiuwei/article/details/74079916"
req = urllib.request.Request(url)
req.add_header("User-Agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36")
data = urllib.request.urlopen(req).read()
print(data)
```

1.3 超时设置
```
import urllib.request
for i in range(1,100):
    try:
        file = urllib.request.urlopen("https://myst729.github.io", timeout = 1)
        data = file.read()
        print(len(data))
    except Exception as e:
        print("出现异常 --> " + str(e))
```

1.4 HTTP协议请求

1.4.1 GET请求
```
import urllib.request
keywd = "hello"
url = "http://www.baidu.com/s?wd=" + keywd
req = urllib.request.Request(url)
data = urllib.request.urlopen(req).read()
print(data)
```
如果出现中文的参数，那么使用`urllib.request.quote()`编码了再拼上去。

1.4.2 POST请求
```
import urllib.request
import urllib.parse
url = "http://www.iqianyue.com/mypost/"
postdata = urllib.parse.urlencode({
    "name":"ceo@iqianyue.com",
    "pass":"123456"
}).encode("utf-8")
req = urllib.request.Request(url,postdata)
req.add_header("User-Agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36")
data = urllib.request.urlopen(req).read()
fhandle = open("/Users/xunwang/Desktop/python/Demo1.4.html", "wb")
fhandle.write(data)
fhandle.close()
```

1.5 代理服务器设置
使用作者提供的网址`http://yum.iqianyue.com/proxy`  去查询最新的代理IP
```
def use_proxy(proxy_addr, url):
    import urllib.request
    proxy = urllib.request.ProxyHandler({'http':proxy_addr})
    opener = urllib.request.build_opener(proxy, urllib.request.HTTPHandler)
    # headers = ("User-Agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36")
    # opener.addheaders = [headers]
    urllib.request.install_opener(opener) #安装全局opener
    data = urllib.request.urlopen(url).read().decode("utf-8")
    return data
proxy_addr = "121.31.101.150:8123"
data = use_proxy(proxy_addr, "http://www.baidu.com")
print(len(data))
```

1.6 异常处理URLError
>一般错误汇总
200 正常
301 重定向到新的url，永久
302 重定向到临时url，非永久
304 请求资源未更新
400 非法请求
401 请求未经授权
403 禁止访问
404 没找到页面
500 服务器内部错误
501 服务器不支持实现请求所需功能

一般产生URLError原因可能为
1.连接不上服务器
2.远程URL不存在
3.无网络
4.触发了HTTPError(即上面错误码的那些)
`一般先用子类处理，处理不了交给父类`
```
import urllib.request
import urllib.error
try:
    urllib.request.urlopen("http://www.baiduss.com")
except urllib.error.HTTPError as e:
    print(e.code)
    print(e.reason)
except urllib.error.URLError as e:
    if hasattr(e, "code"):
        print(e.code)
    if hasattr(e, "reason"):
        print(e.reason)
```

#### 2.正则表达式与cookie的使用
一般使用`re`模块实现python正则表达功能。

2.1 原子

2.1.1 普通字符作为原子
比如`数字，大小写字母，下划线等`
```
import re
pattern = "yue"
string = "http://yum.iqianyue.com"
result = re.search(pattern, string)
print(result)
```
运行结果
```
<_sre.SRE_Match object; span=(16, 19), match='yue'> 
```

2.1.2 非打印字符作为原子
比如`\n，\t`

2.1.3 通用字符作为原子
`\w` 匹配任意一个字母，数字或者下划线
`\d` 匹配任意一个十进制数
`\s` 匹配任意一个空白字符
`以上的大写形式都是匹配对应的补集`
```
import re
pattern = "\w\dpython\w"
string = "abcdfphp345pythony_py"
result = re.search(pattern, string)
print(result)
```

2.1.4 原子表
`[]中括号`括起来的里面的原子具有相同的地位，只要`有里面一个`能够匹配成功，就算匹配成功。
`[^]`代表`除了括号里的原子外`其他均可以匹配
```
import re
pattern1 = "\w\dpython[xyz]\w"
pattern2 = "\w\dpython[^xyz]\w"
pattern3 = "\w\dpython[xyz]\W"
string = "abcdfphp345pythony_py"
result1 = re.search(pattern1, string)
result2 = re.search(pattern2, string)
result3 = re.search(pattern3, string)
print(result1)
print(result2)
print(result3)
```
运行结果
```
<_sre.SRE_Match object; span=(9, 19), match='45pythony_'>
None
None
```

2.2 元字符

`.` 匹配除换行符外的任意字符
```
import re
pattern1 = ".python..."
string = "abcdfphp345pythony_py"
result1 = re.search(pattern1, string)
print(result1)
```
运行结果
```
<_sre.SRE_Match object; span=(10, 20), match='5pythony_p'> 
```
`^` 匹配字符串的开始位置 
```
import re
pattern1 = "^abd"
pattern2 = "^abc"
pattern3 = "py$"
pattern4 = "ay$"
string = "abcdfphp345pythony_py"
result1 = re.search(pattern1, string)
result2 = re.search(pattern2, string)
result3 = re.search(pattern3, string)
result4 = re.search(pattern4, string)
print(result1)
print(result2)
print(result3)
print(result4)
```
运行结果
```
None
<_sre.SRE_Match object; span=(0, 3), match='abc'>
<_sre.SRE_Match object; span=(19, 21), match='py'>
None 
```

`$` 匹配字符串的结束位置

`*` 匹配0次，1次或多次前面的原子

`?` 匹配0次，1次前面的原子 

`+` 匹配1次，多次前面的原子

`{n}` 前面的原子恰好出现n次
`{n,}` 前面的原子至少出现n次
`{n,m}` 前面的原子至少出现n次，至多出现m次
```
import re
pattern1 = "py.*n"
pattern2 = "cd{2}"
pattern3 = "cd{3}"
pattern4 = "cd{2,}"
string = "abcddddfphp345pythony_py"
result1 = re.search(pattern1, string)
result2 = re.search(pattern2, string)
result3 = re.search(pattern3, string)
result4 = re.search(pattern4, string)
print(result1)
print(result2)
print(result3)
print(result4)
```
运行结果
```
<_sre.SRE_Match object; span=(14, 20), match='python'>
<_sre.SRE_Match object; span=(2, 5), match='cdd'>
<_sre.SRE_Match object; span=(2, 6), match='cddd'>
<_sre.SRE_Match object; span=(2, 7), match='cdddd'> 
```

`|` 模式选择符
选择符中的任意一边满足就行
```
import re
pattern1 = "python|php"
string = "abcdddphpdfphp345pythony_py"
result1 = re.search(pattern1, string)
print(result1)
```
运行结果
```
<_sre.SRE_Match object; span=(6, 9), match='php'> 
```

`()` 模式单元符
()中的原子组成一个大原子

```
import re
pattern1 = "(cd){1,}"
pattern2 = "cd{1,}"
string = "abcdddphpdfphp345pythony_py"
result1 = re.search(pattern1, string)
result2 = re.search(pattern2, string)
print(result1)
print(result2)
```

运行结果

```
<_sre.SRE_Match object; span=(2, 4), match='cd'>
<_sre.SRE_Match object; span=(2, 6), match='cddd'> 
```

2.3 模式修正

`I` 匹配时忽略大小写
`M` 多行匹配
`L` 做本地化识别匹配
`U` 根据Unicode字符及解析字符
`S` 让.匹配包括换行符，即用了该模式修正符后，"."匹配就可以匹配任意字符了

```
import re
pattern1 = "python"
pattern2 = "python"
string = "abcdddphpdfphp345Pythony_py"
result1 = re.search(pattern1, string)
result2 = re.search(pattern2, string, re.I)
print(result1)
print(result2)
```
运行结果
```
None
<_sre.SRE_Match object; span=(17, 23), match='Python'> 
```

2.4 贪婪模式与懒惰模式
`贪婪模式`就是尽可能多的匹配，`懒惰模式`就是尽可能少的匹配。
```
import re
pattern1 = "p.*y"
pattern2 = "p.*?y"
string = "abcdddphpdfphp345Pythony_py"
result1 = re.search(pattern1, string)
result2 = re.search(pattern2, string)
print(result1)
print(result2)
```
运行结果
```
<_sre.SRE_Match object; span=(6, 27), match='phpdfphp345Pythony_py'>
<_sre.SRE_Match object; span=(6, 19), match='phpdfphp345Py'> 
```

2.5 正则表达式常见函数
2.5.1 re.match()和re.search()
```
import re
string = "hellomypythonhispythonourpythonend"
pattern = ".python."
result = re.match(pattern, string)
result2 = re.search(pattern, string)
print(result)
print(result2)
```
运行结果
```
None
<_sre.SRE_Match object; span=(6, 14), match='ypythonh'> 
```
`match`函数是从源字符串的开头进行匹配，而`search`函数会在全文中进行匹配

2.5.2 全局匹配函数
使用`re.complie()`预编译，再使用`ffindall()`找出全部结果
```
import re
string = "hellomypythonhispythonourpythonend"
pattern = re.compile(".python.")
result = pattern.findall(string)
# pattern = ".python."
# result = re.compile(pattern).findall()
print(result)
```
运行结果
```
 ['ypythonh', 'spythono', 'rpythone'] 
```

2.5.3 re.sub()函数
使用正则来实现替换功能
```
import re
string = "hellomypythonhispythonourpythonend"
pattern = ".python."
result1 = re.sub(pattern, "php", string) #全部替换
result2 = re.sub(pattern, "php", string, 2) #替换两次
print(result1)
print(result2)
```
运行结果
```
hellomphpiphpuphpnd
hellomphpiphpurpythonend 
```

2.6 cookie
2.6.1 
维持会话状态的常用方式一般为两种，`cookie`和`session`。
如果是`cookie`，登录后会把会话信息保存在客户端，重新访问时会从本地cookie中读取对应的会话信息，从而判断目前的会话状态。
如果是`session`，会话信息保存在服务端，服务端会给客户端发一个`sessionId`，这个信息保存在cookie中（如果本地禁用cookie，可能保存在其他地方）。重新访问时，根本本地信息发送至服务端检索出对应的会话状态。

2.6.2 
进行cookie处理的一种常用思路如下：
1.导入Cookie处理模块http.cookiejar
2.使用http.cookjar.CookJar()创建CookieJar对象。
3.使用HTTPCookieProcessor创建cookie处理器，并以其为参数构建opener对象。
4.创建全局默认的opener对象。
```
import urllib.request
import urllib.parse
import http.cookiejar
#这个地址是在network中监控的真实提交表单的地址
url = "http://bbs.chinaunix.net/member.php?mod=logging&action=login&loginsubmit=yes&loginhash=LApSM"
postdata = urllib.parse.urlencode({
    "username":"same4869",
    "password":"58129553"
}).encode('utf-8')
req = urllib.request.Request(url, postdata)
req.add_header("User-Agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36")
#使用http.cookiejar.CookieJar()创建CookieJar对象
cjar = http.cookiejar.CookieJar()
#使用HTTPCookieProcessor创建cookie处理器，并以其为参数构造opener
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cjar))
#安装为全局opener
urllib.request.install_opener(opener)
file = opener.open(req)
data = file.read()
file = open("/Users/xunwang/Desktop/python/Demo1.5.html", "wb")
file.write(data)
file.close()
url2 = "http://bbs.chinaunix.net/forum.php"
data2 = urllib.request.urlopen(url2).read()
fhandle = open("/Users/xunwang/Desktop/python/Demo1.5.1.html", "wb")
fhandle.write(data2)
fhandle.close()
```

#### 3.手写Python爬虫
