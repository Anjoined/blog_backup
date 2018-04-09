---
layout: post
title: 基于 Python 的小说简易爬虫
date: 2018-04-09 09:50:21
comments: true
tags: 
	- Python
	- Web
	
---



最近在学 Python，个人觉得 Python 是一种比较好玩的编程语言。快速看过一遍之后准备自己写个小说爬虫来巩固下 Python 基础知识。

---

# 1. 前期知识准备
Python 基础语法、正则表达式、BeautifulSoup 库
传送门：
[廖雪峰老师的Python新手教程](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000)
[BeautifulSoup教程，转自:http://www.cnblogs.com/wupeiqi/articles/6283017.html](http://www.cnblogs.com/wupeiqi/articles/6283017.html)

---

# 2. 选择爬取的目标
我选取的目标是网上随便找的一个免费小说网：`https://www.qu.la`（其实是一个盗版小说网站）。选取这个网站的原因是该网站的 html 标签比较容易提取，适合练手，而且亲测没有任何反爬虫机制，可以随便蹂躏（咳咳，大家收敛一点，不要太用力）。坏处么...就是网站服务器不稳定，经常爬着爬着中断了，报`10053`错误，找不到原因只能重来，据说换成 `Python 3`不会出现这个问题？本文用的`Python 2.7`，还没试过`Python 3`，有时间的读者可以试一下。

---

<!--more-->


# 3. 实际操作
古人云：“机会总是留给有准备的人的。”所以我们要先理清好我们的思路，再动手，本文的思路如下：

![upload.png](https://kcms.konkawise.com:443/upload/201804031818515633.png)
## 3.1 下载目标 url 的 html
我选取了《超级神基因》作为要下载的小说进行演示。定义一个`download`方法，传入两个参数。 url 为小说的第一章的网址：`https://www.qu.la/book/25877/8923073.html`，当网络出现错误时，重新发起请求，`num_retries = 5`默认同一链接请求不超过5次。该方法返回了网站的`html`代码。Python 代码如下：

```Python
import urllib2

def download(url, num_retries=5):
    """
    :param url: the fist chapter's url of novel like 'https://www.qu.la/book/25877/8923073.html'
    :param num_retries: times to retry to reconnect when fail to connect
    :return:html of url which be inputted
    """
    print 'Start downloading:', url
    try:
        html = urllib2.urlopen(url).read()
        print 'Download finished:', url
    except urllib2.URLError as e:
        print 'Download fail.Download error:', e.reason
        html = None
        if num_retries > 0:
            print 'Retrying:', url
            html = download(url, num_retries - 1)
    return html
```

## 3.2 获取每一章的标题和正文
这一步我们要查看包含有标题和正文的标签，将相关的标签内容筛选出来。注意一定要将`html`代码下载下来再查看，不要直接用浏览器的开发工具查看源代码。我踩了这个坑，发现一直匹配不到相关的标签，后来有前端老司机告诉我`JavaScript`代码可能会自动完成代码（好像是这么个意思，作者前端 0 基础），你在浏览器看到的代码很可能改变了。
将下载下来的`html`代码保存为一个`.html`文件，保存`html`代码如下：
```python
import re
import os

html = download('https://www.qu.la/book/25877/8923072.html')
with open(os.path.join(r'C:\Users\admin\Desktop', '123.html'), 'wb') as f:
    f.write(html)
```
查看保存的在桌面的`123.html`文件，可以发现章节标题的标签为`<h1>`
![gettitle](https://img-blog.csdn.net/20180403143711220?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MTc1NzUz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
使用正则表达式在`html`代码中匹配该标签的内容，代码如下：
```python
import re

def get_title(html):
    """Find Title of each chapter,return the title of chapter
    """
    title_regex = re.compile('<h1>(.*?)</h1>', re.IGNORECASE)
    title = str(title_regex.findall(html)[0])
    return title
```

同理，在`html`代码中查找文本的相关标签为`<div id="content">`。
![getcontent](https://img-blog.csdn.net/20180403145908349?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MTc1NzUz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
然后这里有个坑，用正则表达式居然匹配不到！！！于是用了另一种方法：使用`BeautifulSoup`库。话不多说，放代码：
```python
from bs4 import BeautifulSoup

def get_content(html):
    """get content of each chapter from the html
    """
    soup = BeautifulSoup(html, 'html.parser')
    # fixed_html = soup.prettify()
    div = soup.find('div', attrs={'id': 'content'})
    [s.extract() for s in soup('script')]
    # print div
    content = str(div).replace('<br/>', '\n').replace('<div id="content">', '').replace('</div>', '').strip()
    return content
```

## 3.3 保存标题和正文
接下来就是把上一步得到的标题和正文保存到文档中去，作者偷懒把地址写死了，这一步比较简单，不多做解释，看代码：
```python
import re

def save(title, content):
    with open(r'C:\Users\admin\Desktop\DNAofSuperGod\novel.txt', 'a+') as f:
        f.writelines(title + '\n')
        f.writelines(content + '\n')
```
## 3.4 获取下一章的 url
在`html`代码中找到`下一章`的链接：
![下一章链接](https://img-blog.csdn.net/20180403152703897?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MTc1NzUz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
老规矩，在`html`代码中匹配这个标签，拿到`href`的内容，代码如下：
```python
from bs4 import BeautifulSoup

def get_linkofnextchapter(html):
    """This method will get the url of next chapter
    :return: a relative link of the next chapter
    """
    soup = BeautifulSoup(html, 'html.parser')
    a = soup.find('a', attrs={'class': 'next', 'id': 'A3', 'target': '_top'})
    # print a['href']
    return a['href']
```

## 3.5 编写启动的方法
调用前面的方法，作为整个程序的入口，代码如下：
```python
def begin(url):
    # make sure panth exited
    if not os.path.isdir(r'C:\Users\admin\Desktop\DNAofSuperGod'):
        os.mkdir(r'C:\Users\admin\Desktop\DNAofSuperGod')
    # remove old file and build a new one
    if os.path.isfile(r'C:\Users\admin\Desktop\DNAofSuperGod\novel.txt'):
        os.remove(r'C:\Users\admin\Desktop\DNAofSuperGod\novel.txt')
    html = download(url)
    # if html is None,download fail.
    if not html == None:
        title = get_title(html)
        print title
        content = get_content(html)
        save(title, content)
        print 'Have saved %s for you.' % title
        link = get_linkofnextchapter(html)
        # judge if has next chapter?
        if not re.match(r'./', link):
            nexturl = urlparse.urljoin(url, link)
            begin(nexturl)
        else:
            print 'Save finished!'
    else:
        print 'Download fail'
```

## 3.6 启动爬虫
终于到达最后一步啦，我们要启动我们的爬虫程序，调用代码很简单：
```python
url = 'https://www.qu.la/book/25877/8923072.html'
begin(url)
```
但是！如果顺利的话，在程序下载到900多章的时候，你可以很幸福地看到程序报错了！

> RuntimeError: maximum recursion depth exceeded

找了度娘后发现这是`Python`的保护机制，防止无限递归导致内存溢出，默认的递归深度是 1000，我们的`begin`方法递归超出了这个深度范围，所以我们可以把这个默认值改大一点即可。
```python
import sys

# change recursion depth to 10000(defult is 1000)
sys.setrecursionlimit(10000)
url = 'https://www.qu.la/book/25877/8923072.html'
begin(url)
```

## 3.7 附上完整代码
```python
import urllib2
import re
import os
import urlparse
import sys
from bs4 import BeautifulSoup

# __author__:Anjoined

"""
This demon is a webspider to get a novel from https://www.qu.la
"""


def download(url, num_retries=5):
    """
    :param url: the fist chapter's url of novel like 'https://www.qu.la/book/25877/8923073.html'
    :param num_retries: times to retry to reconnect when fail to connect
    :return:html of url which be inputted
    """
    print 'Start downloading:', url
    try:
        html = urllib2.urlopen(url).read()
        print 'Download finished:', url
    except urllib2.URLError as e:
        print 'Download fail.Download error:', e.reason
        html = None
        if num_retries > 0:
            print 'Retrying:', url
            html = download(url, num_retries - 1)
    return html


def get_title(html):
    """Find Title of each chapter,return the title of chapter
    """
    title_regex = re.compile('<h1>(.*?)</h1>', re.IGNORECASE)
    title = str(title_regex.findall(html)[0])
    return title


def get_content(html):
    """get content of each chapter from the html
    """
    soup = BeautifulSoup(html, 'html.parser')
    # fixed_html = soup.prettify()
    div = soup.find('div', attrs={'id': 'content'})
    [s.extract() for s in soup('script')]
    # print div
    content = str(div).replace('<br/>', '\n').replace('<div id="content">', '').replace('</div>', '').strip()
    return content


def get_linkofnextchapter(html):
    """This method will get the url of next chapter
    :return: a relative link of the next chapter
    """
    soup = BeautifulSoup(html, 'html.parser')
    a = soup.find('a', attrs={'class': 'next', 'id': 'A3', 'target': '_top'})
    # print a['href']
    return a['href']


def save(title, content):
    with open(r'C:\Users\admin\Desktop\DNAofSuperGod\novel.txt', 'a+') as f:
        f.writelines(title + '\n')
        f.writelines(content + '\n')


def begin(url):
    # make sure panth exited
    if not os.path.isdir(r'C:\Users\admin\Desktop\DNAofSuperGod'):
        os.mkdir(r'C:\Users\admin\Desktop\DNAofSuperGod')
    # remove old file and build a new one
    if os.path.isfile(r'C:\Users\admin\Desktop\DNAofSuperGod\novel.txt'):
        os.remove(r'C:\Users\admin\Desktop\DNAofSuperGod\novel.txt')
    html = download(url)
    # if html is None,download fail.
    if not html == None:
        title = get_title(html)
        print title
        content = get_content(html)
        save(title, content)
        print 'Have saved %s for you.' % title
        link = get_linkofnextchapter(html)
        # judge if has next chapter?
        if not re.match(r'./', link):
            nexturl = urlparse.urljoin(url, link)
            begin(nexturl)
        else:
            print 'Save finished!'
    else:
        print 'Download fail'


# change recursion depth as 10000(defult is 900+)
sys.setrecursionlimit(10000)
url = 'https://www.qu.la/book/25877/8923072.html'
begin(url)
```

---
# 4. 优化

这是在初学`Python`后的练手项目，还有很多优化的空间，暂时先写下来，留待以后改进。

 - 将单线程改为多线程或多进程，利用并发加快下载速度
 - 加入缓存机制，将下载过的`url`缓存到本地，如果程序中断了不需要从头开始下载，从缓存中提取相关信息即可

