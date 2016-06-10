---
layout:     post
title:      "IB账户信息介绍"
subtitle:   " \"仔细介绍account window里的信息\""
date:       2016-06-10 12:00:00
author:     "gambol"
header-img: "img/post-bg-2016.jpg"
tags:
    - IB
    - 经济
--- 

## 起因

ib的个人账户信息实在太复杂了, 一直都迷迷糊糊的. 这次认真学习一下

## 看图说话

这是我的账户截图
![账户图](http://static.zybuluo.com/gambol/cbzln0w2zds2wsf8jx0ounpc/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-10%20%E4%B8%8B%E5%8D%889.35.55.png)

具体解释请看下表

| 行 |  中文翻译 |  计算公式 |  注释|
| ---- | ---- | -----|  ---|
| Net liquidation Value| 净清算值 | 总现金值+股票值+证券期权值+债 券值+基金值 | 账户的总价值 . 等于你的证券总价值 + 现金额|
|Equity with Load Value| 含借贷值股权 | 现金账户: 结算的现金。 保证金账户: 全部现金值+股票值+ 债券值+基金值+欧洲&亚洲期权 值。 | 一直以为, 这个值和 net liquidation value是相同的|
|Current initial Margin| 当前初始保证金 | 所有股票的初始保证金之和 |  |
|Current Maintenance Margin| 当前维持保证金 | 维持这个比例, 你所需要的最低保证金数量 | |
| Current Available Funds| 当前可用金额 | 含借贷值股权-初始保证金 | 这个值,就是你可以提现的值 |
| Current Excess Liquidity | 当前剩余流动性|  含贷款值股权-维持保证金 | 剩余流动性是整个表里最重要的指标. 剩余流动性小于0时, IB会平仓你的部分头寸. 买入股票,或者股价下跌, 剩余流动性都会减少|
| Buying power | 购买力 | 现金账户: 净清算值 -初始保证金.  标准保证金账户:(净清算值 -初始保证金) * (杠杆倍数, 通常为4) | 这个值告诉你,你还有多少钱可以操作 |

## 进一步说明结算公式

1. 保证金要求. 保证金要求 = 证券总价值 * 0.25.  但是对于reg t margin账户来说, 有一部分股票要求的是100%的保证金, 譬如ctrp. 这些股票的列表请看[列表](https://www.ibkr.com.cn/en/index.php?key=&cntry=usa&tag=United+States&ib_entity=hk&ln=&f=5168&ns=T)
2. 触发自动平仓的计算公式.  股票的亏损值 = （总资产-保证金要求）/（1-保证金比例）

## 举例说明

假设, 自有资金10000美金，买入baba200股，股价100美金，初始保证金比例25%。（启动2倍杠杆）

>
1. 证券总价值：100*200=20000美金
2. 现金额=-10000美金
3. 总资产=证券总价值+现金额=20000-10000 =10000美金
4. 保证金要求=20000*25%=5000
5. 购买力=（10000-5000）*4=20000
6. 日内风控值=总资产-保证金要求=10000-5000=5000
7. 触发平仓的亏损额=总资产-保证金要求/1-25% =10000-5000/1-25%=6666
8. 6666美金/200股=33.33，则股价下跌33.33（股价为66.67）时被强制平仓
>

## 参考文献
1. [IB官方说明](https://www.interactivebrokers.com/en/software/tws/usersguidebook/realtimeactivitymonitoring/available_for_trading.htm#XREF_98990_Available_for)
2. [sina博客](http://blog.sina.com.cn/s/blog_48fa17ad0102xjtq.html)
3. [IB账户介绍--from 雪球](https://xueqiu.com/4586203773/30860827)
