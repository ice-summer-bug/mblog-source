---
layout: post
title: Jackson 中的 Features
categories: Java
tags: [Json, Java]
date: 2016-07-10 21:00:00
description: jackson 可选特性说明
---
## Jackson 中的各种 Features


先来看看 jackson 中的各种feature
![图片](/assets/picture/jacksonFeature.png "Feature 类图")

首先是 序列化时候的Feature ——

### SerializaitonFeature

--------

#### `WRAP_ROOT_VALUE(false)` : 序列化的json是否显示根节点

```java
public static void testWrapRoot() {
       SimpleBean simpleBean = new SimpleBean();
       simpleBean.setCode(1);
       simpleBean.setName("TEST_WRAP_ROOT_VALUE");
       simpleBean.setDesc(Lists.newArrayList("test1", "test2", "test3"));

       ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.enable(SerializationFeature.WRAP_ROOT_VALUE);
       try {
           String s = objectMapper.writeValueAsString(simpleBean);
           System.out.println(s);
       } catch (JsonProcessingException e) {
           e.printStackTrace();
       }
   }
```

运行结果：

    {"name":"TEST_WRAP_ROOT_VALUE","code":1,"desc":["test1","test2","test3"]}

去掉注释的运行结果：

    {"SimpleBean":{"name":"TEST_WRAP_ROOT_VALUE","code":1,"desc":["test1","test2","test3"]}}


从结果可以看出 jackson 会把类名作为根节点展示

---------

#### `INDENT_OUTPUT(false)`: 允许或禁止是否以缩进的方式展示json

```java
public static void testIdentOutput() {
    SimpleBean simpleBean = new SimpleBean();
    simpleBean.setCode(1);
    simpleBean.setName("TEST_WRAP_ROOT_VALUE");
    simpleBean.setDesc(Lists.newArrayList("test1", "test2", "test3"));

    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.enable(SerializationFeature.INDENT_OUTPUT);
    try {
        String s = objectMapper.writeValueAsString(simpleBean);
        System.out.println(s);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
}
```

输出的结果：

    {
        "name" : "TEST_WRAP_ROOT_VALUE",
        "code" : 1,
        "desc" : [ "test1", "test2", "test3" ]
    }

----------

#### `FAIL_ON_EMPTY_BEANS`(true): 当类的一个属性外部无法访问(如：没有getter setter 的私有属性)，且没有annotation 标明需要序列化时，如果FAIL_ON_EMPTY_BEANS 是true 将会跑出异常，如果是false 则不会跑出异常

```java
public static void testFailOnEmptyField() {
        TestBean simpleBean = new TestBean();

        ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        try {
            String s = objectMapper.writeValueAsString(simpleBean);
            System.out.println(s);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
    }

    private static class TestBean {
        private String name;
    }
```

代码运行结果

        com.fasterxml.jackson.databind.JsonMappingException: No serializer found for class com.liam.learning.Main$TestBean and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS)
          at com.fasterxml.jackson.databind.JsonMappingException.from(JsonMappingException.java:275)
          at com.fasterxml.jackson.databind.SerializerProvider.mappingException(SerializerProvider.java:1109)
          at com.fasterxml.jackson.databind.SerializerProvider.reportMappingProblem(SerializerProvider.java:1134)
          at com.fasterxml.jackson.databind.ser.impl.UnknownSerializer.failForEmpty(UnknownSerializer.java:69)
          at com.fasterxml.jackson.databind.ser.impl.UnknownSerializer.serialize(UnknownSerializer.java:32)
          at com.fasterxml.jackson.databind.ser.DefaultSerializerProvider.serializeValue(DefaultSerializerProvider.java:292)
          at com.fasterxml.jackson.databind.ObjectMapper._configAndWriteValue(ObjectMapper.java:3672)
          at com.fasterxml.jackson.databind.ObjectMapper.writeValueAsString(ObjectMapper.java:3048)
          at com.liam.learning.Main.testFailOnEmptyField(Main.java:56)
          at com.liam.learning.Main.main(Main.java:15)
          at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
          at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
          at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
          at java.lang.reflect.Method.invoke(Method.java:606)
          at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)

