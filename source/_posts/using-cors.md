---
title: 使用CORS进行跨域请求
date: 2016-02-11 16:05:55
tags: [CORS, HTTP]
categories: 翻译
---
## 引言
　　API的出现增强了富web应用开发的体验。曾经想通过跨域请求api非常困难可以使用json-p这样的技术（有安全限制）或者自己搭建搭服务器。
　　
　　跨域资源共享（CORS）是一种W3C的规范，允许浏览器进行跨域通信。通过设置XMLHttpRequest对象，CORS允许开发者像发起同域请求那样发起跨域请求。

　　想象下，网站bob.com想要访问alice.com网站上的一些数据。这种请求是被同源策略所禁止的。然而，使用CORS，alice.com 增加些特殊响应头可以允许来自bob.com的请求。

　　从上面例子看出，使用CORS需要服务器和客户端之间相互了解。幸运的是，如果你是一位客户端开发者，你可以忽略里面的很多细节。本文将介绍客户端如何进行跨域请求，以及如何配置服务器以支持
CORS。

## 使用CORS请求
　　本节将介绍如何用Javascript进行跨域请求。
### 创建XMLHttpRequest对象
　　浏览器支持情况：
* Chrome 3+
* Firefox 3.5+
* Opera 12+
* Safari 4+
* Internet Explorer 8+

