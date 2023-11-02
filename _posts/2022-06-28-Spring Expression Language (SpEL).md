[原文地址](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/expressions.html)

# 介绍
Spring Expression Language(简称为SpEL)是一个支持在运行时查询和操作一个对象图的强有力的表达式语言。这门语言的语法和Unified EL类似，但是它提供了额外的功能，最显著的是方法调用和基础的字符串模板化功能。

虽然有一些其他的Java表达式语言，比如说GONL，MVEL和JBoss EL等，Spring Expression Language被创造出来是作为一个单一的得到良好支持的表达式语言来提供给Spring社区使用的，它能够被所有的Spring产品使用。它的功能被Spring的其他产品驱动，包括在基于eclipse的SpringSource工具套件中支持代码补全的要求。这就是说，SpEL基于一个技术不可知的API，允许在需要时整合其他表达式语言的实现。

虽然SpEl是Spring全家桶中表达评价的基础，但是它不直接依赖于Spring，它可以单独被使用。为了自成一体，本章许多的例子都将SpEL作为一种独立的表达语言使用。这需要创建一些引导性的基础设施类，比如解析器。大多数Spring的使用者将不需要处理这个基础设施类，只需要将字符串表达式进行评估。一个关于在XML创建时使用SpEL或者基于bean定义的注解这种类型使用的例子在[Expression support for defining bean definitions](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/expressions.html#expressions-beandef)这一章节有提到。

本章内容包括SpEL的功能，它的API和它的语法。在几个地方，一个发明家和发明家的社会类被用作表达式评估的目标对象。这些类的声明和用于填充它们的数据在本章末尾列出。

# 功能概览
这个表达式语言支持以下的功能

- 字面表达
- 布尔和关系运算符
- 正则表达式
- 类表达式
- 访问属性，数组，列表，哈希表
- 方法调用
- 关系操作
- 任务
- 调用构造函数
- bean引用
- 数组