取消注释的运行结果：

    {}

--------

#### FAIL_ON_SELF_REFERENCES(true)
如果POJO 中有一个直接自我引用，在序列化的时候会抛出 `com.fasterxml.jackson.databind.JsonMappingException`

------------------------------------------------

#### WRAP_EXCEPTIONS(true)
如果序列化过程中，如果抛出 Exception 将会被包装，添加额外的上下文信息

------------------------------------------------

#### FAIL_ON_UNWRAPPED_TYPE_IDENTIFIERS(true)
该特性决定了，用 `com.fasterxml.jackson.annotation.JsonUnwrapped` 注解修饰的属性，且该属性的类型还有类型信息（被 `com.fasterxml.jackson.annotation.JsonTypeInfo` 注解修饰）的时候，是否抛出异常；默认情况下会抛出异常，除非禁止该特性

使用示例如下：

```java
    public static void testFailOnUnwrappedTypeIdentifiers() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.enable(SerializationFeature.INDENT_OUTPUT);
        // 禁止特性 FAIL_ON_UNWRAPPED_TYPE_IDENTIFIERS
        objectMapper.disable(SerializationFeature.FAIL_ON_UNWRAPPED_TYPE_IDENTIFIERS);

        TestJsonPojo.TestInputInnerJsonPojo testInputInnerJsonPojo = new TestJsonPojo.TestInputInnerJsonPojo();
        testInputInnerJsonPojo.setDesc("desc1");
        testInputInnerJsonPojo.setTip("tip1");
        testInputInnerJsonPojo.setInput("input information");

        TestJsonPojo testJsonPojo = new TestJsonPojo();
        testJsonPojo.setName("the test name");
        testJsonPojo.setDescription(testInputInnerJsonPojo);

        try {
            System.out.println(objectMapper.writeValueAsString(testJsonPojo));

            System.out.println("===================================================");

            TestJsonPojo.TestSelectInnerJsonPojo testSelectInnerJsonPojo = new TestJsonPojo.TestSelectInnerJsonPojo();
            testSelectInnerJsonPojo.setDesc("desc1");
            testSelectInnerJsonPojo.setTip("tip1");
            testSelectInnerJsonPojo.setSelect("select information");
            testJsonPojo.setDescription(testSelectInnerJsonPojo);
            System.out.println(objectMapper.writeValueAsString(testJsonPojo));
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
    }

    private static class TestJsonPojo {

        private String name;

        @JsonUnwrapped
        private TestInnerJsonPojo description;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public TestInnerJsonPojo getDescription() {
            return description;
        }

        public void setDescription(TestInnerJsonPojo description) {
            this.description = description;
        }

        @JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type", visible = true)
        @JsonSubTypes(value = {
                @JsonSubTypes.Type(value = TestInputInnerJsonPojo.class, name = "input"),
                @JsonSubTypes.Type(value = TestInputInnerJsonPojo.class, name = "select")
        })
        private static abstract class TestInnerJsonPojo {
            private String tip;
            private String desc;

            public String getTip() {
                return tip;
            }

            public void setTip(String tip) {
                this.tip = tip;
            }

            public String getDesc() {
                return desc;
            }

            public void setDesc(String desc) {
                this.desc = desc;
            }
        }

        private static class TestInputInnerJsonPojo extends TestInnerJsonPojo {
            private String input;

            public String getInput() {
                return input;
            }

            public void setInput(String input) {
                this.input = input;
            }
        }

        private static class TestSelectInnerJsonPojo extends TestInnerJsonPojo {
            private String select;

            public String getSelect() {
                return select;
            }

            public void setSelect(String select) {
                this.select = select;
            }
        }
    }
```

上述代码可以执行成功，输出如下：
                
                {
                "name" : "the test name",
                "tip" : "tip1",
                "desc" : "desc1",
                "input" : "input information"
                }
                ===================================================
                {
                "name" : "the test name",
                "tip" : "tip1",
                "desc" : "desc1",
                "select" : "select information"
                }

                Process finished with exit code 0


