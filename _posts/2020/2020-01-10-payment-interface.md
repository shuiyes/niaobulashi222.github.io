---
layout: post
title: 网上支付平台接口使用总结
category: java
tags: [java]
copyright: java

---

2019年年底，也就是12月31号那天，西安这边的项目组工作已经结束，项目组人员调回武汉，时隔两年，终于回武汉了！
这次回武汉，进入一个与政府有关的项目，主要做**统一支付平台**。
主要是归纳一下支付有关的交易工作使用总结。

如果有感兴趣的同学，建议搜查银行官网的支付对接的文档，比如银联、农行等等都有提供支付接口文档。这里所说的支付场景并不是我们日常使用的二维码支付，那又是另外一种方式。这里所讲的就是一般的支付场景：下订单，选择支付方式，支付请求，支付通知。

## 总体架构

针对对象有三：消费者浏览器、商户交易网站服务器、银行网上支付平台（以农行为例）

![总体架构图](https://images.niaobulashi.com/typecho/uploads/2020/01/3949648068.png)

我们相当于交易网站服务器，提供支付请求和支付结果通知的转达，记账，对账，清算等处理。

## 交易流程

#### 支付交易

![支付交易图](https://images.niaobulashi.com/typecho/uploads/2020/01/2439822400.png)

支付交易因为需要三方的配合（消费者、商户交易网站、网上支付平台），且交易流程是分两阶段进行，所以商户交易平台需要开发两个主要的程序才能完成整个支付的流程，此两支程序为“**支付请求程序**”及“**支付结果接收程序**”。

交易的过程根据支付结果的接收方式的不同而不同，两种交易流程分别如下图所述：
**页面通知**支付结果方式：

![页面通知的支付流程图](https://images.niaobulashi.com/typecho/uploads/2020/01/2911269413.png)

**服务器通知**支付结果方式：

![服务器通知的支付流程图](https://images.niaobulashi.com/typecho/uploads/2020/01/3470969017.png)

#### 确保支付结果正确送达商户网站的措施

网上支付平台为了防止网络异常中断所造成的支付结果丢失，建议商户实现下列网上支付平台所提供的机制。

- 交易查询
  针对未收到银行交易结果回复的订单，或银行响应交易状态未明的订单，商户可以在任何时刻主动发起交易查询请求（详细交易说明请参考 5.7 交易查询），查询订单（支付）的状态。例如在商户网站提供消费者支付结果查询的功能，如该订单未收到网上支付平台交易结果，则调用网上支付平台的交易查询交易取得交易结果（订单状态），然后以取得的交易结果更新商户网站的支付状态。

![交易查询](https://images.niaobulashi.com/typecho/uploads/2020/01/1495867818.png)

- 通知商户支付成功

  支付成功后，如果消费者浏览器安装了某些拦截弹出窗口软件（例如 3721），那么支付结果接收页面有可能不会正常弹出，此时消费者可以点击【通知商户支付成功】按钮，重新发送支付结果到商户交易平台，确保商户能够收到网上支付平台的交易结果通知。

#### 其它交易

其它的交易（单笔退款、交易查询、对账单查询）只需要商户及网上支付平台的参与，交易的过程是实时响应，商户只需要简单的开发交易程序即可完成交易的过程。交易过程如下图所述：

![其它交易](https://images.niaobulashi.com/typecho/uploads/2020/01/183663467.png)

## 交易使用时机

- 支付请求交易
  消费者在商户网站上购买商品，并选择网上支付时。
- 支付结果接收
  消费者在网上支付平台上进行在线支付的操作，支付成功后，网上支付平台会将支付的结果通知到商户指定的支付结果通知页面。商户必须开发此页面，否则无法收到支付结果的
  通知。
-  退款
  针对已经结帐的订单，商户可以使用单笔退款或批量退款交易来退还交易金额给消费者。退款的交易由商户自行发起，不需要消费者的参与。
- 交易查询
  针对未收到银行交易结果回复的订单，或银行响应交易状态未明的订单，商户可以发起订单查询请求，查询订单的状态。网上支付平台的支付结果页面也会提供消费者通知商户支付成功的链接按钮，用来确定商户是否已经收到网上支付平台的通知。商户必须开发此页面。
- 交易流水查询
  商户可以指定时间段批量查询交易状态。
-  对账单查询
  网上支付平台每日根据联机交易后台返回的会计日期来生成对账单。商户可下载前一日的交易对账单，确定是否有未回传的成功交易。
- 网上 K  码支付
  不需要跳转页面即能实现网上支付。包括网上 K 码支付账单发送、网上 K 码支付支付请求和网上 K 码支付验证码重发。
- 授权支付
  客户、商户和银行三方签约后，银行可以代替商户对客户进行扣款。包括授权支付签约、授权支付签约结果查询、授权支付解约、单笔授权扣款、批量授权扣款和批量授权扣款结果查询。
- 身份验证
  验证客户证件类型、证件号码和卡号是否与本人户名相匹配，包括需要页面跳转的身份验证和非页面跳转的身份验证。
- 预授权确认/ 取消
  支付请求中的支付类型选择“预授权支付”时，预授权确认交易进行扣款，预授权取消交易取消预授权。

## 两种接收支付结果方式的区别

消费者在网上支付平台上进行在线支付的操作，支付成功后，网上支付平台会将支付结果通知给商户，目前通知方式有两种： **通过显示给消费者的支付结果接收页面通知商户**和**通过支付平台服务器通知商户**

#### 通过显示给消费者的支付结果接收页面通知商户

商户选择此种接收支付结果通知的方式，需要开发一个接收支付结果通知的页面。

商户在向网上支付平台发送交易请求的时候选择通过 页面通知方式接收支付结果，传送给支付平台一个支付结果通知的页面地址；然后消费者在网上进行在线支付，如果支付成功后，网上支付平台会将支付结果信息通过显示给消费者的支付结果通知页面通知给商户。
交易流程如下：

![页面通知的支付流程图](https://images.niaobulashi.com/typecho/uploads/2020/01/2911269413.png)

#### 通过支付平台服务器通知商户

商户选择此种接收支付结果通知的方式，需要开发两个页面：

- 接收服务器通知的页面 ServerURL（ ReceiveServerPage.jsp）。
- 展示给消费者支付结果信息的页面 CustomerURL（ResultSuccess.jsp 和 ResultFail.jsp）。

注意：**这两个页面的 URL  应该是在公网能访问的地址，而且接收服务器通知的页面 仅能 以
http  方式访问，不能用 https  访问。**

![服务器通知的支付流程图](https://images.niaobulashi.com/typecho/uploads/2020/01/3470969017.png)

商户在向网上支付平台发送交易请求的时候选择通过 服务器通知的方式接收支付结果，传送给支付平台一个接收服务器通知的页面（ServerURL），此页面的 HTML 代码里应该包含一个准备展示给消费者支付结果的 URL 链接（CustomerURL） （注意：链接之间需要用<URL></URL>包含，具体代码参见程序范例中的ReceiveServerPage.jsp ），然后消费者在网上进行在线支付，如果支付成功后，网上支付平台会将支付结果通知给商户，商户接收到支付结果信息后，必需将显示给消费者的页面 URL（CustomerURL）链接返回给支付平台服务器，然后支付平台服务器把接收到的这个展示给消费者支付结果信息的页面弹出给消费者显示。

如果第一次向商户发送通知时发生下列情况时：

-  1、无法连接到指定的商户交易结果接收页面；
- 2、商户交易结果接收页面没有正确响应消费者支付结果 URL。

系统将会在消费者的浏览器弹出一个新的窗口，并以此新窗口打开商户支付结果接收页面（ServerURL）。为了保证在此状况下消费者还是可以看到正常的商户交易结果页面（CustomerURL），建议在 ServerURL 页面加上自动转向 CustomerURL 的脚本，此脚本范例请参考程序范例中的 ReceiveServerPage.jsp 页面。

#### 区别

采取通过**页面通知**的方式将支付结果通知给商户，如果消费者的浏览器里安装了一些弹出窗口拦截软件（例如：3721），就会导致页面无法弹出，商户也就无法接收到通知消息；

采用**服务器通知**的方法，网上支付平台会将支付结果消息通过服务器直接发送给商户指定的URL，而且发送失败以后可以重复发送，这样就保证了商户可以不受消费者本地设置的影响，正确的接收到支付结果通知。

## 对账流程

![对账流程](https://images.niaobulashi.com/typecho/uploads/2020/01/3124490748.png)

## 清算流程

![清算流程](https://images.niaobulashi.com/typecho/uploads/2020/01/2672568168.png)








