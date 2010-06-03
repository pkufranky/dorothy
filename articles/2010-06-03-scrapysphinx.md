--- 
title: scrapy+sphinx搭建音乐搜索网站
date: 03/06/2010

安装scrapy
==========

	cd
	git://github.com/insophia/scrapy.git
	echo 'export PYTHONPATH=~/scrapy' >> .bashrc
	echo 'export PATH=~/scrapy/bin:$PATH' >> .bashrc

新建工程
========

	$ scrapy-ctl.py startproject demoz
	$ tree demoz
	demoz
	|-- demoz
	|   |-- __init__.py
	|   |-- items.py
	|   |-- pipelines.py
	|   |-- settings.py
	|   `-- spiders
	|       `-- __init__.py
	`-- scrapy-ctl.py

定义要爬取的Item
================

编辑`items.py`:

	from scrapy.item import Item, Field
	class NovelItem(BaseItem):
		name = Field(required=True)
		category = Field()
		keyword = Field()
		tag = Field()
		language = Field()
		intro = Field()
		star = Field()
		orig_id = Field()
		res_url = Field()
		format = Field(post_output_processor=unicode.lower)
		img_url = Field()

		page_url = Field(required=True)
		updated_at = Field()
		crawled_at = Field(extra=True)
		is_dropped = Field(extra=True) # whether this item is dropped
		click_total = Field()
		click_month = Field()
		click_week = Field()
		recommend_total = Field()
		recommend_month = Field()
		recommend_week = Field()
		word_count = Field()
		author = Field()
		property = Field()
		authorize_status = Field()
		progress = Field()
		public_updated_chapter = Field()
		public_updated_at = Field()
		vip_updated_chapter = Field()
		vip_updated_at = Field()


<script src="http://gist.github.com/365482.js?file=test.php"></script>


定义爬虫
========