而当我们不禁止该特性的时候

```java
    objectMapper.enable(SerializationFeature.FAIL_ON_UNWRAPPED_TYPE_IDENTIFIERS);
```

则会抛出异常

                com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Unwrapped property requires use of type information: cannot serialize without disabling `SerializationFeature.FAIL_ON_UNWRAPPED_TYPE_IDENTIFIERS` (through reference chain: com.liam.learning.jackson.features.SerializationFeatureDemo$TestJsonPojo["description"])
                at com.fasterxml.jackson.databind.exc.InvalidDefinitionException.from(InvalidDefinitionException.java:77)
                at com.fasterxml.jackson.databind.SerializerProvider.reportBadDefinition(SerializerProvider.java:1277)
                at com.fasterxml.jackson.databind.DatabindContext.reportBadDefinition(DatabindContext.java:400)
                at com.fasterxml.jackson.databind.ser.impl.UnwrappingBeanSerializer.serializeWithType(UnwrappingBeanSerializer.java:143)
                at com.fasterxml.jackson.databind.ser.impl.UnwrappingBeanPropertyWriter.serializeAsField(UnwrappingBeanPropertyWriter.java:129)
                at com.fasterxml.jackson.databind.ser.std.BeanSerializerBase.serializeFields(BeanSerializerBase.java:755)
                at com.fasterxml.jackson.databind.ser.BeanSerializer.serialize(BeanSerializer.java:178)
                at com.fasterxml.jackson.databind.ser.DefaultSerializerProvider._serialize(DefaultSerializerProvider.java:480)
                at com.fasterxml.jackson.databind.ser.DefaultSerializerProvider.serializeValue(DefaultSerializerProvider.java:319)
                at com.fasterxml.jackson.databind.ObjectMapper._writeValueAndClose(ObjectMapper.java:4407)
                at com.fasterxml.jackson.databind.ObjectMapper.writeValueAsString(ObjectMapper.java:3661)
                at com.liam.learning.jackson.features.SerializationFeatureDemo.testFailOnUnwrappedTypeIdentifiers(SerializationFeatureDemo.java:141)
                at com.liam.learning.jackson.features.SerializationFeatureDemo.main(SerializationFeatureDemo.java:25)

                Process finished with exit code 0

------------------------------------------------

#### CLOSE_CLOSEABLE(false)
对于 `java.io.Closeable` 接口的实现类，控制在序列化的时候是否调用 `java.io.Closeable#close` 方法

------------------------------------------------

#### FLUSH_AFTER_WRITE_VALUE(true)
控制是否在调用的 `com.fasterxml.jackson.databind.ObjectMapper#writeValue(com.fasterxml.jackson.core.JsonGenerator, java.lang.Object)` 方法的时候是否调用 `com.fasterxml.jackson.core.JsonGenerator#flush` 方法将缓存内容刷新到输出流中；
禁止这个功能的主要原因是处于性能原因；对于网络连接，刷新可能会在最佳状态之前发送消息，并且在某些压缩流中会通过 flush() 方法去完成压缩块

------------------------------------------------

#### WRITE_DATES_AS_TIMESTAMPS(true)
序列化过程中，将 `java.util.Date` 类型或者 `java.time` 包中的数据类型的数据序列化成 long 类型的时间戳

***注意：使用该特性序列化 `java.time` 包中的类的时候需要向 ObjectMapper 注册 JavaTimeModule 才能生效***

1. 第一步引入依赖 `jackson-datatype-jsr310`
```xml
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
        <version>2.11.2</version>
    </dependency>
```

2. 注册模块 `com.fasterxml.jackson.datatype.jsr310.JavaTimeModule`

```java
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.registerModule(new JavaTimeModule());
    objectMapper.enable(SerializationFeature.WRITE_DURATIONS_AS_TIMESTAMPS);
    objectMapper.enable(SerializationFeature.INDENT_OUTPUT);
```

------------------------------------------------

#### WRITE_DATE_KEYS_AS_TIMESTAMPS(false),
控制是否在序列化过程中，将 `Map` 中 `java.util.Date` 类型的 key 序列化成 long 类型的时间戳

