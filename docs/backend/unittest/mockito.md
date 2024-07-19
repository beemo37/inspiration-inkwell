<!-- omit from toc -->
# mockito参考手册

- [mock](#mock)
- [spy](#spy)
- [使用注解](#使用注解)
- [stub](#stub)
  - [when](#when)
  - [thenXXX](#thenxxx)
  - [doXXX](#doxxx)
  - [then和do的区别](#then和do的区别)
  - [默认返回值](#默认返回值)
- [ArgumentCaptor](#argumentcaptor)
- [场景](#场景)
  - [打桩：返回值为空，想做一些事情](#打桩返回值为空想做一些事情)
  - [打桩：返回值为空，什么也不做](#打桩返回值为空什么也不做)
  - [打桩：TransactionTemplate#execute](#打桩transactiontemplateexecute)
  - [打桩：TransactionTemplate#executeWithoutResult](#打桩transactiontemplateexecutewithoutresult)
  - [打桩：返回值是Redisson的RFuture](#打桩返回值是redisson的rfuture)
- [资料](#资料)


```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

## mock

mock是指模拟一个对象，调用这个对象的方法不会有之前的行为，我们可以按照我们的意愿去控制返回值（调用成功）、或抛出异常（调用失败）来模拟测试各种情景。

```java
// 模拟一个List
List<String> mockList = Mockito.mock();
```

实际使用时一般静态导入，简化写法
```java
import static org.mockito.Mockito.*;

...

List<String> mockList = mock();

```

## spy

TODO

## 使用注解

除了手动创建mock、spy对象，跟推荐使用注解，这会使得代码更简洁，更容读。

首先就是要启用mockito注解，共有三种方式：

 - 使用`@ExtendWith(MockitoExtension.class)`注解
 首先需要引入依赖
 ```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
 ```
 ```java
@ExtendWith(MockitoExtension.class)
class MyTest {
    ...
}
 ```

 - 使用`MockitoAnnotations.openMocks(this)`

```java
class Test {

    @BeforeEach
    void setup() {
        MockitoAnnotations.openMocks(this);
    } 
}
```

- 使用`MockitoRule`

```java
class Test {

    // 必须为public
    @Rule
    public MockitoRule role = MockitoJUnit.rule();
    ...
}
```

在使用上还是有点区别的，如第一种默认使用的是严格规则，即你在方法里进行打桩，但是实际上却没使用，执行的时候就会抛出异常


## stub

stub意为桩或打桩，即控制mock或spy对象的行为，如返回值，或抛出异常等。

打桩在方法之间是互相隔离的，一个方法对全局mock对象的打桩不会影响另一个方法。

### when

使用`Mockito.when()`方法定义需要打桩的方法以及方法入参，是打桩的核心，有两种方式，when在前，以及when在后

- when在前

```java
List<String> mockList = mock();

// 打桩
// 当调用mockList.get(0)时，做某事
when(mockList.get(0)).thenXXX
// 当调用mockList.get(1)时，做某事
when(mockList.get(1)).thenXXX
```

注意：这里`when`的参数是需要打桩的<span style="color:red">**方法调用**</span>。

- when在后

```java
// 打桩
// 当调用mockList.get(0)时，做某事
doXXX.when(mockList).get(0);
```

注意：这里`when`的参数是需要打桩的<span style="color:red">**对象**</span>。

---

两者在某些场景下使用是完全一致的，有些情况下有所区别，详见[then和do的区别](#then和do的区别)小节。

在实际使用中一般不会针对特定方法的特定入参打桩，而是任意值都可以，这样可以更加通用以及灵活，此时就需要使用any家族相关匹配器


```java
List<String> mockList = mock();

// 打桩
// 当调用mockList.get()时，做某事
when(mockList.get(anyInt())).xxx;
doXXX.when(mockList).get(anyInt());
```
类似的方法有`anyString()`, `anyBoolean()`, `anyList`，`any(Class)`等，详见`ArgumentMatchers`提供的静态方法。

### thenXXX

thenXXX用来控制打桩的行为，放在`when`之后，具体包含：
```java
// 自定义返回值（简单返回值）
when(mockList.get(anyInt())).thenReturn("abc");
// 自定义返回值（返回值需要通过某种复杂计算获得，或还需处理其他逻辑）
when(mockList.get(anyInt())).thenAnswer(invocation -> {
    int index = invocation.getArgument(0, int.class);
    return "element" + index;
});
// 抛出异常
when(mockList.get(anyInt())).thenThrow(IllegalArgumentException.class);
// 调用真实方法
when(mockList.get(anyInt())).thenCallRealMethod();
```

也可以控制多次行为
```java
when(mockList.get(anyInt()))
    // 第一次调用返回first
    .thenReturn("first")
    // 从第二次调用开始返回rest
    .thenReturn("rest");
```

### doXXX

doXXX方法也是用来控制打桩的行为，放在`when`之前，并且`when`入参不再是方法调用，而是传mock或spy对象，之后再指定具体打桩的方法：

```java
// 自定义返回值（简单返回值）
// 注意when的写法
doReturn("abc").when(mockList).get(anyInt());
// 自定义返回值（返回值需要通过某种复杂计算获得）
doAnswer(invocation -> {
      int index = invocation.getArgument(0, int.class);
      return "element" + index;
    }).when(mockList).get(anyInt());
// 抛出异常
doThrow(IllegalStateException.class).when(mockList).get(anyInt());
// 调用真实方法
doCallRealMethod().when(mockList).get(anyInt());

```

也可使用doXXX处理无返回值（void）的方法
```java
// 什么也不做
doNothing().when(mockList).clear();
// 想做某些事
doAnswer(invocation -> {
    counter = 0;
    return null;
}).when(mockList).clear();
// 
```

### then和do的区别

大部分场景两者效果是相同的，但是某些时候两者有区别：

- 当方法返回值为`void`时，只能使用`doXXX`的方式。
- 当打桩spy对象时，调用`thenXXX`会优先执行一次方法，当方法的入参对参数进行非空列表校验时，参数如果使用anyList等就会抛出异常。此时需要使用`doXXX`

  ```java
  class Test {
    public String joinElement(List<String> list) {
      Assert.notEmpty(list, "list");
      ...
    }
  }
  ...

  Test spy = spy();
  // 这种写法会抛出异常，anyList返回的是空列表，先会执行一遍
  joinElement.when(spy.joinElement(anyList()).thenReturn("abc");
  // 这种写法不会抛出异常，不会执行joinElement
  doReturn("abc).when(spy).joinElement(anyList);
  ```

### 默认返回值

默认情况下，方法返回值为：

* 自定义对象：null
* 基本类型以及包装类型：基本类型的默认值，如int就为0
* map为空的map
* 集合为空的集合
* Optional为Optional.empty()
* ...


## ArgumentCaptor

使用ArgumentCaptor可以获取mock或spy对象执行打桩方法时传递的实际参数

```java
List<String> mockList = mock();

ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
mockList.add("abc");
verify(mockList).add(captor.capture());
// abc
System.out.println(captor.getValue());
```

使用注解的方式：

```java
@Captor
private ArgumentCaptor<String> captor;

...

mockList.add("abc");
verify(mockList).add(captor.capture());
// abc
System.out.println(captor.getValue());

```


ps：从5.3.0版本后，你可以在verify中使用参数验证来代替ArgumentCaptor，看个人喜好

```java
mockList.add("abc");
verify(mockList).add(assertArg(arg -> assertEquals("abc", arg)));
```





## 场景

### 打桩：返回值为空，想做一些事情

```java
doAnswer(invocation -> {
      SmsAccessKeyPo accessKeyPo = invocation.getArgumentAt(0, SmsAccessKeyPo.class);
      ACCESS_KEYS.put(accessKeyPo.getAccessKey(), accessKeyPo);
      return null;
    })
	.when(accessKeyRepository)
    .saveNew(any(SmsAccessKeyPo.class));
```
### 打桩：返回值为空，什么也不做

```java
doNothing()
    .when(externalSourceService)
    .saveNew(any(ExternalSourceBo.class));
```

### 打桩：TransactionTemplate#execute

```java
when(transactionTemplate.execute(any(TransactionCallback.class)))
    .thenAnswer(invocation -> {
        TransactionCallback<?> callback = invocation.getArgument(0);
        return callback.doInTransaction(newTransactionStatus());
});
```

### 打桩：TransactionTemplate#executeWithoutResult

```java
doAnswer(invocation -> {
    Consumer<TransactionStatus> consumer = invocation.getArgument(0);
    consumer.accept(null);
    return null;
    }).when(transactionTemplate).executeWithoutResult(any(Consumer.class));
```

### 打桩：返回值是Redisson的RFuture

```java
@Mock
private RLock redissonLock;

when(redissonLock.unlockAsync(anyLong()))
    .thenReturn(RedissonPromise.newSucceededFuture(null));
```
## 资料

[Baeldung Mockito Guide](Mockito+Guide.pdf)
