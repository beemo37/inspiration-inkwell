<!-- omit from toc -->
# SnakeYaml

- [反序列化](#反序列化)

## 反序列化

假设以下比较复杂的规则引擎配置文件：

```yaml
platforms:
  jdl:
    rules:
      - condition: "xxx"
        action: throw
        message: 订单取消中
      - condition: "xxx"
        action: throw
        message: 订单取消失败
  qimen:
    rules:
      - condition: "xxx"
        action: throw
        message: "订单取消失败:{message}"

  flux:
    rules:
      - condition: "xxx"
        action: throw
        message: "取消订单异常:结果代码:{returnCode},消息:{returnDesc}"
```

并且定义以下的对象模型

```java

/**
 * 订单取消平台配置
 */
@Getter
@Setter
public class BillCancelPlatform {

  /** key为平台名称，value为取消规则。 */
  private Map<String, BillCancelRule> platforms = new CaseInsensitiveMap<>();


  /**
   * 订单取消规则
   */
  @Getter
  @Setter
  public static class BillCancelRule {

    private List<Rule> rules = new ArrayList<>();

  }

  /**
   * 规则条目。
   */
  @Getter
  @Setter
  public static class Rule {

    private String condition;
    private String action;
    private String message;
  }
}

```

那么你只需要按照如下写法即可反序列化到我们想要的模型对象中

```java
public void deserialize(String yamlPath) {
  Constructor constructor = new Constructor(BillCancelPlatform.class, new LoaderOptions());

  try (InputStream inputStream = getClass().getResourceAsStream(yamlPath)) {
    if (inputStream == null) {
      LOGGER.info("No BillCancel rule specified.");
      return;
    }

    platformRules = new Yaml(constructor).loadAs(inputStream, BillCancelPlatform.class);
    assert platformRules != null;

    LOGGER.info("BillCancel rule loading success.");
    platformRules.getPlatforms().forEach((k, v) -> LOGGER.info("{} -> {} items.", k, 
      v.getRules() == null ? 0 : v.getRules().size()));
  } catch (Exception e) {
    LOGGER.error("Fail to load BillCancel rules.", e);

    throw new IllegalStateException(e);
  }
}
```