------------------------------------------------

#### WRITE_DATES_WITH_ZONE_ID(false)
控制是否在序列化 `java.time.ZonedDateTime` 类型数据点时候打印时区信息

***注意：使用该特性的时候需要向 ObjectMapper 注册 JavaTimeModule***

1. 第一步引入依赖 `jackson-datatype-jsr310`
```xml
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
        <version>2.11.2</version>
    </dependency>
```

2. 注册模块 `com.fasterxml.jackson.datatype.jsr310.JavaTimeModule`

```java
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.registerModule(new JavaTimeModule());
    objectMapper.enable(SerializationFeature.WRITE_DATES_WITH_ZONE_ID);
    objectMapper.enable(SerializationFeature.INDENT_OUTPUT);
```

------------------------------------------------

#### WRITE_DURATIONS_AS_TIMESTAMPS(true)

控制是否在序列化 `java.time.Duration` 类型数据是序列成数字还是字符串，ture 则使用数字，false 则使用字符串

***注意：使用该特性的时候需要向 ObjectMapper 注册 JavaTimeModule***

1. 第一步引入依赖 `jackson-datatype-jsr310`
```xml
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
        <version>2.11.2</version>
    </dependency>
```

2. 注册模块 `com.fasterxml.jackson.datatype.jsr310.JavaTimeModule`

```java
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.registerModule(new JavaTimeModule());
    objectMapper.enable(SerializationFeature.WRITE_DURATIONS_AS_TIMESTAMPS);
    objectMapper.enable(SerializationFeature.INDENT_OUTPUT);
```

------------------------------------------------

#### WRITE_CHAR_ARRAYS_AS_JSON_ARRAYS(false): 序列化json 的时候对于 `char[]` 为true 则解析成数组形式； 为false 则解析成一个字符串(String)
示例代码

```java
private static void testWriteCharArraysAsJsonArrays() {
       String t = "tester";
       char[] arr = t.toCharArray();

       ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.enable(SerializationFeature.WRITE_CHAR_ARRAYS_AS_JSON_ARRAYS);
       try {
           String s = objectMapper.writeValueAsString(arr);
           System.out.println(s);
       } catch (JsonProcessingException e) {
           e.printStackTrace();
       }
   }
```

运行结果如下：

    "tester"

去掉注释后，运行结果如下

    ["t","e","s","t","e","r"]

-------

#### WRITE_ENUMS_USING_TO_STRING(false): 序列化时， enable: 用枚举的 enum.toString() 表示枚举值 disable: 用枚举的 enum.name() 表示枚举值

```java
private static void testWriteEnumsUsingString() {
    ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.enable(SerializationFeature.WRITE_ENUMS_USING_TO_STRING);
    try {
        String s = objectMapper.writeValueAsString(CHILD.BOY);
        System.out.println(s);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
}

private enum CHILD {
    BOY, GIRL;

    @Override
    public String toString() {
        return "+" + this.name() + "+";
    }
}
```

代码运行结果：

    "BOY"

去掉注释后的代码运行结果：

    "+BOY+"


--------

#### WRITE_ENUMS_USING_INDEX(false): 序列化时，用枚举的 enum.index() 表示枚举值

```java
private static void testWriteEnumsUsingIndex() {
    ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.enable(SerializationFeature.WRITE_ENUMS_USING_INDEX);
    try {
        String s = objectMapper.writeValueAsString(CHILD.BOY);
        System.out.println(s);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
}

private enum CHILD {
    BOY, GIRL;
}
```

代码运行结果：

    "BOY"

取消注释后代码运行结果：

    0

--------

#### WRITE_NULL_MAP_VALUES(true): 对于 Map 中的 null 值 是否序列化

```java
public static void testWriteNullMapValues() {
    HashMap<String, Object> extMap = Maps.newHashMap();
    extMap.put("test1", null);
    extMap.put("test2", "not null");

    ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.disable(SerializationFeature.WRITE_NULL_MAP_VALUES);
    try {
        String s = objectMapper.writeValueAsString(extMap);
        System.out.println(s);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
}
```

