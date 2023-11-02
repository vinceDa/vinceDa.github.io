## JdbcTemplate简介

> JdbcTemplate是Spring对数据库的操作在jdbc上面做了深层次的封装，它提供了丰富的api来与数据库进行交互来达到简化开发的目的。


## JdbcTemplate主要提供以下几类方法：

- execute方法：可以用于执行任何SQL语句，一般用于执行DDL语句；
- update方法：update方法用于执行新增、修改、删除等语句；
- batchUpdate方法：用于执行批处理相关语句；
- query方法：用于执行查询相关语句；
- queryForXXX方法：同上，但是返回参数有差异；
- call方法：用于执行存储过程、函数相关语句。

## 开始

### 导入jar包

```xml
   <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-jdbc</artifactId>
      <version>5.3.1</version>
    </dependency>

    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>8.0.22</version>
    </dependency>


    <!-- 数据库连接池  -->
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>druid</artifactId>
      <version>1.2.3</version>
    </dependency>
```

### 连接数据库

**注：这里重点是演示api，连接数据库的方式请忽略**

```java
DruidDataSource dataSource = new DruidDataSource();
dataSource.setUsername("username");
dataSource.setPassword("password");
dataSource.setDriverClassName("com.mysql.jdbc.Driver");
dataSource.setUrl("jdbc:mysql://host:port/dbName");
JDBC_TEMPLATE.setDataSource(dataSource);
```

### update

#### update(sql, Objects... args)

```java
    /**
      *  使用update(sql, Objects... args)方式添加数据
      */
    private static void insertSingleWithUpdate() {
        String sql = "insert into user(username, password, phone, email) values(?,?,?,?)";
        JDBC_TEMPLATE.update(sql, "testUserName2", "testPassword2", 								"13713729113","vincedavince2@163.com");
    }

	/**
     *  使用update(sql, Objects... args)方式更新数据
     */
    private static void updateSingleWithUpdate() {
        String sql = "update user set username=?,password=?,phone=?,email=? where id=?";
        JDBC_TEMPLATE.update(sql, "xx", "yy", "13713729112", "zz@163.com", 36);
    }
```

#### update(sql, object[], int[])

```java
 /**
     *  使用update(sql, object[], int[])方式添加数据
     *   object[]: 存储参数值
     *   int[]: 存储参数的类型, 两个数组中的元素要一一对应
     */
    private static void insertSingle() {
        String sql = "insert into user(username, password, phone, email) values(?,?,?,?)";
        JDBC_TEMPLATE.update(sql, new Object[]{"testUserName2", "testPassword2", "13713729113", 				"vincedavince2@163.com"},
                new int[]{Types.VARCHAR, Types.VARCHAR, Types.VARCHAR, Types.VARCHAR });
    }
```

#### update(PreparedStatementCreator ,  KeyHolder)

```java
    /**
     *  新增返回插入的主键
     */
    private static int insertSingleWithKey() {
        KeyHolder keyHolder = new GeneratedKeyHolder();
        final String sql = "insert into user(username,password,phone,email) values(?,?,?,?)";
        JDBC_TEMPLATE.update(new PreparedStatementCreator() {
            @Override
            public PreparedStatement createPreparedStatement(Connection connection) throws SQLException {
                // 此处不加Statement.RETURN_GENERATED_KEYS会报错
                PreparedStatement preparedStatement = connection.prepareStatement(sql, 						Statement.RETURN_GENERATED_KEYS);
                preparedStatement.setObject(1, "testUserName3");
                preparedStatement.setObject(2, "testPassword3");
                preparedStatement.setObject(3, "13713729114");
                preparedStatement.setObject(4, "vincedavince3@163.com");
                return preparedStatement;
            }
        }, keyHolder);
        return keyHolder.getKey().intValue();
    }
```

### batchUpdate

#### batchUpdate(sql, BatchPreparedStatementSetter)

```java
    /**
     *  使用batchUpdate批量插入数据
     */
    private static void batchInsert() {
        final List<UserEntity> userEntities = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            UserEntity userEntity = new UserEntity();
            userEntity.setUsername("username--" + i);
            userEntity.setPassword("password--" + i);
            userEntity.setEmail("email--" + i);
            userEntity.setPhone("phone--" + i);
            userEntities.add(userEntity);
        }
        String sql = "insert into user(username, password, phone, email) values(?,?,?,?)";
        // ints 为每条数据影响到的行数数量
        int[] ints = JDBC_TEMPLATE.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement preparedStatement, int i) throws SQLException {
                // 注入参数
                preparedStatement.setObject(1, userEntities.get(i).getUsername());
                preparedStatement.setObject(2, userEntities.get(i).getPassword());
                preparedStatement.setObject(3, userEntities.get(i).getPhone());
                preparedStatement.setObject(4, userEntities.get(i).getEmail());
            }

            @Override
            public int getBatchSize() {
                // 批量执行的数量
                return userEntities.size();
            }
        });

    }
```

### query

#### queryForObject(sql, rowMapper)

```java
   /**
     *  queryForObject查询单行多列, 使用rowMapper进行JavaBean和数据库字段之间的映射
     */
    private static void queryForObject() {
        RowMapper<UserEntity> rowMapper = new BeanPropertyRowMapper<>(UserEntity.class);
        String sql = "select * from user";
        UserEntity userEntity = JDBC_TEMPLATE.queryForObject(sql, rowMapper);
        System.out.println(userEntity.toString());
    }
```

#### queryForList(sql, class)

```java
 /**
     *  query查询多行单列
     */
    private static void queryForList() {
        String sql = "select username from user";
        List<String > usernames = JDBC_TEMPLATE.queryForList(sql, String.class);
    }
```

#### query(sql, PreparedStatementSetter)

```java
    private static void query2() {
        String sql = "select * from user where username = ?";
        RowMapper<UserEntity> rowMapper = new BeanPropertyRowMapper<>(UserEntity.class);
        List<UserEntity> users = JDBC_TEMPLATE.query(sql, new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement preparedStatement) throws SQLException {
                preparedStatement.setObject(1, "testUserName2");
            }
        }, rowMapper);
    }
```

## 总结

- 个人使用execute方法少，增删改用update，批量操作用batchUpdate，查询使用query
- 几个主要方法使用的参数大体相同，本文提到的Object... |  object[],int[] | PreparedStatementSetter | BatchPreparedStatementSetter | rowMapper都是较为常用的，使用方法可以从示例里面获取
