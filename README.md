# matchengine
内存撮合引擎，全CPU并发编程模型，证券外汇级商用撮合引擎，高并发、高可靠、故障转移、数据永不丢失、自动恢复，最高TPS： 8万/s


# 背景介绍
 

        区块链和比特币从只有行业极客谈论的话题，目前已经变成家喻户晓。比特币进入中国，衍生出很多种交易模式，有币币交易，场外交易，法币交易模式。

        特别是币币交易，每天买卖数几十亿级别以上，所以如何设计高性能电子化撮合引擎来满足当下的需求成了重要的话题。

        所以撮合交易在币币交易系统中扮演者非常重要的角色。了解撮合交易的本质以及业务对于设计撮合系统至关重要。接下来，我们就详细介绍下内存撮合引擎技术的设计思想。

 

# 什么是虚拟货币撮合交易?
简单的来讲撮合交易就是：

       拿身边的房产交易举例，张三想买房，李四、赵六想卖房，但是他们两个不认识也见不着，所以就出现了中介王五，这时候他们各自在王五这边告知买卖报价,在各自都能接受的报价内，相互成交。市场决定一切，张三想花钱买房，李四报价100万卖出，而赵六觉得现在房产行情不好，愿意95万就卖给张三，那么张三势必会找王赵六交易了。

币币交易撮合成交的前提是买入价必须大于或者等于卖出价。当买入价等于卖出价时，成交价就是买入价或者卖出价。当买入价大于卖出价时，计算机在撮合时实际上是根据前一笔成交价而定出最新成交价的。

      选取买入价、卖出价和前一成交价三者居中的一个价格作为最新成交价（如果前一笔成交价低于或等于卖出价，那么最新成交价就是卖出价；如果前一笔成交价高于或等于买入价，那么最新成交价就是买入价；如果前一笔成交价在卖出价与买入价之间，那么最新成交价就是前一笔的成交价）。

 

![](https://img-blog.csdnimg.cn/2020021919010648.PNG)

 

# 撮合引擎原理
 

## 撮合交易算法

       如图所示,撮合引擎的核心业务模块就是撮合交易算法。撮合交易算法的任务一方面是完成对客户所下订单进行公平合理的排列和撮合功能,也要保证撮合算法的公平性、高效性以及扩展性等。由于不同金融交易系统的撮合业务各有不同,因此教程对通用的撮合交易算法进行概括性描述。


![](https://img-blog.csdnimg.cn/20200219190259742.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)
 

 

## 订单队列

      撮合交易的重要组成部分就是买卖订单,通过对买卖订单进行撮合最后形成交易记录。所以对无法立刻完成撮合的订单,需要有买入队列和卖出队列保存订单。队列按照“价格优先、同价格下时间优先”的原则。买入队列按照委托价格从低到高的顺序,卖出队列则按照委托价格从低到高的顺序排列,如图

![](https://img-blog.csdnimg.cn/20200219190357155.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)

 

- 币币交易： 以币买卖币进行交易。如使用比特币来定价以太坊：交易对 ETH/BTC，表示购买一个ETH需要多少BTC。 买入：使用BTC购买ETH，把BTC兑换成ETH；卖出ETH，把ETH兑换成BTC。

- 法币交易(OTC)： 为了解决在线支付通道关闭的备选方案。客户A在交易所发布卖出（买入）价格。另一个客户B可以选择这个这个价格买入，然后A的币质押到交易所（中介），B可以按照A给出的银行卡，支付宝等直接C2C的转账。转成功后，后台点击确认，交易所把币设置到B方账户。

- 杠杆交易： 跟交易所借钱配资，放大本金。对撮合引擎不影响。

- 合约交易： 对未来币种合约到期的趋势判断。看涨开多单，看跌开空单。一般未来时间节点合约有当周(周五)、次周（下周五）、季（最后一个周五）等。可以进行杠杆。

- 限价单： 根据设定的委托单价格和数量进行交易

- 市价单： 设定固定额度(买入)或者数量（卖出）直接进行按照当前最优价格成交。

- 开盘价： 根据设定的委托单价格进行交易。

- 收盘价： 根据设定的委托单价格进行交易。

- 当前价： 最新的汇率价格或者说最新成交价。如BTC/USDT: 8500

 

 撮合顺序

    撮合引擎接收到新的买入订单,则会到卖出队列的头部查找是否存在符合价格规则的卖出订单,如果存在卖出价格小于或等于买入价格的订单,则从队列中取出此订单并撮合成一笔交易;如果卖出队列为空或队列头部不满足价格关系,则将买入订单插入买入队列中,由于买入队列是按照价格与时间先后进行排序,所以新插入的订单会经过一次排序插入到买入队列的相应位置。

相同的,当撮合引擎接收到新的卖出订单,则会到买入队列的头部査找是否存在符合价格规则的买入订单,如果存在买入价格大于或等于卖出价格的订单,则从订单队列中取出此订单并撮合成一笔交易;如果买入队列为空或队列头部不满足价格关系,则将卖出订单插入到卖出队列中,由于卖出队列也是按照价格与时间先后进行排序的所以新插入的订单会经过一次排序插入到卖出队列的相应位置[23]。结合买卖订单情况,撮合算法流程如图所示。从图所示的撮合顺序可知,买卖队列的有序性是保证撮合顺序的确定性的基础,并且撮合过程中每笔订单都可以撮合出当前最优交易。


![](https://img-blog.csdnimg.cn/20200219190635572.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)
 

 

 

内存撮合引擎设计
 撮合引擎的质量

 
![](https://img-blog.csdnimg.cn/20200219190853296.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)


 

初级版架构设计

![](https://img-blog.csdnimg.cn/20200219191121831.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)
 

中极版撮合引擎架构设计
  请微信联系哦

 ![](https://img-blog.csdnimg.cn/20200219191401197.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)

终极版撮合引擎架构设计
 请微信联系哦

![](https://img-blog.csdnimg.cn/20200219191401197.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)

性能跑分
2核8G 20000 -> 4核8G 30000 -> 8核16G 50000 -> 16核32G 80000 -> 32核64G 88000

![](https://img-blog.csdnimg.cn/20200219191720278.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1aHVhbG9uZzEzMTQ=,size_16,color_FFFFFF,t_70)

源码

![](https://img-blog.csdnimg.cn/20200219205646523.PNG)
