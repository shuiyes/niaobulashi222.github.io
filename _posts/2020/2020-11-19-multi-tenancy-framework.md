---
layout: post
title: 如何理解多租户架构？
category: java
tags: [java]
copyright: java

---

　　前段时间公司产品进行了架构的进化，进化到了多租户架构。当我第一次听到多租户时，我也挺纳闷，不理解。但当我逐渐的翻阅资料，以及研发功能时。不断的加深了对多租户的理解。尽管我现在也只是浅浅的懂一点而已。

　　OK,Let's get this straight（让我们搞懂它），接下来让我们问自己几个问题:

　　1.什么是多租户架构？
　　2.多租户架构的优缺点？
　　3.多租户架构的适用场景？

　　让我们带着这几个问题进入下面的阅读。

一、对多租户的理解

　　多租户定义：多租户技术或称多重租赁技术，简称SaaS，是一种软件架构技术，是实现如何在多用户环境下（此处的多用户一般是面向企业用户）共用相同的系统或程序组件，并且可确保各用户间数据的隔离性。简单讲：在一台服务器上运行单个应用实例，它为多个租户（客户）提供服务。从定义中我们可以理解：多租户是一种架构，目的是为了让多用户环境下使用同一套程序，且保证用户间数据隔离。那么重点就很浅显易懂了，多租户的重点就是同一套程序下实现多用户数据的隔离。对于实现方式，我们下面会讨论到。

　　在了解详细一点：在一个多租户的结构下，应用都是运行在同样的或者是一组服务器下，这种结构被称为“单实例”架构（Single Instance），单实例多租户。多个租户的数据是保存在相同位置，依靠对数据库分区来实现隔离操作。既然用户都在运行相同的应用实例，服务运行在服务供应商的服务器上，用户无法去进行定制化的操作，所以这对于对该产品有特殊需要定制化的客户就无法适用，所以多租户适合通用类需求的客户。那么缺点来了，多租户下无法实现用户的定制化操作。

　　在翻阅多租户的资料时，还有一个名词与之相对应，那就是单租户SaaS架构（也被称作多实例架构（Multiple Instance））。单租户架构与多租户的区别在于，单租户是为每个客户单独创建各自的软件应用和支撑环境。单租户SaaS被广泛引用在客户需要支持定制化的应用场合，而这种定制或者是因为地域，抑或是他们需要更高的安全控制。通过单租户的模式，每个客户都有一份分别放在独立的服务器上的数据库和操作系统，或者使用强的安全措施进行隔离的虚拟网络环境中。因为本篇主要是讨论多租户，所以单租户的相关知识就简单了解一下，不做过多的阐述了。

二、多租户数据隔离的三种方案

　　在当下云计算时代，多租户技术在共用的数据中心以单一系统架构与服务提供多数客户端相同甚至可定制化的服务，并且仍可以保障客户的数据隔离。目前各种各样的云计算服务就是这类技术范畴，例如阿里云数据库服务（RDS）、阿里云服务器等等。

　　多租户在数据存储上存在三种主要的方案，分别是：

　　1. 独立数据库

　　这是第一种方案，即一个租户一个数据库，这种方案的用户数据隔离级别最高，安全性最好，但成本较高。 
　　优点： 
　　　　为不同的租户提供独立的数据库，有助于简化数据模型的扩展设计，满足不同租户的独特需求；如果出现故障，恢复数据比较简单。 
　　缺点： 
　　　　增多了数据库的安装数量，随之带来维护成本和购置成本的增加。 
　　这种方案与传统的一个客户、一套数据、一套部署类似，差别只在于软件统一部署在运营商那里。如果面对的是银行、医院等需要非常高数据隔离级别的租户，可以选择这种模式，提高租用的定价。如果定价较低，产品走低价路线，这种方案一般对运营商来说是无法承受的。

　　2. 共享数据库，独立 Schema 
　　这是第二种方案，即多个或所有租户共享Database，但是每个租户一个Schema（也可叫做一个user）。底层库比如是：DB2、ORACLE等，一个数据库下可以有多个SCHEMA
　　优点： 
　　　　为安全性要求较高的租户提供了一定程度的逻辑数据隔离，并不是完全隔离；每个数据库可支持更多的租户数量。
　　缺点： 
　　　　如果出现故障，数据恢复比较困难，因为恢复数据库将牵涉到其他租户的数据； 
　　如果需要跨租户统计数据，存在一定困难。

　　3. 共享数据库，共享 Schema，共享数据表
　　这是第三种方案，即租户共享同一个Database、同一个Schema，但在表中增加TenantID多租户的数据字段。这是共享程度最高、隔离级别最低的模式。
　　即每插入一条数据时都需要有一个客户的标识。这样才能在同一张表中区分出不同客户的数据。
　　优点： 
　　　　三种方案比较，第三种方案的维护和购置成本最低，允许每个数据库支持的租户数量最多。 
　　缺点： 
　　　　隔离级别最低，安全性最低，需要在设计开发时加大对安全的开发量； 数据备份和恢复最困难，需要逐表逐条备份和还原。

　　如果希望以最少的服务器为最多的租户提供服务，并且租户接受牺牲隔离级别换取降低成本，这种方案最适合。 
　　　　
　　在SaaS实施过程中，有一个显著的考量点，就是如何对应用数据进行设计，以支持多租户，而这种设计的思路，是要在数据的共享、安全隔离和性能间取得平衡。

　　因为我们用的底层库是MySQL，且要保证数据的完全隔离，所以用的方案属于第一种。独立数据库。因为MySQL下SCHEMA就是他的数据库名。所以每多服务一个用户，都需要新建一个数据库。如果是DB2或者是ORACLE的话，一个数据库下，可以采用独立的SCHEMA来进行数据隔离，这样会相对节省成本，且数据隔离的强度高。

三、选择合理的实现模式 
　　衡量三种模式主要考虑的因素是隔离还是共享。

　　成本角度因素 

　　　　隔离性越好，设计和实现的难度和成本越高，初始成本越高。共享性越好，同一运营成本下支持的用户越多，运营成本越低。

　　安全因素 

　　　　要考虑业务和客户的安全方面的要求。安全性要求越高，越要倾向于隔离。

　　从租户数量上考虑
　　　　主要考虑下面一些因素 
　　　　系统要支持多少租户？上百？上千还是上万？可能的租户越多，越倾向于共享。 
　　　　平均每个租户要存储数据需要的空间大小。存贮的数据越多，越倾向于隔离。 
　　　　每个租户的同时访问系统的最终用户数量。需要支持的越多，越倾向于隔离。 
　　　　是否想针对每一租户提供附加的服务，例如数据的备份和恢复等。这方面的需求越多， 越倾向于隔离

　　技术储备 
　　　　共享性越高，对技术的要求越高。

--转载学习