代码运行结果：

    {"test1":null,"test2":"not null"}

去除注释后的运行结果：

    {"test2":"not null"}

--------


#### WRITE_EMPTY_JSON_ARRAYS(true): 对于空的 `Collection`、 `数组` ; 为 `true` 则序列化， 为 `false` 则跳过， 默认为 `true`

示例代码：

```java
private static void testWriteEmptyJsonArrays() {
    SimpleBean simpleBean = new SimpleBean();
    simpleBean.setCode(1);
    simpleBean.setName("WRITE_EMPTY_JSON_ARRAYS");
    simpleBean.setDesc(Collections.<String>emptyList());

    ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.disable(SerializationFeature.WRITE_EMPTY_JSON_ARRAYS);
    try {
        String s = objectMapper.writeValueAsString(simpleBean);
        System.out.println(s);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
}
```

运行结果如下：

    {"name":"WRITE_EMPTY_JSON_ARRAYS","code":1,"desc":[],"ext":null}


取消注释后，运行结果如下：

    {"name":"WRITE_EMPTY_JSON_ARRAYS","code":1,"ext":null}

-------

#### WRITE_SINGLE_ELEM_ARRAYS_UNWRAPPED(false): 序列化json时，对于只有单个元素的数组，不用中括号('[]')括起来

```java
public static void testWrietSingleElemArraysUnwrapped() {
    SimpleBean simpleBean = new SimpleBean();
    simpleBean.setCode(1);
    simpleBean.setName("TEST_WRAP_ROOT_VALUE");
    simpleBean.setDesc(Lists.newArrayList("test1"));

    ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.enable(SerializationFeature.WRITE_SINGLE_ELEM_ARRAYS_UNWRAPPED);
    try {
        String s = objectMapper.writeValueAsString(simpleBean);
        System.out.println(s);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
}
```

代码运行结果：

    {"name":"TEST_WRAP_ROOT_VALUE","code":1,"desc":["test1"]}

去掉注释之后的，运行结果

    {"name":"TEST_WRAP_ROOT_VALUE","code":1,"desc":"test1"}

效果一目了然！

-------

#### WRITE_BIGDECIMAL_AS_PLAIN(false): `@deprecated`
`@see`
`com.fasterxml.jackson.core.JsonGenerator.Feature#WRITE_BIGDECIMAL_AS_PLAIN`

-------

#### WRITE_DATE_TIMESTAMPS_AS_NANOSECONDS(true): 序列化json的时候把时间类型值序列化成纳秒的形式

    `注意` 只有最新版本(`jdk8` 中的 `Date/Time` )的时间类型支持本特性, `jdk8` 之前的 `java.util.Date`  和`joda-time` 都不支持！

-------

#### ORDER_MAP_ENTRIES_BY_KEYS(false): 序列化 `Map` 的时候， 为 `true` 则按照 `Map` 的 `key` 进行排序，否则不排序

示例代码：

```java
private static void testOrderMapEntriesByKeys() {
    ObjectMapper objectMapper = new ObjectMapper();
//        objectMapper.enable(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS);
    Map<String, String> data = ImmutableMap.of("1", "test1", "3", "test2", "2", "test3", "0", "test4");
    try {
        String dateStr = objectMapper.writeValueAsString(data);
        System.out.println(dateStr);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
}
```

运行结果：

    {"1":"test1","3":"test2","2":"test3","0":"test4"}

取消注释的运行结果：

    {"0":"test4","1":"test1","2":"test3","3":"test2"}


--------------------------------------

#### EAGER_SERIALIZER_FETCH(true): 序列化是是否应当预先抓取必要的 `JsonSerializer` ， 绝大多数情况下不应该关闭此特性

--------------------------------------

#### USE_EQUALITY_FOR_OBJECT_ID(false);

--------------------------------------


接下来是 反序列化时候的Feature ——

### DeserializationFeature

先来个最常用的：

####  `FAIL_ON_UNKNOWN_PROPERTIES` ： 反序列化遇到未知的字段的时候是否失败， 默认是 true 会失败
