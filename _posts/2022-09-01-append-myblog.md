---
layout: post
author: r3tree
title: "给博客添加搜索功能 文章目录 访问统计"
date: 2022-09-01
music-id: 
permalink: /archives/2022-09-01/1
description: "给博客添加搜索功能 文章目录 访问统计"
categories: "编程"
tags: [search, PageView]
---

## 主页添加搜索功能
在index.html顶部添加以下代码
```html
<!-- 搜索功能-->
<style>
	.search-container {
		position: relative;
		max-width: 650px; /* 整体改背景500 */
		margin: 0 auto;
	}
	input, button {
		border: none;
		outline: none;
	}
	#search-input {
		width: 0;
		border-bottom: 2px solid transparent;
		background: transparent;
		transition: .3s linear;
		position: absolute;
		top: 0;
		right: 0;
		z-index: 2;
		display: inline-block;
		height: 32px;
		padding: 0 46px 0 13px;
	}
	#search-input:focus{
		width: 200px;
		z-index: 1;
		color: #C9CACC;/* 整体改背景 多加 */
		border-bottom: 1px solid #757575;
	}
	.search-container button{
		height: 32px;
		width: 30px;
		cursor: pointer;
		position: absolute;
		top: 0;
		right: 0;
		background: #1D1F21; /* 整体改背景#FFFFFF */
	}
	.search-container button:before {
		content: url(/assets/search.svg);
	}
	.search-container li{
		display:block;
	}
</style>
<article>
	<div class="search-container">
		<input type="text" id="search-input" placeholder="Search">
		<button type="submit"></button>
		<ul id="results-container"></ul>
	</div>
</article>
```

在index.html最底部添加以下代码
```html
<!-- 搜索功能-->
<script src="/assets/js/simple-jekyll-search.min.js"></script>
<script>
	window.simpleJekyllSearch = new SimpleJekyllSearch({
		searchInput: document.getElementById('search-input'),
		resultsContainer: document.getElementById('results-container'),
		json: '{{ site.url }}/search.json',
		searchResultTemplate: '<li><a href="{url}?query={query}" title="query={query}">{title}</a></li>',
		noResultsText: 'Not found',
		limit: 10,
		fuzzy: false,
		exclude: ['Welcome']
	})
</script>
<script>
	document.addEventListener('DOMContentLoaded', function() {
		const input = document.getElementById('search-input');
		const ul = document.getElementById('results-container');
		ul.style.display = 'none';
		input.addEventListener('input', function() {
			if (input.value.trim() !== '') {
				ul.style.display = 'inline-block';
				ul.style.margin = '35px 0';
				/* ul.style.paddingLeft = '0px';左右缩行*/
			} else {
				ul.style.display = 'none';
			}
		});
	});
</script>
```
## 文章添加目录

在post.html顶部添加以下代码
```html
<link href="/assets/toc.css" rel="stylesheet" />
<div id="toc" class="floating-div" onclick="toggleSize(this)"></div>
```

在post.html最底部添加以下代码
```html
<script src="/assets/js/toc.js" type="text/javascript"></script>
<script type="text/javascript">
	$(document).ready(function() {
		$('#toc').toc();
	});
</script>
```

## 主页添加访问量
在footer.html任意位置添加以下代码
```html
<!--添加访问统计-->
<script async src="/assets/js/PageView.pure.mini.js"></script>
<span id="busuanzi_container_site_pv">本站访问量<span id="busuanzi_value_site_pv"></span>次</span>
```

## 文章添加阅读量
在post.html里`page.date | date: '%B %-d, %Y'`后面添加以下代码
```html
<span id="busuanzi_container_page_pv">
    阅读：<span id="busuanzi_value_page_pv"></span>
</span>
```