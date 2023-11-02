---
tags: [ddd]
---

## 什么是Domain Primitive？
Domain Primitive翻译成中文就是领域原型。它的定义就想当于Java中的八大基础类型，Java需要在这些基础类型之上去构建更加复杂的特性。Domain Primitive也是如此，它是领域驱动设计中最基础的一种类型。它的设计思路也向我们展示了领域模型核心的思想。

引入Domain Primitive有三大原则：

1. MakeImplicitConceptsExplicit：将隐性的概念显性化
2. MakeImplicitContextExplicit：将隐性的上下文显性化

3. Encapsulate Multi-Object Behavior：封装多对象行为

可能现在看到这几个原则有点一头雾水，不过没关系。接下来我们将通过三个案例去分析其中不合理的地方来解释这几个原则。

## 面向过程的编码有什么坏处？
```java
/**
 * 更新用户的身份证号，同时根据身份证号自动填充生日、性别字段
 */
public void updateCustomerIdCard(String customerGUID, String idCard) throws ValidationException, ParseException {
    // 假如此处需要将参数的具体错误信息返回, 所有这样写的代码都需要改动
    if (CharSequenceUtil.isEmpty(customerGUID) || !Validator.isUUID(customerGUID)) {
        throw new ValidationException("customerGUID");
    }
    if (CharSequenceUtil.isEmpty(idCard) || !IdcardUtil.isValidCard18(idCard)) {
        throw new ValidationException("idCard");
    }
    String birth = idCard.substring(6, 14);
    DateFormat format = new SimpleDateFormat("yyyyMMdd");
    Date parse = format.parse(birth);
    // 第17位数为奇数是男性, 偶数为女性
    int sex = idCard.charAt(16) % 2 == 0 ? 0 : 1;
    Customer customer = new Customer();
    customer.setCustomerGUID(customerGUID);
    customer.setIdCard(idCard);
    customer.setSex(sex);
    customer.setBirthDay(parse);
    System.out.println("insert success: " + customer.toString());
}
```

我们之前说了，这几个案例是不合理的。所以，我将从以下几个方面来说明它为什么不合理，合理的做法又是什么。

### 接口清晰度
```java
updateCustomerIdCard(String customerGUID, String idCard)
```
乍一看，这就是一个根据customerGUID更新客户身份证号的接口，看起来没什么问题。

但是如果客户端调用的时候，将两个参数传反了，程序会报错嘛？答案是肯定的，会报错，因为我们有数据校验。但是它是运行时错误，很有可能在上线之后才会出现这个问题。所以，我们有什么办法能够避免这种问题呢？

### 数据校验
```java
if (CharSequenceUtil.isEmpty(idCard) || !IdcardUtil.isValidCard18(idCard)) {
    throw new ValidationException("idCard");
}
```
继续往下走，这段数据校验逻辑在普通不过了，几乎每个方法的开始都会有这一步操作去校验数据的合法性。

继续找茬！假如我们的项目中有很多类似的接口都要用到这个数据校验，我们自然的会写多个类似的校验方式去做这个工作。这时候，需求变了，我们的校验内容需要修改，然后我们去挨个修改校验方法，一不小心漏了一个，完了，出bug了。

项目运行了一段时间后，客户反应我们的报错信息不明确，需要针对每种错误提示不同的错误信息。我们又要面对上面提到的这种情况了，在项目越来越复杂的情况下，维护成本会变得越来越高。

### 业务代码的清晰度
我们可以通过方法的注释知道在修改idCard的时候同步修改了生日和性别。但是假如不看注释，你看到这些代码的时候是不是一脸懵？
```java
idCard.substring(6, 14);
idCard.charAt(16);
```

### 可测试性
假设一个方法有M个参数，N种校验逻辑，那么这个方法的测试用例个数为M * N。如果X个方法的测试用例为M * N * X。仅仅是这样简单的计算，我们就能感觉到测试的成本有多大了。

### 解决方案
用自定义的强类型替换String，相关的业务封装在强类型中。

我们分别将String customerGUID和String idCard分别用两个强类型来替代。
```java
/**
 * @author ohYoung
 * @date 2021/9/13 0:36
 */
public class CustomerGUID {

    private final String customerGUID;

    public CustomerGUID(String customerGUID) throws ValidationException {
        if (CharSequenceUtil.isEmpty(customerGUID) || !Validator.isUUID(customerGUID)) {
            throw new ValidationException("param is not a UUID type");
        }
        this.customerGUID = customerGUID;
    }
}


/**
 * @author ohYoung
 * @date 2021/9/13 0:36
 */
public class IdCard {
    private final String idCard;

    public IdCard(String idCard) throws ValidationException {
        if (CharSequenceUtil.isEmpty(idCard)) {
            throw new ValidationException("身份证号不能为空");
        } else if (!IdcardUtil.isValidCard18(idCard)) {
            throw new ValidationException("身份证号格式错误");
        }
        this.idCard = idCard;
    }

    public Date getBirthDay() throws ParseException {
        String birth = idCard.substring(6, 14);
        DateFormat format = new SimpleDateFormat("yyyyMMdd");
        return format.parse(birth);
    }

    public Integer getSex() {
        return idCard.charAt(16) % 2 == 0 ? 0 : 1;
    }

}
```
这里介绍一下主要修改了哪些地方：

1. private final String idCard保证了IdCard类型的不可变性
2. 校验逻辑放在对象内部
3. 业务方法封装在对象内部，作为对象的方法提供给外部调用

