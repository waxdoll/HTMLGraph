# HTMLGraph

<p style="font-size: 36px; text-align: center;"><a href="https://cumtzzp.github.io/pages/htmlgraph/" target="_blank">demo</a></p>

以前介绍过很多次的一个分析 HTML 页面 DOM 树并生成非常漂亮元素连接图的应用 [Websites as Graphs](http://www.aharef.info/static/htmlgraph)，很可惜，现在这个网站已经无法访问了。本页面基于 [jQuery](http://jquery.com/), [Protovis](http://mbostock.github.io/protovis/) 和 [YQL](https://developer.yahoo.com/yql/) 重现了该应用。

[Protovis](http://mbostock.github.io/protovis/) 是一个不再维护的项目，大名鼎鼎的 [d3.js](http://d3js.org/) 是它的延续项目，因为之前恰好看见 [Protovis](http://mbostock.github.io/protovis/) 的一个 [Force-Directed Layouts](http://mbostock.github.io/protovis/ex/force.html) 示例，和我要实现的功能非常像，所以也就没去研究更新的 [d3.js](http://d3js.org/)。我需要做的非常简单，生成类似下面的数据交给 [Protovis](http://mbostock.github.io/protovis/) 生成力导向布局图即可：

<pre>
{
    nodes: [
        { nodeName: "html", group: 1 },
        { nodeName: "head", group: 2 },
        { nodeName: "body", group: 2 },
    ],
    links:[
        { source: 1, target: 0, value: 1 },
        { source: 2, target: 0, value: 1 }
    ]
}
</pre>

其中，nodes 中的对象定义了力导向布局图中要显示的节点，也可以自行为对象增加属性满足自己的需要；而 links 定义了这些节点之间如何连接，从来源（source）连接到目标（target），来源和目标的属性值均为上述 nodes 中定义的节点对象的索引。如果 links 数组中的数据存在问题，会出现“Cannot read property 'linkDegree' of undefined”的错误。

力导向图是个图，而 HTML 文档对应的树要更简单一些。我们要做的是接受用户输入的一个 URL，获取该 URL 返回的 HTML 内容，遍历该 HTML 内容中的所有元素，将每个元素作为一个节点添加到 nodes 集合中，并将每个元素的所有子元素到该元素的连接表示出来添加到 links 集合中。

为了便于部署，我一开始就希望完全脱离服务器端而只是用浏览器端的方法完成所有流程，问题来了，由于跨域（cross domain）的原因，我们无法在浏览器的安全级别下使用 javascript 获取页面的 HTML 代码（服务器端解决这个问题就很容易，[Websites as Graphs](http://www.aharef.info/static/htmlgraph) 使用 Java Applet 也轻松解决了这个问题，但浏览器端因需要安装 JRE 变得不易部署）。解决方案是使用 [YQL](https://developer.yahoo.com/yql/)， [YQL](https://developer.yahoo.com/yql/) 可以使用类似 SQL 的语法查询网页中的内容并以 JSON 或 XML 等格式返回。例如，我们要获取 http://sm.cumt.edu.cn 首页的 HTML 源文件，其 [YQL](https://developer.yahoo.com/yql/) 的写法为：

select * from html where url = 'http://sm.cumt.edu.cn'

把该语句附加到 yahooapis 的地址后形成类似 http://query.yahooapis.com/v1/public/yql?q=select%20*%20from%20html%20where%20url=%22http://sm.cumt.edu.cn%22 的形式返回的结果就是 XML。这就提供给我们在浏览器端使用 jQuery 请求用户指定 URL 对应的 HTML 片段的能力。需要指出的是，由于 [YQL 本身的一些限制](https://developer.yahoo.com/yql/provider/)（如对方网站 robots.txt 的设置），有些网站无法返回数据。

如果你使用上述链接查看返回的 XML 中的 HTML 片段，会发现只有 body 部分，而忽略了 head 部分的 HTML 代码。这时，需要修改 YQL 语句：

select * from html where url = 'http://sm.cumt.edu.cn' and xpath = '/html'

调用该语句的 jQuery 代码大概是：

<pre>
$.getJSON(
    "http://query.yahooapis.com/v1/public/yql?q=" + 
    "select%20*%20from%20html%20where%20" +
    "url=%22" + encodeURIComponent($("#pageurl").val()) +
    "%22%20and%20" + 
    "xpath=%22/html" +
    "%22&format=xml'&callback=?",
    function(data){
        if(data.results[0]){
            var rmtDoc = $($.parseXML(data.results[0]));
            getChildren(rmtDoc, 0);
            //使用 Protovis 渲染力导向布局图
        } 
        else {
            alert("错误：无法加载目标页面地址指定的页面内容！");
        }
    });
});
</pre>

其中，#pageurl 是接受用户输入页面地址的文本框，而 getChildren 是用于递归生成 [Protovis](http://mbostock.github.io/protovis/) 需要的数据的：

<pre>
var dataobj = { nodes:[], links:[] };
var ordinal = depth = 0;

function getChildren(nd, id) {
    if (nd.children().length == 0) {
        return;
    }
    else {
        var children = nd.children();
        for (var i = 0; i < children.length; i++) {
            ordinal++;
            var docd = $(children[i]).parents().length;
            depth = Math.max(depth, docd); 
            dataobj.nodes[dataobj.nodes.length] = {nodeName: children[i].tagName, group: docd, ordinal: ordinal};
            if (ordinal > 1) {
              dataobj.links[dataobj.links.length] = {source: ordinal - 1, target: id, value: 1};
            }
            getChildren($(children[i]), ordinal - 1);
        }
    }
}
</pre>

获得数据 dataobj 后，使用 Protovis 渲染力导向布局图就简单了。可以在 force.node.add(pv.Dot) 时修改代码指定图中节点的大小（size）及颜色（fillStyle）等。按照 [Websites as Graphs](http://www.aharef.info/static/htmlgraph) 原作者的定义，我也使用了基本相同的颜色图例：

* <span style="color: #000000;">黑</span>: HTML 标签，根节点 
* <span style="color: #FF0000;">红</span>: 表格相关，如 TABLE, TR, TD 等
* <span style="color: #EFA500;">橙</span>: 换行或区块引用，如 BR, P, BLOCKQUOTE 等 
* <span style="color: #FFFF00;">黄</span>: 表单相关，FORM, INPUT, TEXTAREA, OPTION 等 
* <span style="color: #008000;">绿</span>: DIV 标签
* <span style="color: #00FFFF;">青</span>: HTML 5 中的新标签
* <span style="color: #0000FF;">蓝</span>: 超级链接，A 标签
* <span style="color: #EE82EE;">紫</span>: 图片，IMG 标签
* <span style="color: #808080;">灰</span>: 所有其它标签

当然，青色的标签是 [Websites as Graphs](http://www.aharef.info/static/htmlgraph) 原来没有而此处新增的。

<figure>
    <img src="https://cumtzzp.github.io/images/snap_htmlgraph.png" alt="HTML Graph">
</figure>

细节大概就是这样，一个玩具而已，请到 [https://cumtzzp.github.io/pages/htmlgraph/](https://cumtzzp.github.io/pages/htmlgraph/) 试用。时间仓促，没有特别仔细了解 [Protovis](http://mbostock.github.io/protovis/) 的细节，所以难免会出现一些问题，请在页面 [http://zzp.lol/HTML-Graph/](http://zzp.lol/HTML-Graph/) 反馈，谢谢！
