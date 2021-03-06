这个文件记录了编写`36kr_newsflash_reader`的过程。

做完一件事最好有个总结，好处有三:

* 备忘

* 总结归纳，加深理解

* 提升写作和表达能力

---

1. 首先是初步形成想法。因为不久前看了[慕课网的这个公开课](http://www.imooc.com/learn/563)，然后前天在微博上偶然发现了[36kr的newsflash](http://new.36kr.com/newsflashes)，简单的一个新闻列表，就和RSS一样，然后就本能地想要做一个脚本把这些资讯搞下来，在终端里面敲个回车就能看，浏览器都省得开了，简单快捷。

	刚开始没想到要涉及这么多知识点，以为跟着视频一步一步做就能搞定，预期2小时完成，事实证明图样图森破:(，虽然最后结果还是证明了python的简洁优雅:)

2. 根据第`1`点当中公开课的重点，以及[liaoxuefeng](http://www.liaoxuefeng.com/)的python教程，开始学习并测试`urllib`，代码参见`test`文件夹下的`test_1.py`,`test_2.py`。

	使用`urllib`的模块可以非常方便地操作URL，以`Get`为例，只需要像`test_1.py`一样把URL的字符串对象传入`request.urlopen()`方法，并设置对象接受返回的数据，即可得到`HTTP响应`。

	`test_2.py`则是更进一步，使用`request`对象的`add_header()`方法，将python的Get请求伪装成浏览器的Get请求(不同的Header会返回不同的对象，如PC风格、移动风格)。最后我把返回的respond？对象保存为本地html文件。

3. 初步掌握urllib后，回到公开课，由于我的需求是：只要该页面上的新闻，而不需要多个网页(后来才知道这应该叫做spider，而不是crawler，参见[这里](http://www.admin5.com/article/20080825/100523.shtml))，因此直接从入手解析器。

	视频当中使用的是[BeautifulSoup4](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)(一开始安装有问题，在Google和Stack Overflow的帮助下安装成功)。根据其[Documentation](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)，完成初步学习，参见`test_bs4.py`和`test_bs4.html`

	然后，最大的困难出现了。。。使用Get+bs4的方法总是出现Error：能把html保存，但无法用bs4解析。由于保存下来的html很奇怪，接合Chrome解析调试的时候突然想到，这里面有很多js，这尼玛该不会是动态和静态的区别吧！？Google一下 爬虫 python 动态，果然。。掉坑里了，公开课里面的方法只适用于静态网页。(`test_3.py`是含有bug的代码)

	更多说明请参见文末的**[Reference]**。

4. 再次感谢Google，根据 [这里](http://www.ahlinux.com/python/15816.html)， [这里](https://www.zhihu.com/question/21471960/answer/81061538) 和 [这里](http://xlzd.me/2015/12/19/python-crawler-04), 总结得到了大致思路。为了节省资源，不使用模拟浏览器环境的方法(搭建js的运行环境，浏览器引擎)，而是用`Chrome的开发者工具`来检测分析`网络请求`。

	在开发者工具中，点开`Network`，仔细查找后发现在`Name`是`newsflashes.json`的那一项，点开它的`Preview`，确实能看到当前页面的那些`hash_title`和`description_text`。

	再回到`Headers`选项卡，得到真正的内容提供者`http://new.36kr.com/newsflashes.json`。接着再寻找一些信息，就能打包发送请求了。

	在上面的参考中，都提到了Headers中的`Form Data`。其实这个一般出现在登陆界面，就是要提交表单信息的时候，比如检测[这里](https://passport.weibo.cn/signin/login?entry=mweibo&res=wel&wm=3349&r=http%3A%2F%2Fm.weibo.cn%2F) 和[这里](https://www.zhihu.com/#signin) 都会发现`Form Data`。而我在要爬取的[网页](http://new.36kr.com/newsflashes) 中检测了半天也没用`Form Data` ，我猜应该是因为不需要登陆/没有验证码的原因。我只能先试一试只有url，也就是只有  
	`req = request.Request('http://new.36kr.com/newsflashes.json')`  
	的情况下能否成功。

	这时我又注意到了在Request Headers下面有`User-Agent:Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.87 Safari/537.36`，和之前伪装成移动端访问的包头不太一样，所以我决定也把这段也更新一下。

5. 思路清楚了，下面就是如何用python实现了，代码参见`test_4.py`。由于我想要做的是专用脚本，针对单一网页的spider，功能单一目标明确，所以可以尽可能简单。

	把刚才得到的新的URL和Header放到request对象里面，然后使用`request.urlopen()`方法，直接就返回JSON格式的数据了，比想象的还要简单。

	好了，真正的信息源已经被挖出，接下来是怎么处理并提取需要的信息了(我只需要title，不需要简述以及其他附加信息)。

	读写JSON数据可以参考官方文档，也可以参考[这里](http://python3-cookbook.readthedocs.org/zh_CN/latest/c06/p02_read-write_json_data.html). 首先需要引入一些包和模块。接着，我把返回的数据存储成了`test_4.json`文件，然后用`json.loads()`方法把这个文件读取到`json_data`这个对象里面，并添加参数使其规格化，可读性更好。这里其实是把这个json文件解码成了一个`dict`，而这个dict里面又可以分解成更多dict和list。观察后，根据key得到value，根据index来索引list，即可提取最终需要的信息。

7. 把上面的`test_4.py`简化一下，就得到了最终的`36kr_newsflash_reader.py`. 代码只有十几行，再次佩服python的哲学：**简单优雅**

<br/><br/><br/>

* PS: 每页只有30条新闻，如果要翻看下一页，则需要再次inspect网页，分析得到请求资源的地址是如下的结构(代码请参见`test_5_two_pages.py`)：

	```python

	'http://new.36kr.com/newsflashes.json?b_id=' + str(news_id) + '&d=next'

	```

	不过最终版本还是舍弃了这一功能，因为如果要看更多内容/标题，直接上浏览器就得了，太多的东西在Terminal里面会影响阅读体验，过犹不及。

## Refernce

[imooc ](http://www.imooc.com/learn/563)

[BeautifulSoup ](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)

[Spider与crawler不同点 ](http://www.admin5.com/article/20080825/100523.shtml)

[分析动态网页_1 ](http://www.ahlinux.com/python/15816.html)

> 用Python实现常规的静态网页抓取时，往往是用urllib2来获取整个HTML页面，然后从HTML文件中逐字查找对应的关键字。但是，在动态页面中，所显示的内容往往不是通过HTML页面呈现的，而是通过调用js等方式从数据库中得到数据，回显到网页上。如果按照之前的办法： up=urllib2.urlopen(url) + cont=up.read() 就抓取不到上述内容了。由源码可以看出，HTML提供文字模板，js根据不同的id提供不同的变量，“填入”到文字模板中，形成了一个具体的网页。所以单纯抓取此HTML，只能得到一些文字模板，而无法得到具体内容。

[分析动态网页_2 ](http://chenqx.github.io/2014/12/23/Spider-Advanced-for-Dynamic-Website-Crawling/)

> 文中介绍静态bbs网页的抓取。但是，互联网大部分的web页面都是动态的，经常逛的网站例如京东、淘宝等，商品列表都是js，并有Ajax渲染，这样就获取不到网页内容（获取到后台数据后再组合成html展示出来的）。单纯获取页面而没有执行到js的话是无法看到商品数据列表信息的。

[分析动态网页_3 ](https://www.zhihu.com/question/21471960/answer/81061538)

> 然后，打开北邮人论坛的首页，发现它的首页HTML源码中确实没有页面所显示文章的内容，那么，很可能这是通过JS异步加载到页面的。请记住，对于一些前端渲染的网页，虽然在HTML源码中看不到我们需要的数据，但是更大的可能是它会通过另一个请求拿到纯数据（很大可能以JSON格式存在），我们不但不需要模拟浏览器，反而可以省去解析HTML的消耗。

[Web crawler with Python - 04.另一种抓取方式 ](http://xlzd.me/2015/12/19/python-crawler-04)

[读写JSON数据 ](http://python3-cookbook.readthedocs.org/zh_CN/latest/c06/p02_read-write_json_data.html)

[详解抓取网站，模拟登陆，抓取动态网页的原理和实现 ](http://www.crifan.com/files/doc/docbook/web_scrape_emulate_login/release/html/web_scrape_emulate_login.html)