在用DP对象替换我们的面向过程编程的代码后：
```java
/**
 * @author ohYoung
 * @date 2021/9/12 23:25
 */
public class DpCode {

    public void updateCustomerIdCard(CustomerGUID customerGUID, IdCard idCard) throws ParseException {
        DpCustomer customer = new DpCustomer();
        customer.setCustomerGUID(customerGUID);
        customer.setIdCard(idCard);
        customer.setSex(idCard.getSex());
        customer.setBirthDay(idCard.getBirthDay());
        System.out.println("insert success: " + customer.toString());
    }
}
```
我们可以很明显的看到，代码变得十分清晰，没有多余的非业务流程（校验、获取出生日期的过程等）。既然我们之前从四个维度说了传统代码的坏处，我们现在也从这几个方面说说使用DP的好处：

1. 接口清晰度：接口参数换成了强类型参数，如果参数传错会在编译期间报错
2. 数据校验：校验方法封装在对象内部，校验逻辑发生改变只需要修改一个对象即可
3. 业务代码的清晰度：所有非业务的代码全部在对象体内，看到的都是核心业务逻辑，十分清晰
4. 可测试性：针对单个方法的测试用例个数为M + N，多个方法的测试用例数量为N + M + P，大大降低了测试成本

这其实就是我们上面说的第一个原则：**将隐性的概念显性化。**

## 隐性的上下文带来的隐患？
什么是上下文？
比如金额为1，1元、1美金、1欧元的价值是不同的。这里的元、美金、欧元就是上下文，1在这几个不同的上下文内表示的内容都不相同。

假如我们现在要实现一个转账功能，A用户转账X元给B用户，用伪代码实现如下：
```java
public void transfer (BigDecimal money) {
	TransferService.trasfer(money, "CNY");
}
```
这段代码表示的是默认转账的货币为元，假如某一天业务发生转变导致货币变更，那这就是一个bug。

在这个示例中，货币就是一个隐性的上下文。

根据DP的第二个原则，我们将代码改造如下：
```java
public class Currency {
    private String currency;
    public Currency(String currency) {
        this.currency = currency;
    }
}

public class Money {
    private BigDecimal amount;
    private Currency currency;
    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
}

public void transfer (Money money) {
	TransferService.trasfer(money);
}
```
通过将属性currency和金额amount组合成一个Money对象将隐性的上下文显性化，避免了很多当前可能看不出来但是以后会造成严重后果的bug。

## 多个对象的业务逻辑应该怎么处理？
以转账的案例为基础，假如A用户和B用户所用的货币不同，那么自然地就涉及到汇率转换了，那么我们的代码就有可能是下面这样的：
```java
public void pay(Money money, Currency targetCurrency) {
    // 如果货币相同，走原来的逻辑
    if (money.getCurrency().equals(targetCurrency)) {
        TransferService.transfer(money);
    } else {
        // 如果货币不同，先调用远程服务获取汇率，然后根据汇率计算最终的金额
        BigDecimal rate = ExchangeService.getRate(money.getCurrency(), targetCurrency);
        BigDecimal targetAmount = money.getAmount().multiply(new BigDecimal(rate));
        Money targetMoney = new Money(targetAmount, targetCurrency);
        TransferService.transfer(targetMoney);
    }
}
```
从上面的代码可以看出，当涉及到多个对象的计算时，面向过程的编码好像又出现了。那么应该怎么办呢？
```java
public class ExchangeRate {
    private BigDecimal rate;
    private Currency from;
    private Currency to;

    public ExchangeRate(BigDecimal rate, Currency from, Currency to) {
        this.rate = rate;
        this.from = from;
        this.to = to;
    }

    public Money exchange(Money fromMoney) {
        notNull(fromMoney);
        isTrue(this.from.equals(fromMoney.getCurrency()));
        BigDecimal targetAmount = fromMoney.getAmount().multiply(rate);
        return new Money(targetAmount, to);
    }
}
```
很简单！根据基础的DP对象构建上一层的DP对象，将基础DP对象之间的业务行为封装成方法供客户端调用。改造后的代码如下：
```java
public void pay(Money money, Currency targetCurrency) {
   ExchangeRate rate = ExchangeService.getRate(money.getCurrency(), targetCurrency);
   Money targetMoney = rate.exchange(money);
   TransferService.transfer(targetMoney);
}
```

## 什么情况下使用DP

- 有格式限制的 `String`：比如`Name`，`PhoneNumber`，`OrderNumber`，`ZipCode`，`Address`等
- 有限制的`Integer`：比如`OrderId`（>0），`Percentage`（0-100%），`Quantity`（>=0）等
- 可枚举的 `int`：比如 `Status`（一般不用Enum因为反序列化问题）
- `Double`或 `BigDecimal`：一般用到的`Double`或`BigDecimal`都是有业务含义的，比如 `Temperature`、`Money`、`Amount`、`ExchangeRate`、`Rating`等
- 复杂的数据结构：比如 `Map<String,List<Integer>>`等，尽量能把 `Map`的所有操作包装掉，仅暴露必要行为

## 总结
我们从接口清晰度、数据校验、业务代码的清晰度和可测试性四个方面来讲述了DP给我们的业务带来的好处。我们评估一个项目开发的成本不仅仅是开发方面，测试成本、阅读代码成本、重构成本以及代码演进的成本都需要考虑在内。而DP的思想则帮助我们很好的解决了这些方面的问题，当我们判断我们的业务符合DP的使用场景后，我们应该尽可能地去使用它来让我们的代码变得更加简洁、灵活且健壮。

## 参考

1. [阿里技术专家详解 DDD 系列- Domain Primitive](https://developer.aliyun.com/article/716908)，本文绝大部分内容都是借鉴的这篇文章，全是干货，我只是根据我的理解再复述了一遍
2. [Domain Primitives: what they are and how you can use them to make more secure software](https://freecontent.manning.com/domain-primitives-what-they-are-and-how-you-can-use-them-to-make-more-secure-software/)
