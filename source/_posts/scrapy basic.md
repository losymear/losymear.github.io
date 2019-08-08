----
title: scrapy basic
date: 2019-08-06 15:14:49
tags:
    - python
    - 爬虫
toc: true

----

[scrapy](https://scrapy.org/)是一个通用的爬虫框架，基于python，建议阅读[官方tutorial](https://docs.scrapy.org/en/latest/intro/tutorial.html)。
<!-- more -->
scrapy提供了同名的命令行工具`scrapy`，运行`scrapy -h`查看可用命令。
scrapy的机制见[Architecture overview](https://docs.scrapy.org/en/latest/topics/architecture.html)。


## 推荐配置使用的python库
- `furl`  is a small Python library that makes parsing and manipulating URLs easy.


## 示例说明
创建新scrapy项目可以用命令`scrapy startproject projName`，会生成如下目录结构：
```sh
projName
├── projName
│   ├── __init__.py
│   ├── __pycache__
│   ├── items.py
│   ├── middlewares.py
│   ├── pipelines.py
│   ├── settings.py
│   └── spiders
│       ├── __init__.py
│       └── __pycache__
└── scrapy.cfg
```
示例代码如下，命名为`quotes_spider.py`并放到`spiders`文件夹：
```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'
    start_urls = [
        'http://quotes.toscrape.com/tag/humor/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.xpath('span/small/text()').get(),
            }

        next_page = response.css('li.next a::attr("href")').get()
        if next_page is not None:
            yield response.follow(next_page, self.parse)
```

然后通过命令`scrapy runspider quotes_spider.py -o quotes.json`启动命令，会将结果放到`quotes.json`文件中。`runspider`命令会找到文件`quotes_spider.py`中的`scrapy.Spider`定义，开始爬取。 起始页面是Spider类里的`start_urls`列表，每爬取一个页面后，会将一个`spider.Response`对象传入到`parse`回调中。
在`parse`回调中，先是用css选择器`'div.quote'`找到当前页的所有名言，用`yield data_as_dict`保存该条记录；然后通过css选择器`'li.next a::attr("href")'`看是否有下一页按钮，如果有则通过`yield response.follow(new_url, new_parse)`将新页面加入到待爬取队列中。
注意命令行指定了`-o quotes.json`，所以会保存到json文件中。这是通过[feed exports](https://docs.scrapy.org/en/latest/topics/feed-exports.html#topics-feed-exports)机制，也可以保存为XML、CSV或者其它格式，甚至自己写一个[Item Pipeline](https://docs.scrapy.org/en/latest/topics/item-pipeline.html#topics-item-pipeline)来存入到数据库。

scrapy爬虫能不断爬取新链接，并保存数据的重要机制是：**在回调函数中，如果`yield`了一个`Request`对象，则会将这个对象加入到待爬取的队列中；如果`yield`了一个`dict`，则会保存数据**。



## [settings](https://docs.scrapy.org/en/latest/topics/settings.html)
可以通过配置项来控制爬虫的行为。
不同来源的优先级（由高到低）：
- 命令行 `-s`或`--set`，如`scrapy crawl myspider -s LOG_FILE=scrapy.log`。
- spider中的`custom_settings`字典。
- project settings module. 一般是在`settings.py`中设置。
- scrapy命令一般都有它自己的默认设置，会覆盖全局设置。可以查看对应类的`default_settings`属性。
- default global settings 在`scrapy.settings.default_settings`中。

[built-in settings reference](https://docs.scrapy.org/en/latest/topics/settings.html#topics-settings-ref)部分举例：
- `DOWNLOAD_DELAY` 爬取同一网站，两个网页的间隔时间。也可以修改spider的`download_delay`属性。
- `RANDOMIZE_DOWNLOAD_DELAY` 不等待固定时间，而是在`0.5*DOWNLOAD_DELAY`和`1.5*DOWNLOAD_DELAY`之间。
- `ITEM_PIPELINES` 设置要使用的pipeline。
- `LOG_*` 和日志有关的设置项。

## [Selector](https://docs.scrapy.org/en/latest/topics/selectors.html)
如果需要从爬取的HTML文本中解析出想要的数据（如下一页的链接），就需要一个HTML解析库。 很多人会考虑使用`BeautiflSoup`，它很强大，但有一个缺点：太慢。scrapy的Selector对[parsel](https://parsel.readthedocs.io/en/latest/usage.html#using-selectors)库做了简单封装（为了更好的集成`scrapy.http.Response`），建议学习下。
Selector支持CSS和XPath两种方式，如可以使用`response.css('span::text').get()`或者`response.selector.xpath('//span/text()').get()`。 `CSS`的方式背后也是使用`XPath`，并且`Xpath`更加灵活。

调试Selector时，可以使用[scrapy shell](https://docs.scrapy.org/en/latest/topics/shell.html)，当然也可以在浏览器中使用开发者工具。


## 核心类

### [scrapy.Spider](https://docs.scrapy.org/en/latest/topics/spiders.html#scrapy-spider)
一个类，定义了爬虫规则（爬取链接、数据处理、follow规则），实现自定义爬虫时需要继承该类。
scrapy爬取流程如下：
1. 首先生成初始的`Request`对象，用于爬取数据，并指定回调函数来处理这些请求的返回值。默认通过`start_requests()`方法来生成`Request`对象。
2. 在回调函数中，处理`Request`对应的`Response`，并返回提取数据的dict、`Item`对象、`Request`对象，或者这些对象的iterable。返回的`Request`同样会包含一个回调函数（可以与当前函数一样也可不同），并且会被Scrapy继续爬取。
3. 在回调函数中，一般会使用`Selector`来处理返回值，当然也可以选择其它工具如`BeautifulSoup`、`lxml`，并用这些解析到的数据生成item。
4. 最后，spider返回的数据一般会被持久化到数据库（通过`Item Pipeline`）或者写入到文件（`Feed exports`）。

`Spider`类的需要了解的属性和方法：
- `name`属性。 scrapy用于标识一个spider，需要唯一。
- `allowed_domains`。 可选属性，允许访问的域名列表。
- `start_urls`。 默认的`start_request`实现会从`start_urls`中生成初始`Request`。
- `custom_settings`属性。 一个dict，当运行spider时会覆盖project级别的设置，比如设置User_Agent等，可选项见[文档](https://docs.scrapy.org/en/latest/topics/settings.html#topics-settings-ref)。
- `settings`属性。 用于运行spider的配置，是`Settings`的实例。
- `start_requests()` 。 scrapy使用该方法来生成spider初次执行时使用的url。可以将它实现为generator。默认的实现为使用`start_urls`里的每个url生成`Request(url, dont_filter=True)`。
- `parse(response)` 。 用于处理response的默认方法（如果没有明确指定）。 这个方法需要处理response并返回抽离出的数据或者返回一些要爬取的链接。返回`Request`的iterable或者`Item`对象的dict。



### [`scrapy.http.Request`](https://docs.scrapy.org/en/latest/topics/request-response.html#request-objects)
这里请回顾前面提到的[架构图](https://docs.scrapy.org/en/latest/topics/architecture.html)。
`Request`对象一般由Spider生成并交给Engine。当Request到达Downloader后，Downloader会执行网络请求，返回一个封装过的`Response`对象给spider，spider会随后调用callback来处理Response。
签名为`class scrapy.http.Request(url[, callback, method='GET', headers, body, cookies, meta, encoding='utf-8', priority=0, dont_filter=False, errback, flags, cb_kwargs])`，部分解释：
- `url`  HTTP请求的url
- `method`  HTTP方法，默认为`GET`请求。
- `callback`  回调函数，用于处理返回的`Response`。 如果不声明则使用Spider类里的`parse`方法。
- `errback`  在处理Request发生异常时，会调用该回调。它的第一个参数是[Twisted Failure](https://twistedmatrix.com/documents/current/api/twisted.python.failure.Failure.html)。
- `dont_filter`  该参数用于告知scheduler，不用对该url做去重处理。 比如一个列表请求`list`有10个详情页`detail?id=1`、`detail?id=2`...`detail?id=10`，如果某次请求只请求了详情页1~5并且允许过滤`list`这个请求。当意外中断后重新执行时，由于scheduler发现`list`已经请求过，不再请求，那么详情页6~10就会被忽略。  注意这和[DUPEFILTER](https://docs.scrapy.org/en/latest/topics/settings.html?highlight=duplicate#dupefilter-class)机制有关。
- `cb_kwargs` 可以在`Response`中获取到， 这在需要跨多页来获取最终item的情况下很有用。 不过这只在1.7以上版本可用，旧版本可以通过`meta`来实现。[使用示例](https://docs.scrapy.org/en/latest/topics/request-response.html#passing-additional-data-to-callback-functions)。 注：1.7之后不使用`Request.meta`来传递参数，而只是用于组件间（如middlewares和extensions）交换信息。
- `priority` 请求的优先级。scheduler会根据该参数来决定调度顺序。
`Request`对象有和构造器参数同名的属性：`url`、`method`、`headers`... `Request.meta`可以有任意的键值，不过一些特殊键值会被Scrapy和它的内置扩展识别。 比如`download_timeout`表示Downloader会等待的超时时间、`max_retry_times`表示请求的最大重试次数。其它特殊属性见[列表](https://docs.scrapy.org/en/latest/topics/request-response.html#request-meta-special-keys)。

#### `Request`内置子类
Scrapy实现了`Request`的子类`FormRequest`，用于模拟HTTP的表单POST请求。使用示例：
```python
return [FormRequest(url="http://www.example.com/post/action",
                    formdata={'name': 'John Doe', 'age': '27'},
                    callback=self.after_post)]
```

另一个子类`JSONRequest`则是简化了HTTP POST请求，它增加了额外两个参数： `class scrapy.http.JSONRequest(url[, ... data, dumps_kwargs])`。使用`JSONRequest`会自动将请求头`Content-Type`设为`application/json`，将`Accept`设置为`application/json, text/javascript, */*; q=0.01`。
如果给定参数`data`且未给定`Request.body`，那么会自动将`method`设置为`POST`。
示例：
```python
data = {
    'name1': 'value1',
    'name2': 'value2',
}
yield JSONRequest(url='http://www.example.com/post/action', data=data)
```

### [`scrapy.http.Response`](https://docs.scrapy.org/en/latest/topics/request-response.html#response-objects)
`Response`对象表示一个HTTP响应，一般是由Downloader返回，交由Spider处理。
签名为`class scrapy.http.Response(url[, status=200, headers=None, body=b'', flags=None, request=None])`。
- `status` 响应的状态码，如200、404等。
- `body`  响应体。如果想要获得响应体的字符串，需要使用`Response`的子类如`TextResponse`，这些子类有属性`response.text`可以获取响应体文本。如果响应体是个JSON，则需要使用`json.loads(response.text)`。
- `request` 生成该Response的Request。注意如果发生重定向会导致request改变。
- `meta` 即原始的`Request.meta`，不会受重定向影响。

方法有：
- `urljoin(url)` 等价于`urlparse.urljoin(response.url, url)`
- `follow(url, callback=None, method='GET', headers=None, body=None, cookies=None, meta=None, encoding='utf-8', priority=0, dont_filter=False, errback=None, cb_kwargs=None)` 返回一个`Request`对象。 接收与`Request.__init__`相同的参数，不过`url`是一个相对路径。

#### `Response`内置子类
`TextResponse`在`Response`的基础上增加了编码的功能：
- `text`属性。等价于`response.body.decode(response.encoding)`，不过会在第一次调用后缓存。
- `encoding`属性。 表示响应体的编码格式。 1). 解析响应头`Content-Type`，如果不合法则进入下一步； 2).  查看在响应体中声明的编码。这只对`HtmlResponse`和`XmlResponse`有效，`TextResponse`本身未实现该机制； 3). 从响应体中推断，这种方式很脆弱，但是如果其它机制都无法推断的话只能通过该方式。
- `selector`属性。 一个`Selector`实例。
- `xpath(query)`  即`TextResponse.selector.xpath(query)`。
- `css(query)` 即`TextResponse.selector.css(query)`。

此外还有`HtmlResponse`和`XmlResponse`，这些都是`TextResponse`的子类，能够获取响应体声明的编码。


## 数据存储：Item和pipeline

### [Items](https://docs.scrapy.org/en/latest/topics/items.html#topics-items)
*只是简单地使用`scrapy`的话，可以跳过。*
爬取网页的目标一般是保存结构化的数据。前面的示例中使用了python内置的字典类型`dict`，不过在大型项目中频繁使用使用`dict`很容易出错（比如因为typo），因此scrapy提供了`Item`类，可以在`parse`回调中yield一个`Item`实例而不是`dict`。
简单示例如下：
```python
import scrapy

class Product(scrapy.Item):
    name = scrapy.Field()
    price = scrapy.Field()
    stock = scrapy.Field()
    tags = scrapy.Field()
    last_updated = scrapy.Field(serializer=str)

def parse():
    # 其它处理
    product = Product(name='Desktop PC', price=1000)
    product.keys()  # ['price', 'name']
    'last_updated' in product # False
    'last_updated' in product.fields # True
    yield product
```


### [Item Loader](https://docs.scrapy.org/en/latest/topics/loaders.html)
*只是简单地使用`scrapy`的话，可以跳过。*
`Item`提供了存储数据结果的容器，而Item Loader则提供了填充容器的机制。这比使用Item的dict-like API方便多了。

> Item Loaders are designed to provide a flexible, efficient and easy mechanism for extending and overriding different field parsing rules, either by spider, or by source format (HTML, XML, etc) without becoming a nightmare to maintain.

使用示例：
```python
from scrapy.loader import ItemLoader
from myproject.items import Product

def parse(self, response):
    l = ItemLoader(item=Product(), response=response)
    l.add_xpath('name', '//div[@class="product_name"]')
    l.add_xpath('name', '//div[@class="product_title"]')
    l.add_xpath('price', '//p[@id="price"]')
    l.add_css('stock', 'p#stock]')
    l.add_value('last_updated', 'today') # you can also use literal values
    return l.load_item()
```

注意示例中`name`字段被添加了两次，ItemLoader会使用合适的机制来合并多个结果。
对field的处理： `ItemLoader`为每一个field都有input processor和output processor。input processor会在接收到通过`add_xpath()`等方法提取到的值（记为`field1_data_idx`，idx为1,2... 因为前面提到过一个field可以添加多次）后执行，然后将处理得到的值（记为`field1_data_idx_input_processed`）存到ItemLoader内部的list中。当所有的数据都收集好后，对于`field1`，`ItemLoader`内部会有个list内容为`[field1_data_1_input_processed,field1_data_2_input_processed...]`。 当调用`ItemLoader#load_item`时，output processor会执行，对于每一个field，它接收对应的内部list为参数， 返回值则是该field对应的最终结果。
input processor和output processor都是callable对象。
自定义`ItemLoader`示例：
```python
from scrapy.loader import ItemLoader
from scrapy.loader.processors import TakeFirst, MapCompose, Join

class ProductLoader(ItemLoader):

    default_output_processor = TakeFirst()

    name_in = MapCompose(unicode.title)
    name_out = Join()

    price_in = MapCompose(unicode.strip)

    # ...
```
可以看出，对于每一个field，`ItemLoader`可以声明`$fieldName_in`和`$fieldName_out`这两个processor。如果有field没有声明对应的input/output processor，则会使用默认处理器`default_input_processor`和`default_output_processor`。


自定义processor示例：
```python
import scrapy
from scrapy.loader.processors import Join, MapCompose, TakeFirst
from w3lib.html import remove_tags

def filter_price(value):
    if value.isdigit():
        return value

class Product(scrapy.Item):
    name = scrapy.Field(
        input_processor=MapCompose(remove_tags),
        output_processor=Join(),
    )
    price = scrapy.Field(
        input_processor=MapCompose(remove_tags, filter_price),
        output_processor=TakeFirst(),
    )
```
注意在`Field`中定义的process优先级低于ItemLoader里的`field_in`、`field_out`。


### [Item Pipeline](https://docs.scrapy.org/en/latest/topics/item-pipeline.html)
当在Spider中`yield`了一个item后，这个item会交给一组Item Pipeline依次处理。
每个Item Pipeline会接收item，对它执行一些操作，然后决定是否将它传给下一个Pipeline或者结束执行。
典型示例：1) 清洗HTML数据； 2) 验证爬取的数据（如是否包括指定field）； 3) 检验是否重复，重复则退出Pipeline； 4) 保存数据。
每个item pipeline是个Python类，需要实现一些方法，示例：
```python
import pymongo


# 定义了一个Item Pipeline组件，会将item写入到Jmongodb
class MongoPipeline(object):

    collection_name = 'scrapy_items'

    def __init__(self, mongo_uri, mongo_db):
        self.mongo_uri = mongo_uri
        self.mongo_db = mongo_db

    # 如果存在该方法，会用该方法从一个`Crawler`中来创建pipeline实例；`Crawler`对象可以访问Scrapy核心组件如settings、signals；
    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            mongo_uri=crawler.settings.get('MONGO_URI'),
            mongo_db=crawler.settings.get('MONGO_DATABASE', 'items')
        )

    # 如果存在，会在spider启动时调用
    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.mongo_uri)
        self.db = self.client[self.mongo_db]

    # 如果存在，会在spider关闭时调用
    def close_spider(self, spider):
        self.client.close()

    # 必须存在，用于处理item。 需要返回`dict`或者`Item`，或者抛出`DropItem`异常表示退出Pipeline。
    def process_item(self, item, spider):
        self.db[self.collection_name].insert_one(dict(item))
        return item
```

如果需要证Item Pipeline组件生效，需要在settings的`ITEM_PIPELINES`中声明，例如：
```python
custom_settings = {
    'ITEM_PIPELINES' = {
        'myproject.pipelines.PricePipeline': 300,
        'myproject.pipelines.JsonWriterPipeline': 800,
    }
}
```
其中整数值为0~1000，表示优先级，即从数值小的Pipeline开始执行。



### [Feed exports](https://docs.scrapy.org/en/latest/topics/feed-exports.html)
*建议跳过该节*。
**不建议使用。 Feed export适用于一些标准格式如JSON。如果需要使用自定义数据格式或者存储到数据库，那么直接在Item Pipeline中处理比写一个Custom Exporter简单的多。**
爬虫的一个目的是存储结构化的数据，并且一般要求符合某种格式来让其它程序处理。Scrapy提供了*Feed Export*支持，允许你使用多种序列化格式或者存储机制在保存items。比如前面示例中的`scrapy runspider quotes_spider.py -o quotes.json`，就是使用了Scrapy自带的Feed  export类型`JSON`。
默认支持的序列化格式有：JSON、JSON lines、CSV、XML、Marshal、Pickle。

Feed export是通过[Item Export](https://docs.scrapy.org/en/latest/topics/exporters.html)来生效的，比如JSON类型就是靠内置的`scrapy.exporters.JsonItemExporter`类。



## Middleware
### [Downloader Middleware](https://docs.scrapy.org/en/latest/topics/downloader-middleware.html#topics-downloader-middleware)
Downloader Middleware会对Scrapy的request/response流程进行修改。要想使用一个Downloader Middleware可以配置`DOWNLOADER_MIDDLEWARES`：
```python
DOWNLOADER_MIDDLEWARES = {
    'myproject.middlewares.CustomDownloaderMiddleware': 543,
}
```

Scrapy内置了一些[Middleware](https://docs.scrapy.org/en/latest/topics/downloader-middleware.html#built-in-downloader-middleware-reference)，部分举例：
- `DefaultHeadersMiddleware`会让所有的请求加上`DEFAULT_REQUEST_HEADERS`配置中定义的请求头。
- `DownloadTimeoutMiddleware`会根据`DOWNLOAD_TIMEOUT`或者`Request`的`download_timeout`进行超时处理。
- `RetryMiddleware`会在遇到500等状态码时重试。


### [Spider Middleware](https://docs.scrapy.org/en/latest/topics/spider-middleware.html#topics-spider-middleware)
另一种Middleware形式，和Downloader Middleware的区别见这个[stackoverflow问题](https://stackoverflow.com/questions/17872753/what-is-the-difference-between-scrapys-spider-middleware-and-downloader-middlew)。


## [common practice](https://docs.scrapy.org/en/latest/topics/practices.html)

### Running multiple spiders in the same process
示例
```python
import scrapy
from scrapy.crawler import CrawlerProcess

class MySpider1(scrapy.Spider):
    # Your first spider definition
    ...

class MySpider2(scrapy.Spider):
    # Your second spider definition
    ...

process = CrawlerProcess()
process.crawl(MySpider1)
process.crawl(MySpider2)
process.start() # the script will block here until all crawling jobs are finished
```

### [AutoThrottle extension](https://docs.scrapy.org/en/latest/topics/autothrottle.html)
一个扩展，自动控制爬虫速度。

### [jobs](https://docs.scrapy.org/en/latest/topics/jobs.html)
scrapy可以暂停一个爬虫，并在之后重新启动。  通过将一些关键数据保存到`$JOBDIR`文件夹中： `scrapy crawl somespider -s JOBDIR=crawls/somespider-1`。


### [在jupyter notoebook中使用scrapy](https://stackoverflow.com/questions/40856730/how-to-run-scrapy-project-in-jupyter)

### [如何在程序中运行Spider类](https://stackoverflow.com/questions/13437402/how-to-run-scrapy-from-within-a-python-script)
- `CrawlerProcess`
- `CrawlerRunner`
另外`scrapy crawl`子命令的实现可以见[crawl.py](https://github.com/scrapy/scrapy/blob/master/scrapy/commands/crawl.py)。