（详细列表请访问[http://caniuse.com/#search=cors](http://caniuse.com/#search=cors)）

　　Chrome浏览器，火狐，Opera和Safari都使用XMLHttpRequest对象。Internet Explorer使用
XDomainRequest对象，有和XMLHttpRequest对象差不多的功能，但是增加了额外的安全防范措施。

　　开始之前，你需要创建相应的请求对象。Nicholas Zakas 写了个[辅助方法](https://www.nczonline.net/blog/2010/05/25/cross-domain-ajax-with-cross-origin-resource-sharing/)来兼容不同浏览器：<br>
<pre><code>
function createCORSRequest(method, url) {
  var xhr = new XMLHttpRequest();
  if ("withCredentials" in xhr) {

    // Check if the XMLHttpRequest object has a "withCredentials" property.
    // "withCredentials" only exists on XMLHTTPRequest2 objects.
    xhr.open(method, url, true);

  } else if (typeof XDomainRequest != "undefined") {

    // Otherwise, check if XDomainRequest.
    // XDomainRequest only exists in IE, and is IE's way of making CORS requests.
    xhr = new XDomainRequest();
    xhr.open(method, url);

  } else {

    // Otherwise, CORS is not supported by the browser.
    xhr = null;

  }
  return xhr;
}

var xhr = createCORSRequest('GET', url);
if (!xhr) {
  throw new Error('CORS not supported');
}
</code></pre>


### Event handlers
　　原始的XMLHttpRequest对象只有一个event handler:onreadystatechange,它处理所有响应。
XMLHttpRequest2包含许多新的event handler，onreadystatechange依然可以使用。下面是
完整列表：


| Event Handler  | 描述           |
| -------------  |-------------  |
| onloadstart*   | 当请求发生时触发 |
| onprogress     | 读取及发送数据时触发     |
| onabort*       | 当请求被中止时触发，如使用abort()方法    |
| onerror        | 当请求失败时触发     |
| onload         | 当请求成功时触发     |
| ontimeout      | 	当调用者设定的超时时间已过而仍未成功时触发     |
| onloadend*     | 请求结束时触发（无论成功与否）      |

> IE的XDomainRequest不支持带 * 的项目，数据来自[http://www.w3.org/TR/XMLHttpRequest2/#events](http://www.w3.org/TR/XMLHttpRequest2/#events)


　　大多数情况下，我们只需要处理onload和OnError事件：
<pre><code>
xhr.onload = function() {
 var responseText = xhr.responseText;
 console.log(responseText);
 // process the response.
};

xhr.onerror = function() {
  console.log('There was an error!');
};
</code></pre>

　　请求发生错误时，浏览器不能提供有效的错误信息。例如，FireFox在失败时将responseText置空并返回一个0值作为状态码，这当中并不包含任何错误的具体情况。尽管浏览器把错误信息输出到控制台，但是这些信息无法用JavaScript获得到。当触发onerror事件时，也仅仅知道发生错误了而已。
### withCredentials
　　标准的CORS请求不操作Cookie，既不发送也不设置。为了使请求中包含Cookie，就需要将withCredentials设置为true。
<pre><code>
xhr.withCredentials = true;
</code></pre>

　　服务器在处理这一请求时，需要将Access-Control-Allow-Credentials设置为true，后面会有详细介绍。
　　
> Access-Control-Allow-Credentials: true

　　withCredentials属性使得请求包含了远程域的所有Cookie，但是值得注意的是这些Cookie仍然遵守同源策略，所以在代码中并不能通过"document.cookie"或者响应头来获得cookie信息。这些cookie仅由远程域控制。
### 发送请求
　　现在CORS Request 已经配置好，可以调用send()方法来发送请求。
>xhr.send();


　　如果请求需要包含其他内容，只需将其作为send()方法的一个参数。

　　一旦服务端配置成功，那么我们只需处理后续的onload事件，就像我们平时所熟悉的XHR一样。
### 完整例子
　　下面是CORS request的完整代码。[运行](http://www.html5rocks.com/en/tutorials/cors/#)该示例并通过浏览器调试器来观察实际的网络请求。

```
// Create the XHR object.
function createCORSRequest(method, url) {
  var xhr = new XMLHttpRequest();
  if ("withCredentials" in xhr) {
    // XHR for Chrome/Firefox/Opera/Safari.
    xhr.open(method, url, true);
  } else if (typeof XDomainRequest != "undefined") {
    // XDomainRequest for IE.
    xhr = new XDomainRequest();
    xhr.open(method, url);
  } else {
    // CORS not supported.
    xhr = null;
  }
  return xhr;
}

// Helper method to parse the title tag from the response.
function getTitle(text) {
  return text.match('<title>(.*)?</title>')[1];
}

// Make the actual CORS request.
function makeCorsRequest() {
  // All HTML5 Rocks properties support CORS.
  var url = 'http://updates.html5rocks.com';

  var xhr = createCORSRequest('GET', url);
  if (!xhr) {
    alert('CORS not supported');
    return;
  }

  // Response handlers.
  xhr.onload = function() {
    var text = xhr.responseText;
    var title = getTitle(text);
    alert('Response from CORS request to ' + url + ': ' + title);
  };

  xhr.onerror = function() {
    alert('Woops, there was an error making the request.');
  };

  xhr.send();
}
```
## 服务器配置
　　我们需要知道到服务端到底从浏览器那里收到了怎样的内荣。为了完成CORS请求，浏览器会增加一些额外的请求头，有时还会发送额外的请求。（可以通过网络封包分析软件来查看，比如 [Wireshark](http://www.wireshark.org/)）。

　　因为浏览器已经负责实现了CORS最关键的部分；但是服务端的后台脚本则需要我们自己进行处理。这节介绍如何配置服务器来支持CORS。
![](/images/20160227021223.png)
CORS流程图

### CORS请求分类
　　Cross-orgin request 分为两类
1. 简单请求
2. 复杂请求

　　简单请求有下列特征：
* HTTP 方法匹配下面任何一个（区分大小写）：
  * HEAD
  * GET
  * POST
* 包含这些HTTP Header（区分大小写）：
  * Accept
  * Accept-Language
  * Content-Language
  * Last-Event-ID
  * Content-Type，值仅能是下列之一：
    * application/x-www-form-urlencoded
    * multipart/form-data
    * text/plain

　　简单请求的特征就是这样，可以由浏览器完成请求无需CORS。比如，JSON-P可以发出跨域GET请求。Or HTML could be used to do a form POST.

　　任何一个不满足上述条件的请求都是复杂请求，复杂请求需要额外的预请求（preflight request），下面我们将介绍。
### 简单请求

　　我们先看一个简单请求。下面是JavaScript代码，紧随其后的是浏览器发出的请求，粗体字表示CORS请求所需的header。
JavaScript:
<pre><code>
var url = 'http://api.alice.com/cors';
var xhr = createCORSRequest('GET', url);
xhr.send();
</code></pre>
HTTP请求:
<pre><code>
GET /cors HTTP/1.1
<b>Origin</b>: http://api.bob.com
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
</code></pre>

　　CORS请求总是包含一个**Origin**请求头，它由浏览器添加。该值包含协议名、地址和一个可选的端口。例如：http://api.alice.com.
　　有**Origin**请求头的请求并不一定是跨域请求。不仅跨域请求需要Origin请求头，一些同域请求也可能需要有该请求头。例如，Firefox在同域请求中不添加Origin请求头，但是Chrome和Safari在POST/PUT/DELETE同域请求中包含Origin请求头。下面是个栗子。
　　HTTP请求
<pre><code>
POST /cors HTTP/1.1
<b>Origin</b> : http://api.bob.com
<b>Host</b>: api.bob.com
</code></pre>

　　无论请求中是否有CORS所需的请求头，服务器都会响应。但是如果Origin中的域名不在服务器的允许列表中，服务器将返回错误信息。
　　下面是有效的响应，CORS所需headers用粗体表示。
　　HTTP 响应：
<pre><code>
<b>Access-Control-Allow-Origin</b>: http://api.bob.com
<b>Access-Control-Allow-Credentials</b>: true
<b>Access-Control-Expose-Headers</b>: FooBar
Content-Type: text/html; charset=utf-8

</code></pre>

　　在响应中，和CORS相关项目都是以"Access-Control-"为前缀，具体含义如下：
 + Access-Control-Allow-Origin (必含)-不可省略，否则CORS请求失败。该项表示服务器响应指定的域的CORS请求。值为 “\*”时响应任何域的CORS请求。

 + Access-Control-Allow-Credentials（可选）该项表示请求中是否包含Cookie信息，默认情况，请求中不包含Cookie信息。只有一个可选值true（必须小写）。如果不包含Cookie信息，请略去该项而不是设置其值为false。这一项与XmlHttpRequest2对象当中的withCredentials属性应保持一致，即withCredentials为true时该项也为true；withCredentials为false时，省略该项不写。反之则导致请求失败。除非你确实需要CORS请求中包含Cookie信息，否则不要设置该项。

 * Access-Control-Expose-Headers（可选）- XMLHttpRequest2 对象的getResponseHeader()方法返回相应头的信息。在CORS请求中，该方法只能获得一些简单信息。能获得的信息如下：

    * Cache-Control
    * Content-Language
    * Content-Type
    * Expires
    * Last-Modified
    * Pragma

　　如果你希望客户端能获得到其他header信息，你必须设置Access-Control-Expose-Headers。其值为其他相应头名字，多个值之间用逗号隔开。


### 复杂请求

　　到目前我们只介绍了简单请求，但是如果我们想做更多的操作怎么办？比如你需要发送PUT、DELTET等HTTP动作，或者发送Content-Type:application/json的内容。这时就需要我所说的复杂请求了。
　　复杂请求表面上看起来和简单请求使用上差不多，但实际上浏览器发送了不止一个请求。其中最先发送的是一种“预请求”，此时作为服务端，也需要返回“预回应”作为响应。预请求实际上是对服务端的一种权限请求，只有当预请求成功返回，实际请求才开始执行。

JavaScript：
<pre><code>
var url = 'http://api.alice.com/cors';
var xhr = createCORSRequest('PUT', url);
xhr.setRequestHeader(
    'X-Custom-Header', 'value');
xhr.send();
</code></pre>

预请求：
<pre><code>
OPTIONS /cors HTTP/1.1
<b>Origin</b>: http://api.bob.com
<b>Access-Control-Request-Method</b>: PUT
<b>Access-Control-Request-Headers</b>: X-Custom-Header
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
</code></pre>

　　类似简单请求，预请求也包含CORS需要的请求头。预请求用HTTP OPTIONS方法发送请求（确保你的服务器响应该方法的请求）。它也包含一些额外的请求头：
* Access-Control-Request-Method - 实际的HTTP请求方法，即使简单请求也会有该请求头。

* Access-Control-Request-Headers - 以逗号分隔的列表，请求所使用的header。

　　预请求用来验证服务器是否允许执行实际的请求。服务器校验上面两个请求头，验证请求方法和请求头是有效和可处理的。如果允许则服务器应返回如下响应：

预请求：
<pre><code>
OPTIONS /cors HTTP/1.1
<b>Origin</b>: http://api.bob.com
<b>Access-Control-Request-Method</b>: PUT
<b>Access-Control-Request-Headers</b>: X-Custom-Header
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
</code></pre>

预响应
<pre><code>
<b>Access-Control-Allow-Origin</b>: http://api.bob.com
<b>Access-Control-Allow-Methods</b>: GET, POST, PUT
<b>Access-Control-Allow-Headers</b>: X-Custom-Header
Content-Type: text/html; charset=utf-8
</code></pre>

Access-Control-Allow-Origin（必含） - 类似简单请求的响应，预响应也应该包含该响应头。

Access-Control-Allow-Methods（必含) - 以逗号分隔的列表，服务器支持的请求方法。

Access-Control-Allow-Headers（如果请求有Access-Control-Request-Headers请求头，该项为必含）- 以逗号分隔的列表，服务器支持的请求头。类似 上面的 Access-Control-Allow-Methods ，列出了服务器支持的所有的header，而不仅仅是预请求中的 Access-Control-Request-Headers。

Access-Control-Allow-Credentials（可选） - 和简单请求定义一样。

Access-Control-Max-Age (可选) - 预检对服务器带来很大压力，因为客户端每次操作都执行两次请求。preflight response 可以缓存起来，来减轻服务器压力，该项设置了缓存失效时间。

一旦预检成功，浏览器将发起实际请求，看起来和简单请求一样。响应也类似：

实际请求:
<pre><code>
PUT /cors HTTP/1.1
<b>Origin</b>: http://api.bob.com
Host: api.alice.com
<b>X-Custom-Header</b>: value
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
</code></pre>

实际响应:
<pre><code>
<b>Access-Control-Allow-Origin</b>: http://api.bob.com
Content-Type: text/html; charset=utf-8
</code></pre>

使服务器拒绝CORS请求，可以仅仅返回一个不包含任何CORS header的响应。预检失败服务器也应拒绝该次请求。由于响应里没有CORS特定的header，浏览器认为请求失败，不会发起实际请求:

预检请求：
<pre><code>
OPTIONS< /cors HTTP/1.1
<b>Origin</b>: http://api.bob.com
<b>Access-Control-Request-Method</b>: PUT
<b>Access-Control-Request-Headers</b>: X-Custom-Header
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
</code></pre>

预检请求响应
```
// ERROR - 没有 CORS header, 这是一个无效请求!
Content-Type: text/html; charset=utf-8
```

如果CORS请求发生错误，浏览器将触发onerror事件，并且会把错误信息输出到控制台：
> XMLHttpRequest cannot load http://api.alice.com. Origin http://api.bob.com is not allowed by Access-Control-Allow-Origin.


浏览器不会给出详细的错误情况，仅仅告知我们请求出错了。

### 安全问题

跨域请求始终是网页安全中一个比较头疼的问题，CORS提供了一种跨域请求方案，但没有为安全访问提供足够的保障机制，如果你需要信息的绝对安全，不要依赖CORS当中的权限制度，应当使用更多其它的措施来保障，比如OAuth2。

---
>编译自[http://www.html5rocks.com/en/tutorials/cors/](http://www.html5rocks.com/en/tutorials/cors/)    <br>作者：Monsur Hossain
本文中的所有译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接，本文翻译工作遵照 [CC协议](http://zh.wikipedia.org/wiki/Wikipedia:CC)，如果有侵犯到您的权益，请及时联系我。
