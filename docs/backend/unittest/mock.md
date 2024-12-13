<!-- omit from toc -->
# mockito技巧

## 向容器中注册mock对象

### 普通bean

```java
OssConnection ossConnection = mock(OssConnection.class);
OssConnection ossConnection2 = mock(OssConnection.class);
GenericApplicationContext context = new GenericApplicationContext();
context.getBeanFactory().registerSingleton("test", ossConnection);
context.getBeanFactory().registerSingleton("test2", ossConnection2);
context.refresh();
```

### primary bean

primary无法直接注册单例bean，只能通过BeanDefinition
```java
OssConnection ossConnection = mock(OssConnection.class);

GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
beanDefinition.setBeanClass(OssConnection.class);
beanDefinition.setInstanceSupplier(() -> ossConnection); // 使用 InstanceSupplier 提供 
beanDefinition.setPrimary(true); // 设置为 @Primary

GenericApplicationContext context = new GenericApplicationContext();
context.registerBeanDefinition(CONNECTION_BEAN_NAME, beanDefinition);
context.refresh();
```