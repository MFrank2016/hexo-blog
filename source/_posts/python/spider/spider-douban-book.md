---
title: 豆瓣读书评分9.0以上的书籍都长什么样
date: 2019-04-27 10:25
tags: 
 - 爬虫
 - 豆瓣
 - Python
categorys:
 - 编程
---

## 说明

又到了周末，搞点事情，前两天把python看了看，翻了翻数据分析的书，感觉挺有意思的，用数据来说话，比拍脑袋想要强多了，可以提供一个更理性的视角，所以准备研究研究，一来是学一学python，二来是培养数据思维。

想要做数据分析，那前提是得有数据给你分析，说实话，我个人觉得这里最重要的反而是数据。技能谁都能学，互联网从来不缺学习资源，分析方法也无非那么几种，但数据却不是每个人都有的，比如每个公司的用户数据就是非常隐私且珍贵的，特别是对互联网公司而言，那些数据可以说是公司的命脉，也是形成竞争优势的核心资源。

之前做过一次数据整合分析的项目，充当了一个类似数据产品之类的角色，虽然项目进展的不太顺利，但从中学到了很多，对数据的生产，数据提取，数据清洗，数据可视化及数据分析的整体流程有一个全面的认知。作为一个java后端开发，很清楚那些数据是如何进入数据库或者日志中的。之前都说“编程不规范，同事两行泪”，在做数据清洗的过程中，更深刻的体会到了这一点。

好的设计与好的代码产生好的数据，糟糕的设计与代码产生糟糕的数据，甚至是无法使用的脏数据。更致命的是，糟糕的代码可以重构，但糟糕的数据却不行，缺失的数据无法补全，错误的数据很难或者根本无法修复。所以在后续的开发过程中，对于数据的落库也会更加谨慎。

对于个人而言，想要进行数据分析，数据来源那就只能自己去获取了，虽然有些网站上可以找到数据集，但只能当做学习时使用，想要做特定数据的分析，还得靠自己去获取，爬虫便成了获取数据的得力助手。把想要分析的数据从网上爬取下来，整理成指定格式后存入数据库或者csv文件中，便可以进行后续的数据可视化及数据分析工作了。

## 爬虫

扯了这么远，终于说到今天的主题了，因为想看看豆瓣9.0以上评分的书籍都有什么特性，想用这些数据生成一个推荐书单或者做一个随机优秀书籍推荐之类的，所以需要利用一下豆瓣读书的资源。

之前看一些文章说爬虫违法，所以特地去验证了一下真伪后，得出如下结论：

1. 不得爬取用户隐私数据
2. 不得进行商业用途
3. 不能访问过于频繁造成服务器瘫痪

所以对于这次爬虫行为而言，控制好频率就可以了。

不过，作为一条有职业操守的爬虫，还是应该遵守robots协议。所以顺便翻了翻豆瓣的[robots协议](https://www.douban.com/robots.txt)：

```xml
User-agent: *
Disallow: /subject_search
Disallow: /amazon_search
Disallow: /search
Disallow: /group/search
Disallow: /event/search
Disallow: /celebrities/search
Disallow: /location/drama/search
Disallow: /forum/
Disallow: /new_subject
Disallow: /service/iframe
Disallow: /j/
Disallow: /link2/
Disallow: /recommend/
Disallow: /doubanapp/card
Disallow: /update/topic/
Sitemap: https://www.douban.com/sitemap_index.xml
Sitemap: https://www.douban.com/sitemap_updated_index.xml
# Crawl-delay: 5

User-agent: Wandoujia Spider
Disallow: /
```

用第一个UA就能满足需求了，第二个UA是，豌豆荚？？？哈哈，虽然不知道是什么梗，先不管了。撸起袖子就是干。

找到一个9.0评分的榜单，大大减少了工作量，这样就不用先爬一下整站书籍来筛选了，看了看榜单，更新时间为2018-12-25，目前一共530本，分为22页，也就是说22次访问就能搞定了，不会给豆瓣服务器造成压力。

### 目标

目标URL：https://www.douban.com/doulist/1264675/?start=0&sort=seq&playable=0&sub_type=4

数据量：530

预计访问次数：22

数据存储：csv

抓取内容格式：书籍名称 作者 作者国籍 评分 评价人数 出版社 出版年 封面链接

### 代码

有了小目标，接下来就是用刚学的python来现学现卖了。

先来测试一下robots协议是否允许该url的访问：

```python
import urllib.robotparser

url = 'https://www.douban.com/doulist/1264675/?start=0&sort=seq&playable=0&sub_type=4'
rp = urllib.robotparser.RobotFileParser()
rp.set_url('https://www.douban.com/robots.txt')
rp.read()
can_fetch = rp.can_fetch("*",url)
print(can_fetch)
```

输出为`True`，表示允许使用“*”作为UA来爬取该url。

接下来定一下步骤：

```python
"""
auth: Frank
date: 2019-04-27
desc: 爬取豆瓣读书评分9.0以上书籍并存入csv文件

目标URL：https://www.douban.com/doulist/1264675/?start=0&sort=seq&playable=0&sub_type=4

数据量：530

预计访问次数：22

数据存储：csv

抓取内容格式：书籍名称 作者 作者国籍 评分 评价人数 出版社 出版年 封面链接
"""

# 设置headers

# robots.txt检测

# 获取网页数据

# 解析书籍数据

# 存入csv文件

```

前两步是最简单的：

```python
url = 'https://www.douban.com/doulist/1264675/?start=0&sort=seq&playable=0&sub_type=4'

logging.basicConfig(level=logging.DEBUG)

# 设置headers
headers = {'User-Agent': "*"}


# robots.txt检测
def robot_check(robots_txt_url, headers, url):
    rp = urllib.robotparser.RobotFileParser()
    rp.set_url(robots_txt_url)
    rp.read()
    can_fetch = rp.can_fetch(headers['User-Agent'], url)
    logging.debug("can_fetch:" + str(can_fetch))
    return can_fetch
```

然后是获取网页内容，这里使用requests模块来获取网页内容：

```python
def get_web_data(url):
    try:
        data = requests.get(url, timeout=3, headers=headers)
    except requests.exceptions.ConnectionError as e:
        logging.error("请求错误，url:", url)
        logging.error("错误详情：", e)
        data = None
    except:
        logging.error("未知错误，url:", url)
        data = None
    return data;
```

接下来进行网页内容解析，借助一下BeautifulSoup模块来解析网页元素。

```python

```