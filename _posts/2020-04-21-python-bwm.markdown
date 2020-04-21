---
title: "爬取汽车之家缩略图"
date: 2020/4/21 20:55:28  
category: python
tags: [python, scrapy]
excerpt: 主要是联系scrapy下的ImagesPipeline类的使用与重写
---

bwmSpider.py

```python
# -*- coding: utf-8 -*-
import scrapy
from bwm.items import BwmItem


class BwmspiderSpider(scrapy.Spider):
    name = 'bwmSpider'
    allowed_domains = ['autohome.com.cn']
    start_urls = ['https://car.autohome.com.cn/pic/series/2723.html#pvareaid=3454507']

    def parse(self, response):
        divs = response.xpath('//div[@class="uibox"]')[1:]
        # 获取到了每一个想要提取的div，现在要获取其中title和图片的url
        for div in divs:
            title = div.xpath('./div[1]//a[1]/text()').get()
            # 这一块不需要获取显示更多的
            # pic = div.xpath('./div[2]//li[not(@class)]/a/@href').getall()---错了，这个获取的是href我需要jpg
            pic = div.xpath('./div[2]//li[not(@class)]//img/@src').getall()
            image_urls = list(map(lambda url: "https:" + url, pic))
            yield BwmItem(title=title, image_urls=image_urls)

```

items.py

```python
import scrapy


# 利用scrapy的image下载器的话，就必须要images和image_urls两个属性
class BwmItem(scrapy.Item):
    title = scrapy.Field()
    image_urls = scrapy.Field()
    images = scrapy.Field()
```

pipelines.py

```python
# -*- coding: utf-8 -*-

# Define your item pipelines here
#
# Don't forget to add your pipeline to the ITEM_PIPELINES setting
# See: https://doc.scrapy.org/en/latest/topics/item-pipeline.html
from scrapy.exporters import JsonLinesItemExporter
import os
from urllib import request
import re
from scrapy.pipelines.images import ImagesPipeline
from bwm.settings import IMAGES_STORE


class BwmPipeline(object):
    def __init__(self):
        # 只执行一次，生命周期长
        # os.path.dirname(__file__)--获取到的是这个文件的当前目录
        # print(os.path.dirname(__file__))
        # os.getcwd()--获取到的是这个文件的上一层目录
        # print(os.getcwd())
        self.path = os.path.dirname(__file__) + "\images"
        print(self.path)
        if not os.path.exists(self.path):
            os.makedirs(self.path)
        # self.fp = open("bwm.json", "wb")
        # self.exporter = JsonLinesItemExporter(self.fp, ensure_ascii=False, encoding="utf-8")
        # self.fp.write(b"[")

    def process_item(self, item, spider):
        self.download_image(item)
        return item

    def close_spider(self, spider):
        print("下载完毕")
        # self.fp.write(b"]")
        # self.fp.close()

    def download_image(self, item):
        # 我先试着用原始方法下载图片--每一个item就是一个分类，进来先创建文件夹进行分类
        category_path = self.path + "\\" + item["title"]
        if not os.path.exists(category_path):
            os.makedirs(category_path)
        for url in item["pic"]:
            print(url)
            filename = category_path + "\\" + re.search(r"__(.+?\.jpg)", url).group(1)
            # 下载图片
            request.urlretrieve(url=url, filename=filename)
            print(filename + "-----------下载成功")


# 因为下载文件的话，是写好的全部下载在full文件夹下，现在想改文件夹，所以就要重写scrapy.ImagesPipeline
# 一个是file_path   一个是get_media_requests
class BWMImagesPipeline(ImagesPipeline):
    # 这个函数是在下载之前，也就是下载请求，在这里面加上属性类别去区分
    def get_media_requests(self, item, info):
        request_objs = super(BWMImagesPipeline, self).get_media_requests(item, info)
        for request_obj in request_objs:
            request_obj.category = item["title"]
        # 再将所有的requests请求返回
        return request_objs

    def file_path(self, request, response=None, info=None):
        path = super(BWMImagesPipeline, self).file_path(request, response, info)
        print(path)
        # path在原来是full+pid(一个哈希值).jpg--在这一块获取scrapy已经给好的名字，然后修改其存储地址就行
        file_name = path.replace("full/", "")
        print(file_name)
        dir = os.path.join(IMAGES_STORE, request.category)
        print(dir)
        if not os.path.exists(dir):
            os.makedirs(dir)
        path = os.path.join(dir, file_name)
        print(path)
        return path

```

