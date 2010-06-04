--- 
title: scrapy+sphinx搭建小说搜索引擎
date: 03/06/2010


使用scrapy实现爬虫
==================


安装scrapy
----------

详情见官方[Install guide][scrapy install]，debian下简要步骤如下:

	apt-get install python-twisted python-libxml2
	cd
	git clone git://github.com/pkufranky/scrapy.git
	echo 'export PYTHONPATH=~/scrapy' >> .bashrc
	echo 'export PATH=~/scrapy/bin:$PATH' >> .bashrc


新建工程
--------

	$ scrapy-ctl.py startproject apple
	$ mv apple novelspider
	$ tree novelspider
	novelspider
	|-- apple
	|   |-- __init__.py
	|   |-- items.py
	|   |-- pipelines.py
	|   |-- settings.py
	|   `-- spiders
	|       `-- __init__.py
	`-- scrapy-ctl.py


定义要爬取的Item
----------------

编辑`apple/items.py`:

	from scrapy.item import Item, Field
	class NovelItem(Item):
		name = Field()
		intro = Field()
		img_url = Field()
		page_url = Field()


定义爬虫
--------

以爬取[起点][qidian]小说为例, 编辑`apple/spiders/qidian.py`:

	# -*- coding: utf-8 -*-

	import re
	from scrapy.contrib_exp.crawlspider import Rule
	from scrapy.contrib.loader.processor import TakeFirst, RemoveTag

	from apple.contrib.spider import BaseCrawlSpider
	from apple.items import NovelItem
	from apple.contrib.loader import DefaultXPathItemLoader

	class QidianSpider(BaseCrawlSpider):
		name = 'qidian'
		regex_home = r'http://all.qidian.com/$'
		regex_list = r'bookstore.aspx\?.*ChannelId=-1&.*PageIndex=\d+'
		regex_item = r'Book/\d+\.aspx$'
		start_urls = [
				'http://all.qidian.com/',
				]

		rules = [
				Rule(regex_home, 'parse_home'),
				Rule(regex_list, 'parse_list'),
				Rule(regex_item, 'parse_item'),
				]

		def parse_home(self, response): # {{{
			links = self.extract_links(response, allow=self.regex_list, restrict_xpaths='.storelistbottom')

			m = re.search(ur'GoPage.*1/(\d+).*?页', response.body_as_unicode(), re.M)
			total_page = int(m.group(1))

			reqs = []
			for p in range(1, total_page+1):
				url = re.sub('PageIndex=\d+', 'PageIndex=%d' % p, links[0].url)
				req = self.make_request(url, priority=self.priority_list)
				reqs.append(req)
			return reqs
		# end def }}}

		def parse_list(self, response): # {{{
			reqs = self.extract_requests(response, priority=self.priority_item, allow=self.regex_item)
			return reqs
		# end def }}}

		def parse_item(self, response): # {{{
			loader = DefaultXPathItemLoader(NovelItem(), response=response)
			loader.add_xpath('name', 'div.book_info div.title h1')
			loader.add_xpath('intro', 'div.book_info div.intro div.txt', TakeFirst(), RemoveTag('div'))
			loader.add_xpath('img_url', 'div.book_pic img/@src')
			loader.add_value('page_url', response.url)

			item = loader.load_item()

			return item
		# end def }}}

	SPIDER = QidianSpider()


测试爬虫
--------

1. 编辑写单元测试`apple/tests/test_qidian_spider.py`:

		# -*- coding: utf-8 -*-

		import re
		from apple.spiders.qidian import QidianSpider
		from apple.tests.spider_test import SpiderTestCase

		class QidianSpiderTestCase(SpiderTestCase): # {{{
			def setUp(self):
				self.spider = QidianSpider()

			def test_parse_home(self): # {{{
				url = 'http://all.qidian.com/'
				reqs = self._parse(url)
				self.assertGreater(len(reqs), 2000, url)
				self.assertReMatch('http://.+bookstore\.aspx\?.*ChannelId=-1.*PageIndex=2', reqs[1].url, url)
			# end def }}}

			def test_parse_list(self): # {{{
				url = 'http://www.qidian.com/book/bookstore.aspx?ChannelId=-1&SubCategoryId=-1&Tag=all&Size=-1&Action=-1&OrderId=6&P=all&PageIndex=1&update=-1&Vip=-1'
				reqs = self._parse(url)
				self.assertEqual(len(reqs), 100, url)
				self.assertReMatch(self.spider.regex_item, reqs[1].url, url)
			# end def }}}

			def test_parse_item(self): # {{{
				def test(url, expected):
					item = self._parse_one(url)
					self.assertObjectMatch(expected, item, url)

				url = 'http://www.qidian.com/Book/172.aspx'
				expected = {
						'name': u'女人街的除魔事务所',
						'r:intro': u'<br />.*让我深刻体味这可怕的魔鬼吧',
						'img_url': u'http://image.cmfu.com/books/1.jpg',
						'page_url': url,
						}
				test(url, expected)
			# end def }}}

		# end class }}}

2. 使用[trial][]进行单元测试:

		$ trial apple/tests
		apple.tests.test_qidian_spider
		  QidianSpiderTestCase
			test_parse_home ...                                                    [OK]
			test_parse_item ...                                                    [OK]
			test_parse_list ...                                                    [OK]
		...


存储爬取结果
------------

1. 创建数据库`spiderdb`
	
		mysql -uroot -e "CREATE DATABASE IF NOT EXISTS spiderdb DEFAULT CHARACTER SET utf8"

2. 建表`novel_items`存储爬取的item

		$ mysql -uroot spiderdb < sql/spiderdb.sql
		$ cat sql/spiderdb.sql
		CREATE TABLE IF NOT EXISTS novel_items (
			id mediumint(9) NOT NULL auto_increment,
			name varchar(128) NOT NULL,
			intro text,
			page_url varchar(256) UNIQUE NOT NULL,
			img_url varchar(1024),

			PRIMARY KEY (id)
		) ENGINE MyISAM;

3. 创建存储pipeline

	- 爬虫返回的Item (`QidianSpider.parse_item`)，会经过一系列pipeline (可配置)进行处理
	- 下面定义一个将其存储到数据库的pipeline, 编辑`apple/pipelines.py`:

			from apple.contrib.store import StoreItem

			class StoragePipeline(StoreItem):
				def process_item(self, spider, item):
					self.store_item(item, spider)
					return item

4. 配置item经过的pipeline和存储的数据库和表

	- 在`settings.py`的末尾添加如下内容:

			ITEM_PIPELINES = [
				'apple.pipelines.StoragePipeline',
			]
			DB_URL = 'mysql://root:@localhost/spiderdb?charset=utf8&use_unicode=0'
			DB_TABLE = 'novel_items'


运行爬虫
--------

		python scrapy-ctl.py crawl qidian


使用sphinx实现搜索
==================


安装sphinx
----------

sphinx官方不支持中文分词，这里使用[sphinx-for-chinese][]的支持中文的patch，并使用了[coreseek][]的中文词典

	git clone git://github.com/pkufranky/sphinx.git
	cd sphinx
	./configure --prefix=/usr/local/sphinx
	sudo make install
	sudo src/mkdict xdict_coreseek.txt /usr/local/sphinx/xdict


配置sphinx
----------


[scrapy install]: http://doc.scrapy.org/0.9/intro/install.html
[trial]: http://twistedmatrix.com/trac/wiki/TwistedTrial "Twisted的单元测试框架"
[qidian]: http://www.qidian.com/
[coreseek]: http://www.coreseek.com/
[sphinx-for-chinese]: http://sphinx-for-chinese.googlecode.com
