## 什么是MapStrcut？
MapStruct是一个代码生成器，它基于约定大于配置的方式极大简化了Java bean间的字段映射实现。

它生成的映射代码使用原生的普通方法调用(get/set)，所以它是快速的、类型安全且容易理解的。



## 为什么要使用MapStruct？
多层架构的应用通常会要求不同的对象模型间做映射（vo -> Dto -> po）。写这样的映射代码是乏味且易错的工作。MapStruct旨在通过尽可能多的自动生成这样的代码来简化这个工作。



和其他映射框架相比。MapStruct在编译时期生成代码确保了高性能，允许开发者得到更快的反馈以及彻底的错误检查。



这里有一篇[文章](https://vinceda.github.io/posts/Java-Bean%E8%BD%AC%E6%8D%A2%E6%A1%86%E6%9E%B6%E6%80%A7%E8%83%BD%E5%AF%B9%E6%AF%94(%E8%AF%91)/)介绍了几种Java映射框架的性能。从各方面来看，MapStruct是一个优秀的映射框架。



## 如何使用MapStruct？
MapStruct是一个注解处理器，它被插入到Java编译器中并且可以通过命令行构建（Maven、Gradle等）也可在你喜欢的IDE中使用。



MapStruct使用合理的默认值，但当涉及到配置或实现特殊行为时，就会根据你的方式来执行。



接下来我将从几种常见的场景来介绍MapStruct的使用方式。由于展示的场景过多，为了让读者专注了解MapStruct的用法，我们会将示例代码上传到[这里](https://github.com/vinceDa/mapstruct-example)。你可以通过单测查看结果，通过生成的impl类查看MapStruct的生成逻辑。



这里将省略了所有非必要的的代码，只留下了源对象（无后缀）和目标对象（dto为后缀）以及用于映射的XXMapper。



### 数据准备
依赖
```xml
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.4.2.Final</version>
</dependency>

<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct-processor</artifactId>
  <version>1.4.2.Final</version>
</dependency>
```


所有源对象和目标对象如下:



Father: 主要是createTime的属性不一样，且包含了sons的对象

```java
public class Father {

    private String name;

    private Integer age;

    private List<Son> sons;

    private LocalDateTime createTime;

    // get && set
}

public class FatherDto {

    private String name;

    private Integer age;

    private List<SonDto> sonDtos;

    private String create_time;

    // get && set
}
```



Mather和Son: 完全一致

```java
public class Mather {

    private String name;

    private Integer age;

    private LocalDateTime createTime;

    // get && set
}

public class MatherDto {

    private String name;

    private Integer age;

    private LocalDateTime createTime;

    // get && set
}
```



```java
public class Son {

    private String name;

    private Integer age;

    private String createdBy;
    
    // get && set
}

public class SonDto {

    private String name;

    private Integer age;

    private String createdBy;
    
    // get && set
}
```



Familly: 把Father、Mather和Son的name属性放在了一起

```java
public class Family {

    private String fatherName;

    private String sonName;

    private String matherName;
    
    // get && set
}
```



### 基础用法
```java
/**
 * 基础映射
 * @author vince
 * @date 2023/6/15 11:53
 */
@Mapper
public interface BaseMapper {
    BaseMapper instance = Mappers.getMapper(BaseMapper.class);
    /**
     * 相同字段名默认转换, 不需要添加额外注解
     */
    MatherDto toDto(Mather mather);
    /**
     * 不同字段名需要添加@Mapping注解
     */
    @Mapping(target = "created_by", source = "createdBy")
    SonDto toDto(Son son);
    /**
     * 集合映射会自动找到已存在的映射方法
     */
    List<SonDto> toDto(List<Son> sons);
}
```


从例子可以看到：

1. 相同字段会默认映射，不需要额外配置

2. 不同字段需要添加@Mapping注解，target表示你转换后的字段名，source表示源字段名

3. 集合的映射会去找单个对象的映射，然后通过for循环映射，实现如下：

   ```java
    public List<SonDto> toDto(List<Son> sons) {
           if ( sons == null ) {
               return null;
           }
           
           List<SonDto> list = new ArrayList<SonDto>( sons.size() );
           for ( Son son : sons ) {
             // for循环中使用单个对象的映射方法
               list.add( toDto( son ) );
           }
   
           return list;
       }
   ```

   

### 对象嵌套

在实际场景中，我们经常会遇到存在实体中包含其他实体。例如菜单树，订单和子订单等。接下来我们就来看看这种场景应该如何实现。



```java
@Mapper(uses = {BaseMapper.class})
public interface NestedMapper {

    NestedMapper instance = Mappers.getMapper(NestedMapper.class);

    /**
     * 这里可以引用BaseMapper中已存在的映射方法, 也可以直接把BaseMapper中的方法复制过来
     * 像这样:
     *     @Mapping(target = "created_by", source = "createdBy")
     *     SonDto toDto(Son son);
     * 或者通过default方法直接实现:
     *    default SonDto toDto(Son son) {
     *       ....
     *    }
     *
     */
    @Mapping(target = "sonDtos", source = "sons")
    FatherDto toDto(Father father);

}
```

这里最重要的一行代码就是开头的**@Mapper(uses = {BaseMapper.class})**，它表示除了使用当前映射类的规则外，还可以去BaseMapper中寻找适应的规则（这里找的是son对象的映射）。



### 多参数合并

这里我们将Father、Mother和Son的名称全部在Family对象中展示（对应到实际场景就是订单详情中要显示商品详情），实现代码如下：

```java
@Mapper
public interface MergeMapper {

    MergeMapper instance = Mappers.getMapper(MergeMapper.class);

    @Mapping(target = "fatherName", source = "father.name")
    @Mapping(target = "matherName", source = "mather.name")
    @Mapping(target = "sonName", source = "son.name")
    Family merge(Father father, Mather mather, Son son);
}
```


### 使用Java表达式映射（expression的使用）

```java
@Mapper(uses = {BaseMapper.class})
public interface ExpressionMapper {

    ExpressionMapper instance = Mappers.getMapper(ExpressionMapper.class);

    /**
     * constant: 常量
     * expression: java表达式
     * dateFormat内置的表达式
     */
    @Mapping(target = "name", constant = "father")
    @Mapping(target = "sonDtos", expression = "java(father.getSons() == null ? new java.util.ArrayList<>() : BaseMapper.instance.toDto(father.getSons()))")
    @Mapping(target="create_time", source = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    FatherDto toDto(Father father);
}
```



这里有几点：

1. 常量可以用constant表示
2. 可以使用MapStruct的内置表达式，例如dateFormat
3. 通过expression来执行java代码，代码必须包裹在java()中，否则会报错



### 使用注解

使用注解可以实现项目范围的公共参数映射，例如我会做一个MybatisPlus的分页对象Page和项目内定义的分页对象PageResponse的注解映射，用于所有的分页参数转换。



创建注解：

```java
@Retention(RetentionPolicy.CLASS)
@Mappings({
        @Mapping(target = "name", constant = "0"),
        @Mapping(target = "created_by", source = "createdBy")
})
public @interface ToAnnotation {
}
```



使用注解：

```java
@Mapper
public interface AnnotationMapper {

    AnnotationMapper instance = Mappers.getMapper(AnnotationMapper.class);

    /**
     * 使用注解对公共字段进行映射
     */
    @ToAnnotation
    SonDto toDto(Son son);

}
```



### 映射的前后置处理

在映射开始前和结束后对实体做对应的改动：

```java
@Mapper
public abstract class AbstractMapper {
    /**
     * 前置处理
     */
    @BeforeMapping
    protected void setCusName(Father father) {
        father.setName("father_before");
    }

    /**
     * 后置处理
     */
    @AfterMapping
    protected void setCusAge(@MappingTarget FatherDto fatherDto) {
        fatherDto.setAge(999);
    }

    public abstract FatherDto toDto(Father father);
}
```



### 在Spring中使用

其实也很简单，在Mapper接口中的@Mapper注解内添加对应的配置即可。Mapper被Spring托管后，创建instance的代码也可以省略了。
```java
@Mapper(componentModel = "spring")
// MultiMapper INSTANCE = Mappers.getMapper(MultiMapper.class);
```



## 常见问题

### 1. 和lombok冲突怎么解决？
话不多说，直接上Maven配置解决。
```xml
<properties>
    <org.projectlombok.version>1.18.16</org.projectlombok.version>
    <org.mapstruct.version>1.4.2.Final</org.mapstruct.version>
    <lombok-mapstruct-binding.version>0.2.0</lombok-mapstruct-binding.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${org.projectlombok.version}</version>
    </dependency>

    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>${org.projectlombok.version}</version>
                    </path>
                    <!-- This is needed when using Lombok 1.18.16 and above -->
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok-mapstruct-binding</artifactId>
                        <version>${lombok-mapstruct-binding.version}</version>
                    </path>
                    <!-- Mapstruct should follow the lombok path(s) -->
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${org.mapstruct.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```


值得注意的是：

1. 确保Lombok 最低版本为 1.18.16
2. annotationProcessorPaths 中，mapstruct-processor 的配置要在 lombok 之后



## 总结

这篇文章介绍了MapStruct的几种基本用法，其实除了这些用法，MapStruct还有许多高阶用法。由于我也没有深入使用过，这里就先不写了，后续用到会持续进行更新，如果写的有问题欢迎大家指出，谢谢！
