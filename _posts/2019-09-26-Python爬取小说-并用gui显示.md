## Python爬取小说（附图形界面）
为了方便没有python环境的同学，这里是打包好的exe直接运行就行了（[点击我进去下载](https://github.com/NoBugBoy/ptyhon-)）首先爬的是基本全部免费小说的网站URL：
```python
http://www.quanshuwang.com/modules/article/search.php?searchkey={}&searchtype=articlename
```
这里的搜索关键字是URL编码所以要通过urllib去把字符编个码才行，然后这搜索结果的界面分两种，一种就是只有索引名称唯一的比如（**剑来**），然后就是一些关键词在索引上的比如（**凡人修仙传，重生之*****），这种属于偏向模糊查询，查询到的结果一般是多个，但是只有第一个是真正想要的：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926113539672.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926113329901.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
然后这个里就需要有分支去处理多个结果的情况，把它变成像第一张图的样子然后合并分支逻辑，python玩的并不熟练（**毕竟是做java的233333**）虽然知道不能用异常控制逻辑但是这样貌似比较方便，然后就有了下面这**一坨**代码：
```python
 try:
        #如果没有这个那么就是唯一的 否则就去取第一条
        html = bs.select('.main-index')[0].select('.hottext')[0]
        href = bs.select('#navList > section > ul > li:nth-child(1) > span > a.clearfix.stitle')[0].get('href')
        body = requests.get(href, headers=headers, timeout=60)
        bts = BeautifulSoup(body.content, 'html.parser')
        href = bts.select('.reader')[0].get('href')
        return retunInfo(href)
    except:
        #单独一本
        href = bs.select('.reader')[0].get('href')
        return retunInfo(href)
```
处理完这些就是获取小说的章节，和具体内容(**需要GBK解码否则乱码**)这部分很简单，但是由于某些小说章节太多导致速度可能不太理想，最后去掉一些垃圾信息
```python
def getContent(url):
    body = requests.get(url,headers=headers,timeout=60)
    bs = BeautifulSoup(body.content.decode('gbk'), 'html.parser')
    content = bs.select('#content')[0].get_text()
    #去掉垃圾信息
    return content.replace('style5();','').replace('style6();','').replace('吾网66721提醒书友注意休息眼睛哟','').replace('享受阅读乐趣，尽在吾网661,661是我们唯一的域名哟！','')
```
最后就是渲染图形界面，这里了解了一下pythonGui，应该是Tkinter比较全面，但是我这里主要就是为了展示并不打算弄的太复杂，就使用了不太完善的easyGui去处理（**这里有个巨坑，就是阅读小说页面要不然就最大化才能看到下面的下一章按钮，否则就只能等看到最后面然后向上拖拽一下了，比较不人性化**）然后展示一下页面：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926114912945.png)

![这里是选择页面](https://img-blog.csdnimg.cn/20190926114728363.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
这里是被我拖拽过的，其实可以滑轮滚动，但是居然自动按文字数量填充高度，醉了.....

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926114803639.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)




然后再贴代码之前，这东西还有一些bug就是不能太频繁的搜索，而且有时候会出现其实有但是却查不到的现象（可能是限制了访问频率），喜欢的帮忙点个赞，关注谢谢啦...，下面是全部源码


```python
import urllib
import sys
import easygui as ui
import requests
from bs4 import BeautifulSoup

URL = 'http://www.quanshuwang.com/modules/article/search.php?searchkey={}&searchtype=articlename'
headers = {
    "User-Agent":"Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36"

}

def retunInfo(href):
    detail = requests.get(href,headers=headers,timeout=60)
    bs = BeautifulSoup(detail.content, 'html.parser')
    list = bs.select('#chapter > div.chapterSo > div.chapterNum > ul > div.clearfix.dirconone')[0]
    if list is None or len(list) == 0 :
        list = bs.select('#navList > section > ul > li:nth-child(1) > span > a.clearfix.stitle')[0]
        raise Exception
    content = list.select('a')
    urls = {}
    urlToName = {}
    names = []
    urlIndex = []
    for index in content:
        names.append(index.get('title'))
        urlIndex.append(index.get('href'))
        urls[index.get('title')] = index.get('href')
        urlToName[index.get('href')]=index.get('title')
    return urls,names,urlToName,urlIndex

def getList(bookName):

    key = urllib.parse.quote(bookName.encode('gb2312'))
    souce = requests.get(URL.format(key),headers=headers,timeout=60)
    bs = BeautifulSoup(souce.content.decode('gbk'), 'html.parser')
    try:
        #如果没有这个那么就是唯一的 否则就去取第一条
        html = bs.select('.main-index')[0].select('.hottext')[0]
        href = bs.select('#navList > section > ul > li:nth-child(1) > span > a.clearfix.stitle')[0].get('href')
        body = requests.get(href, headers=headers, timeout=60)
        bts = BeautifulSoup(body.content, 'html.parser')
        href = bts.select('.reader')[0].get('href')
        return retunInfo(href)
    except:
        #单独一本
        href = bs.select('.reader')[0].get('href')
        return retunInfo(href)

#获取内容
def getContent(url):
    body = requests.get(url,headers=headers,timeout=60)
    bs = BeautifulSoup(body.content.decode('gbk'), 'html.parser')
    content = bs.select('#content')[0].get_text()
    #去掉垃圾信息
    return content.replace('style5();','').replace('style6();','').replace('吾网66721提醒书友注意休息眼睛哟','').replace('享受阅读乐趣，尽在吾网661,661是我们唯一的域名哟！','')
#查询渲染
def search(name):
    if len(name) > 0:
        try:
            urls, names, urlToName,urlIndex = getList(name)
        except Exception:
            raise
        index = ui.choicebox('选择小说章节  https://blog.csdn.net/Day_Day_No_Bug',
                             '小说阅读器 https://blog.csdn.net/Day_Day_No_Bug', names)
        url = urls[index]
        next = ui.buttonbox(title=urlToName[url]+'https://blog.csdn.net/Day_Day_No_Bug',msg=str(getContent(url)),choices=['返回搜索','下一章'])
        #从第一次进来之后就只走这里了
        while True:
            if next is not None and len(next) == 3:
                url = urlIndex[names.index(index)+1]
                index = names[names.index(index)+1]
                next = ui.buttonbox(title=index+'https://blog.csdn.net/Day_Day_No_Bug',msg=getContent(url),choices=['返回搜索','下一章'])
            elif next is not None and len(next) == 4:
                go()
            else:
                sys.exit(0)
#主搜索页面
def go():
    name = ui.enterbox(msg="输入小说名", title="小说阅读器 https://blog.csdn.net/Day_Day_No_Bug", strip=True)
    if name is not None and len(name) > 0:
        try:
            search(name)
        except Exception:
            ui.msgbox('未找到该小说 或同本小说查询频繁 ')
            go()
    else:
        sys.exit(0)


if __name__ == '__main__':
    #开始
    go()

```

 1. [爬取携程最新车票【导出EXCEL】](https://blog.csdn.net/Day_Day_No_Bug/article/details/100930832)
 2. [爬取豆瓣各种类型电影【生成图表】](https://blog.csdn.net/Day_Day_No_Bug/article/details/100932761)
 3. [爬取小红书短视频和图片【下载】](https://blog.csdn.net/Day_Day_No_Bug/article/details/101350657)