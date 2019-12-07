---
title: 获取唯一号之UUID
tags: uuid
categories: work
---

​	出现这篇文章，其实说明我当初写的唯一的询价单号出现了问题~我还记得写下那段代码的那天，天有点阴，那天的风格外的冷，我搓了搓手，思考了一会后，便开始啪啪啪的敲打起键盘来

<!--more-->

​	其实这里的询价单号，我都没想到会这么快就出现了问题，好吧，其实我根本就没想过~ 毕竟保险的项目的购买频率也不高，意味着也不会有很高的并发。

第一次脑子里是这么想的

```
 //这点并发量，用这个就够了吧
 //事实上很蠢，我自己都不相信，所以pass
 public String getQuotationNumber(){
    	return new Date().getTime()+"";
    }
```

摇了摇头，我觉得改装下估计就好了，于是就有了如下的代码

```
//在获取当前时间毫秒数+100内的随机数
//这还能重复？重复我就吃屎
public String getQuotationNumber(){
    	String number = new Date().getTime()+""+(int) (Math.random()*100);
    	return number;
    }
```

​	事实上，我真的是太年轻了，上面的代码就是最近出错的地方。而令我没想到的是这个错误并不是出现在生产上，而是出现在的冒烟测试中，好吧，也庆幸是在测试中出现的问题。

​	于是，我又摇了摇头，哦不，copy了一份别人的操作，看代码

```
 public String getQuotationNumber(){
    	int machineId = 1;//最大支持1-9个集群机器部署
        int hashCodeV = UUID.randomUUID().toString().hashCode();//hashcode会不会重复啊？各位大神？
        if(hashCodeV < 0) {//有可能是负数
            hashCodeV = - hashCodeV;
        }
        return machineId + String.format("%015d", hashCodeV);
        //这里的“%015d”指的是，不满十五位的用0填充的整数
    }
```

 UUID是指在一台机器上生成的数字，它保证对在同一时空中的所有机器都是唯一的。通常平台会提供生成的API。按照[开放软件基金会](https://baike.baidu.com/item/%E5%BC%80%E6%94%BE%E8%BD%AF%E4%BB%B6%E5%9F%BA%E9%87%91%E4%BC%9A)(OSF)制定的标准计算，用到了以太网卡地址、纳秒级时间、芯片ID码和许多可能的数字

UUID由以下几部分的组合：

（1）当前日期和时间，UUID的第一个部分与时间有关，如果你在生成一个UUID之后，过几秒又生成一个UUID，则第一个部分不同，其余相同。

（2）时钟序列。

（3）全局唯一的IEEE机器识别号，如果有网卡，从网卡MAC地址获得，没有网卡以其他方式获得。

UUID的唯一缺陷在于生成的结果串会比较长。关于UUID这个标准使用最普遍的是微软的GUID(Globals Unique Identifiers)。在ColdFusion中可以用CreateUUID()函数很简单地生成UUID，其格式为：xxxxxxxx-xxxx- xxxx-xxxxxxxxxxxxxxxx(8-4-4-16)，其中每个 x 是 0-9 或 a-f 范围内的一个十六进制的数字。而标准的UUID格式为：xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (8-4-4-4-12)，可以从cflib 下载CreateGUID() UDF进行转换。

总之，先这样吧~

这里的hashcode是否会重复的问题也想请问下各位老哥~