settings.py

```python
# -*- coding: utf-8 -*-
import os

# Scrapy settings for bwm project
#
# For simplicity, this file contains only settings considered important or
# commonly used. You can find more settings consulting the documentation:
#
#     https://doc.scrapy.org/en/latest/topics/settings.html
#     https://doc.scrapy.org/en/latest/topics/downloader-middleware.html
#     https://doc.scrapy.org/en/latest/topics/spider-middleware.html

BOT_NAME = 'bwm'

SPIDER_MODULES = ['bwm.spiders']
NEWSPIDER_MODULE = 'bwm.spiders'

# Crawl responsibly by identifying yourself (and your website) on the user-agent
# USER_AGENT = 'bwm (+http://www.yourdomain.com)'

# Obey robots.txt rules
ROBOTSTXT_OBEY = False

# Configure maximum concurrent requests performed by Scrapy (default: 16)
# CONCURRENT_REQUESTS = 32

# Configure a delay for requests for the same website (default: 0)
# See https://doc.scrapy.org/en/latest/topics/settings.html#download-delay
# See also autothrottle settings and docs
# DOWNLOAD_DELAY = 3
# The download delay setting will honor only one of:
# CONCURRENT_REQUESTS_PER_DOMAIN = 16
# CONCURRENT_REQUESTS_PER_IP = 16

# Disable cookies (enabled by default)
# COOKIES_ENABLED = False

# Disable Telnet Console (enabled by default)
# TELNETCONSOLE_ENABLED = False

# Override the default request headers:
DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'Accept-Language': 'en',
    'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.113 Safari/537.36'
}

# Enable or disable spider middlewares
# See https://doc.scrapy.org/en/latest/topics/spider-middleware.html
# SPIDER_MIDDLEWARES = {
#    'bwm.middlewares.BwmSpiderMiddleware': 543,
# }

# Enable or disable downloader middlewares
# See https://doc.scrapy.org/en/latest/topics/downloader-middleware.html
# DOWNLOADER_MIDDLEWARES = {
#    'bwm.middlewares.BwmDownloaderMiddleware': 543,
# }

# Enable or disable extensions
# See https://doc.scrapy.org/en/latest/topics/extensions.html
# EXTENSIONS = {
#    'scrapy.extensions.telnet.TelnetConsole': None,
# }

# Configure item pipelines
# See https://doc.scrapy.org/en/latest/topics/item-pipeline.html
# 这个打开就可以用pipelines
ITEM_PIPELINES = {
    # 要启动人家的pipe了
    # 'bwm.pipelines.BwmPipeline': 300,
    # 要启动人家的pipe了
    #'scrapy.pipelines.images.ImagesPipeline': 1,
    # 我为了分类重写了ImagesPipeline类，现在要用我的了
    'bwm.pipelines.BWMImagesPipeline':1
}

# Enable and configure the AutoThrottle extension (disabled by default)
# See https://doc.scrapy.org/en/latest/topics/autothrottle.html
# AUTOTHROTTLE_ENABLED = True
# The initial download delay
# AUTOTHROTTLE_START_DELAY = 5
# The maximum download delay to be set in case of high latencies
# AUTOTHROTTLE_MAX_DELAY = 60
# The average number of requests Scrapy should be sending in parallel to
# each remote server
# AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0
# Enable showing throttling stats for every response received:
# AUTOTHROTTLE_DEBUG = False

# Enable and configure HTTP caching (disabled by default)
# See https://doc.scrapy.org/en/latest/topics/downloader-middleware.html#httpcache-middleware-settings
# HTTPCACHE_ENABLED = True
# HTTPCACHE_EXPIRATION_SECS = 0
# HTTPCACHE_DIR = 'httpcache'
# HTTPCACHE_IGNORE_HTTP_CODES = []
# HTTPCACHE_STORAGE =a 'scrapy.extensions.httpcache.FilesystemCacheStorage'
# 设置图片下载地址，如果为文件那么属性为FILES_STORE
IMAGES_STORE = os.path.dirname(__file__) + "\images"

```

文件分布：

![](https://github.com/TIGER-FF/tiger-ff.github.io/blob/master/_posts/2018-04-21-python-bwm.png)
