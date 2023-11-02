#### 什么是JPA

> 	JPA是Java Persistence API的简称，中文名Java持久层API，是JDK 5.0注解或XML描述对象－关系表的映射关系，并将运行期的实体对象持久化到数据库中。
	Sun引入新的JPA ORM规范出于两个原因：其一，简化现有Java EE和Java SE应用开发工作；其二，Sun希望整合ORM技术，实现天下归一。


#### Spring boot Jpa

> Spring Boot Jpa 是 Spring 基于 ORM 框架、Jpa 规范的基础上封装的一套 Jpa 应用框架，可使开发者用极简的代码即可实现对数据的访问和操作。它提供了包括增删改查等在内的常用功能，且易于扩展！学习并使用 Spring Data Jpa 可以极大提高开发效率！


#### 场景

根据查询条件查询对应的用户, 条件为(可任意组合)

- 用户状态(启用/禁用)
- 根据搜索框的值进行模糊查询, 可根据用户名, 手机号码, 邮箱匹配
- 用户所属部门

#### JpaSpecificationExecutor

> JpaSpecificationExecutor是 JPA 提供的动态接口，利用类型检查的方式，进行复杂的条件查询


```java
package org.springframework.data.jpa.repository;

import java.util.List;
import java.util.Optional;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.lang.Nullable;

public interface JpaSpecificationExecutor<T> {
    Optional<T> findOne(@Nullable Specification<T> var1);

    List<T> findAll(@Nullable Specification<T> var1);

    Page<T> findAll(@Nullable Specification<T> var1, Pageable var2);

    List<T> findAll(@Nullable Specification<T> var1, Sort var2);

    long count(@Nullable Specification<T> var1);
}
```

针对上述场景我们尝试用实现JpaSpecificationExecutor的方式来构建我们的条件, 如下:

```java
    Specification<UserDTO> specification = (Specification<UserDTO>) (root, criteriaQuery, criteriaBuilder) -> {
        List<Predicate> orPredicates = new ArrayList<>();
        List<Predicate> andPredicates = new ArrayList<>();
        if (StrUtil.isNotBlank(userQueryCriteria.getBlurry())) {
            orPredicates.add(criteriaBuilder.like(root.get("name"), "%" + userQueryCriteria.getBlurry() + "%"));
            orPredicates.add(criteriaBuilder.like(root.get("phone"), "%" + userQueryCriteria.getBlurry() + "%"));
            orPredicates.add(criteriaBuilder.like(root.get("email"), "%" + userQueryCriteria.getBlurry() + "%"));
        }
        if (userQueryCriteria.getDepartmentId() != null) {
            andPredicates.add(criteriaBuilder.equal(root.get("departmentId"), userQueryCriteria.getDepartmentId()));
        }
        if (userQueryCriteria.getEnable() != null) {
            andPredicates.add(criteriaBuilder.equal(root.get("enable"), userQueryCriteria.getEnable()));
        }
        Predicate[] orPredicateArray = new Predicate[orPredicates.size()];
        orPredicateArray = orPredicates.toArray(orPredicateArray);
        Predicate[] andPredicateArray = new Predicate[andPredicates.size()];
        andPredicateArray = andPredicates.toArray(andPredicateArray);
        if (orPredicates.isEmpty() || andPredicates.isEmpty()) {
            return criteriaBuilder.or(criteriaBuilder.and(andPredicateArray), criteriaBuilder.or(orPredicateArray));
        }
        return criteriaBuilder.and(criteriaBuilder.and(andPredicateArray), criteriaBuilder.or(orPredicateArray));
    };
userDTOS = userService.listAll(specification, pageable);
```

	上述代码能实现任意组合且满足所有条件的查询，但是代码量极多且重复,真正使用起来极其难看，所以我们考虑将其封装一下。

#### 封装思路

1. 标识查询的字段
2. 定义比较的类型，like、equal、ge、le等等
3. 条件任意组合的实现以及适当的扩展

#### 实现

通常动态查询时，查询参数会被封装成一个query对象

