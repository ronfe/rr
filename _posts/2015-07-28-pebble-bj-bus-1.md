---
layout: post
title:  "【闲开发】Pebble北京公交实时提醒（1）"
date:  2015-07-28
categories: 养鸡场
featured_image: /images/jun28post.jpg  
---

### 任何东西，只要是玩具，都有玩腻的时候。  

最近从某宝处入了一块Pebble Time。本欲仅用作消息提醒，但信手翻了Pebble的开发者文档，觉得可以帮我做更多的事，比如公交实时提醒。

作为帝都打工仔如我，每晚下班后坐公交回家也算司空见惯之事。然而可恨常有“我等的车儿还不来”之慨叹，如能知晓离我最近的公交车在哪里，先不论是否可作为要不要等的决策，便是知情而等，感觉也是安稳的。幸而现在北京公交官网提供了部分线路的实时查询功能，内中正好有我下班的线路，故而兴起，有此作。

这次闲开发的主要目标是在用户（其实也就是自己用而已）设置自己所在的公交站、监控的公交线路及方向（可设置多条线路）后，告知用户最近的车还有多少站到达。另外如还有闲情，做个menu，显示用户设置的每条线路在其设置公交站之前的公交停站情况。

举例，如用户设置622路（北京站东-成和园小区）成和园小区方向，大栅栏站上车。假设离他最近的公交车下一站行至前门（大栅栏上站），则返回“剩余1站”，如果该车从前门站驶出，则显示“即将到达”。

需求就是这么简单。

这篇文章要解决的问题集中在“后端”，也就是从北京公交网站GET数据，并对其进行处理，以供前端使用的部分。

还是以622为例，打开它的[实时查询页面](http://www.bjbus.com/home/fun_rtbus.php?uSec=00000160&uSub=00000162&sBl=622&sBd=5327505317773260570&sBs=7 )。

通过devtools发现实时刷新是通过轮巡，定时向
```http://www.bjbus.com/home/ajax_search_bus_stop.php?act=busTime&selBLine=622&selBDir=5327505317773260570&selBStop=7```  这个URL发请求实现的。结合网页在选择公交页面时发送的其他请求，分析一下这个URL的query string：

* ```act=busTime``` //实时查询
* ```selBLine=622```  //公交线路，622
* ```selBDir=5327505317773260570```  //行驶方向，532……相当于一个方向的ID，这个值可以发```http://www.bjbus.com/home/ajax_search_bus_stop.php?act=getLineDirOption&selBLine=622``` 拿到，此处暂且不论，后文会用
* ```selBStop=7``` //查询站序号，始发站为1，依次递增，大栅栏为7，站序和站点信息可以通过```http://www.bjbus.com/home/ajax_search_bus_stop.php?act=getDirStationOption&selBLine=622&selBDir=5327505317773260570``` 拿，此处亦暂且不论

现在的问题是，这个链接不是API，不会通过XML或者JSON这样“规整”的格式返回，而是返回一组DOM。这样对于北京公交网站而言自然方便，直接把返回的DOM塞入文档中就好，但是对我这样“窃”数据的人来说（是的，读书人的事，怎么能叫偷呢！），又多了一步分析DOM的琐事。

如果发这个请求，那么一定会GET到一段又臭又长的DOM代码。

经过一番观察，我终于找到了这一大段DOM的要点：

* 在```id```为```cc_stop```的```div```下，有一个```ul```，里面存放的是线路的每一个站点及站点与站点之间的空隙
* 站点```div```的```id```就是其序号，站点和站点之间空隙的```div``` id则是下一站点的序号+m （例如第1站和第2站之间的空隙id为2m）
* 如果某个公交车在某个站点停靠，那么在那个站点的```div```中会多出一个```class```为```buss```的i标签
* 如果某个公交车在站点与站点之间的空隙上（在行驶中），那么在对应的某个id为```/\d+m/```的div子元素里有一个```class```为```busc```的标签

知道这些，数据就很简单了。因为解决这个问题的关键是拿到在设置站点之前所有车辆的停站信息（那些有busc或buss元素的父元素id），而这些恰巧就是我感兴趣停站信息。我只需要向这个URL发请求，在拿到这段DOM后的回调里匹配一下，就能拿到。代码：

    var URL = 'http://www.bjbus.com/home/ajax_search_bus_stop.php?act=busTime&selBLine=622&selBDir=5327505317773260570&selBStop=7';
    var regex = /\d+m?(?=\\"><i +class=\\"bus[cs]\\")/g;
    ajax({
              url: URL,
              type: 'GET'
          }, function(data){
              var b = data;
              var resArray = b.match(regex);
           }, function(err){
            console.error(err);
          }
    );

```regex```这个正则就是要取```class```为```busc```或```buss```的i元素中，其父元素的```id```。在Ajax请求成功的回调里，```b```存储拿到的DOM，而```resArray```匹配的正是有公交车的那些站点id。

这些id中，自然既有可能是站点序号（公交停靠在该站点），也有可能是站点序号+m（公交驶向该站点）。为了后续数据处理方便（特别是计算站差），可以把所有带m的站点全部转换成数字再减掉0.5（例如"2m"换为1.5），所有不带m的站点全部转换为数字，代码（在```resArray```拿到数据后）：

    var outArray = [];
    if (resArray.length >= 1){
        for (var i = 0; i < resArray.length; i++){
            var tempArr = resArray[i].split('m');
            if (tempArr.length === 1){
                outArray.push(Number(tempArr[0]));
            }
            else {
                outArray.push(Number(tempArr[0]) - 0.5);
            }
        }
        
这样```outArray```中存储的就是已经结构化的数据了。

为了实现实时更新，我也来个轮巡。因为公交数据更新比较慢，所以```interval```可以设置略长些，这里把上述代码封装为```sendRequest```函数后，设置10000ms拿停站信息:

    setInterval(sendRequest, 10000);
    
今天就到这儿，下次假设用户设定为常数，做前端吧！

弘飞  7月25日，禺中，于自宅

http://ronfe.io