```java
/**
 *  用户查询参数实体类
 * @author vince
 * @date 2019/10/12 16:47
 */
@Data
public class UserQueryCriteria {

    /**
     *  模糊查询条件: 用户名/邮箱/手机号码
     */
    private String blurry;

    /**
     *  是否启用
     */
    private Boolean enable;

    /**
     *  部门id
     */
    private Long departmentId;
}
```

用枚举标识查询连接条件

```
/**
 *  查询的连接条件
 * @author vince
 * @date 2019/10/28 22:27
 */
public enum ConnectConditionEnum {
 
    EQUAL,
    // 下面四个用于Number类型的比较
   
    GT,

    GE,

    LT,

    LE,

    NOT_EQUAL,

    LIKE,

    NOT_LIKE,
    // 下面四个用于可比较类型(Comparable)的比较

    GREATER_THAN,

    GREATER_THAN_OR_EQUAL_TO,

    LESS_THAN,

    LESS_THAN_OR_EQUAL_TO
}
```

定义注解用于标识实体对应的查询字段

```java
/**
 *  查询字段的属性描述
 * @author vince
 * @date 2019/10/28 22:32
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface QueryField {

    /**
     *  数据库字段名, 默认为空即默认Query实体的字段要与数据库的字段一致, 各个字段的column用and连接查询
     */
    String column() default "";

    /**
     *  模糊查询的字段, 默认为空数组即默认Query实体的字段要与数据库的字段一致, 数组中的元素用or连接查询
     */
    String[] blurry() default {};

    /**
     *  查询的连接条件, 例: equal, like, gt, lt...
     */
    ConnectConditionEnum connect() default ConnectConditionEnum.EQUAL;

}
```

然后改造一下query对象

```java
@Data
public class UserQueryCriteria {

    /**
     *  模糊查询条件: 用户名/邮箱/手机号码
     */
    @QueryField(blurry = {"username", "email", "phone"})
    private String blurry;

    /**
     *  是否启用
     */
    @QueryField(column = "enable")
    private Boolean enable;

    /**
     *  部门id
     */
    @QueryField(column = "department_id")
    private Long departmentId;
}
```

基础查询的query对象大都相似，抽象一个BaseQuery来定义通用的行为

```java
**
 *  查询类的基类
 * @author vince
 * @date 2019/10/28 22:31
 */
@Data
public abstract class BaseQuery<T> {

    /**
     *  将查询转换为Specification
     * @return Specification实体
     */
    public abstract Specification<T> toSpecification();

    protected Specification<T> toSpecificationWithAnd() {
        return this.toSpecificationWithLogicType("and");
    }

    public Specification<T> toSpecificationWithOr() {
        return this.toSpecificationWithLogicType("or");
    }

    /**
     *  根据逻辑类型生成Specification实体
     * @param logicType  and/or...
     * @return Specification实体
     */
    private Specification<T> toSpecificationWithLogicType(String logicType) {
        BaseQuery query = this;
        return (Specification<T>) (root, criteriaQuery, criteriaBuilder) -> {
            // 获取查询类的所有字段, 包括父类
            List<Field> fields = listAllFieldWithRoot(query.getClass());
            List<Predicate> predicates = new ArrayList<>(fields.size());
            for (Field single : fields) {
                // 拿到字段上的@QueryField注解
                QueryField queryField = single.getAnnotation(QueryField.class);
                if (queryField == null) {
                    continue;
                }
                single.setAccessible(true);
                try {
                    Object value = single.get(query);
                    if (value == null) {
                        continue;
                    }
                    String column = queryField.column();
                    String[] blurryArray = queryField.blurry();
                    ConnectConditionEnum connect = queryField.connect();
                    Path path;
                    // 模糊查询
                    if (blurryArray.length != 0) {
                        List<Predicate> orPredicates = new ArrayList<>();
                        for (String blurry : blurryArray) {
                            orPredicates.add(criteriaBuilder.like(root.get(blurry), "%" + value + "%"));
                        }
                        Predicate[] predicateArray = new Predicate[orPredicates.size()];
                        predicates.add(criteriaBuilder.or(orPredicates.toArray(predicateArray)));
                        continue;
                    }
                    if (!column.isEmpty()) {
                        path = root.get(column);
                        switch (connect) {
                            case EQUAL:
                                predicates.add(criteriaBuilder.equal(path, value));
                                break;
                            case GT:
                                predicates.add(criteriaBuilder.gt(path, (Number) value));
                                break;
                            case GE:
                                predicates.add(criteriaBuilder.ge(path, (Number) value));
                                break;
                            case LT:
                                predicates.add(criteriaBuilder.lt(path, (Number) value));
                                break;
                            case LE:
                                predicates.add(criteriaBuilder.le(path, (Number) value));
                                break;
                            case NOT_EQUAL:
                                predicates.add(criteriaBuilder.notEqual(path, value));
                                break;
                            case LIKE:
                                predicates.add(criteriaBuilder.like(path, "%" + value + "%"));
                                break;
                            case NOT_LIKE:
                                predicates.add(criteriaBuilder.notLike(path, "%" + value + "%"));
                                break;
                            case GREATER_THAN:
                                predicates.add(criteriaBuilder.greaterThan(path, (Comparable) value));
                                break;
                            case GREATER_THAN_OR_EQUAL_TO:
                                predicates.add(criteriaBuilder.greaterThanOrEqualTo(path, (Comparable) value));
                                break;
                            case LESS_THAN:
                                predicates.add(criteriaBuilder.lessThan(path, (Comparable) value));
                                break;
                            case LESS_THAN_OR_EQUAL_TO:
                                predicates.add(criteriaBuilder.lessThanOrEqualTo(path, (Comparable) value));
                                break;
                            default:
                                break;
                        }
                    }
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
            Predicate predicate = null;
            // and连接
            if (StrUtil.isEmpty(logicType) || "and".equals(logicType)) {
                predicate = criteriaBuilder.and(predicates.toArray(new Predicate[0]));
                // or连接
            } else if ("or".equals(logicType)) {
                predicate = criteriaBuilder.or(predicates.toArray(new Predicate[0]));
            }
            return predicate;
        };
    }

    /**
     *  找到类所有字段包括父类的集合
     * @param clazz 类Class
     * @return 类所有字段的集合
     */
    private List<Field> listAllFieldWithRoot(Class<?> clazz) {
        List<Field> fieldList = new ArrayList<>();
        Field[] fields = clazz.getDeclaredFields();
        if (fields.length != 0) {
            fieldList.addAll(Arrays.asList(fields));
        }
        Class<?> superclass = clazz.getSuperclass();
        // 如果父类是Object, 直接返回
        if (superclass == Object.class) {
            return fieldList;
        }
        // 递归获取所有的父级的Field
        List<Field> superClassFieldList = listAllFieldWithRoot(superclass);
        if (!superClassFieldList.isEmpty()) {
            superClassFieldList.stream()
                    // 去除重复字段
                    .filter(field -> !fieldList.contains(field))
                    .forEach(fieldList::add);

        }
        return fieldList;
    }
}
```

	在BaseQuery中定义了`toSpecificationWithAnd`和`toSpecificationWithOr`方法来表示不同的查询条件，同时也定义了`toSpecification`的抽象方法，当通用的查询方法不满足时可自行实现`toSpecification`方法

	继续改进query对象，继承BaseQuery

```java
@Data
public class UserQueryCriteria extends BaseQuery<UserQueryCriteria> {

    /**
     *  模糊查询条件: 用户名/邮箱/手机号码
     */
    @QueryField(blurry = {"username", "email", "phone"})
    private String blurry;

    /**
     *  是否启用
     */
    private Boolean enable;

    /**
     *  部门id
     */
    @QueryField(column = "department_id")
    // @ApiModelProperty(value = "部门id")
    private Long departmentId;

    @Override
    public Specification<UserQueryCriteria> toSpecification() {
        // 此处可自行扩展
        return super.toSpecificationWithAnd();
    }
}
```

现在可以将最原始的方式替换掉了

```java
userDTOS = userService.listAll(userQueryCriteria.toSpecification(), pageable);
```

有错误的地方还望指出！

本文借鉴：[Spring-Data-JPA 动态查询黑科技](https://juejin.cn/post/6844903502888566792)

感谢！
