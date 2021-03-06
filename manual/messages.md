# Log4j 2 API - Messages

## Messages

尽管Log4j 2提供了接收String参数和Object参数的Logger方法，但内部实现都会把它们关系到相应的Message对象中。应用程序可以随意地构建个性化的Message，传递给Logger。虽然看起来相对于直接传递message format和parameters性能更低，但测试表明，使用现代JVM，创建和销毁Message对象的成本是很小的。此外，当使用接收message format和parameters的方法时，只有配置的日志级别可以允许处理消息时，才会创建Message对象。

若存在这样的一个Map对象：{"Name"="John Doe", "Address"="123 Main St.", "Phone"="(999) 555-1212"}，还有一个User对象，具有getId方法，该方法返回"jdoe"。开发人员希望提到这样的一个日志：“User John Doe has logged in using id jdoe”。可以这样实现：

```java
logger.info("User {} has logged in using id {}", map.get("Name"), user.getId());
```

这种做法是最常用的，并没有问题，但是对象和所需的输出格式的复杂性使得用起来比较恶心麻烦。另一种做法就是，使用Message：

```java
logger.info(new LoggedInMessage(map, user));
```

这种做法中，格式化被委托给LoggedInMessage对象的getFormattedMessage方法。虽然这里创建了一个新对象，但在LoggedInMessage被允许输出之前，不会调用LoggedInMessage的对象上的任何方法。当对象的toString方法无法满足您所要求的消息格式时，特别有用。

使用Message的另一个好处就是，简化了Layout的使用。在其他日志框架中，Layout必须循环遍历每个参数，并根据实际的对象确定如何处理。使用Message，Layout可以把将格式化委托给Message，或者根据实际的Message类型执行格式化。

使用前面的Marker来识别SQL语句的示例，使用Message。首先，定义Message：

```java
public class SQLMessage implements Message {
  public enum SQLType {
      UPDATE,
      QUERY
  };

  private final SQLType type;
  private final String table;
  private final Map<String, String> cols;

  public SQLMessage(SQLType type, String table) {
      this(type, table, null);
  }

  public SQLMessage(SQLType type, String table, Map<String, String> cols) {
      this.type = type;
      this.table = table;
      this.cols = cols;
  }

  public String getFormattedMessage() {
      switch (type) {
          case UPDATE:
            return createUpdateString();
            break;
          case QUERY:
            return createQueryString();
            break;
          default;
      }
  }

  public String getMessageFormat() {
      return type + " " + table;
  }

  public Object getParameters() {
      return cols;
  }

  private String createUpdateString() {
    return formatCols(cols);
  }

  private String createQueryString() {
    return formatCols(cols);
  }

  private String formatCols(Map<String, String> cols) {
      StringBuilder sb = new StringBuilder();
      boolean first = true;
      for (Map.Entry<String, String> entry : cols.entrySet()) {
          if (!first) {
              sb.append(", ");
          }
          sb.append(entry.getKey()).append("=").append(entry.getValue());
          first = false;
      }
      return sb.toString();
  }
}
```

接下来，就在应用程序中使用message。

```java
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;
import java.util.Map;

public class MyApp {

    private Logger logger = LogManager.getLogger(MyApp.class.getName());
    private static final Marker SQL_MARKER = MarkerManager.getMarker("SQL");
    private static final Marker UPDATE_MARKER = MarkerManager.getMarker("SQL_UPDATE", SQL_MARKER);
    private static final Marker QUERY_MARKER = MarkerManager.getMarker("SQL_QUERY", SQL_MARKER);

    public String doQuery(String table) {
        logger.entry(param);

        logger.debug(QUERY_MARKER, new SQLMessage(SQLMessage.SQLType.QUERY, table));

        return logger.exit();
    }

    public String doUpdate(String table, Map<String, String> params) {
        logger.entry(param);

        logger.debug(UPDATE_MARKER, new SQLMessage(SQLMessage.SQLType.UPDATE, table, parmas);

        return logger.exit();
    }
}
```

对比之前的例子，本示例中，在doUpdate方法里的`logger.debug`方法调用时没有使用isDebugEnabled判断，因为在调用SQLMessage的过程中已执行了该判断（即达到了延迟计算`formatCols(cols)`方法的目的）。而且SQL列的所有格式都隐藏在SQLMessage中，而不必混合在业务逻辑里，这使业务逻辑更清晰。最后，在必要时，可以在Filter和/或Layout里对SQLMessage做某些特殊处理。

### FormattedMessage

首先判断传递给[FormattedMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/FormattedMessage.html)的消息模式（message pattern）参数，判断是否是`java.text.MessageFormat`模式。如果是，则使用[MessageFormatMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/MessageFormatMessage.html)对其格式化。如果不是，则接下来检查它是否包含对`String.format()`有效的格式标记。如果是，使用[StringFormattedMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/MessageFormatMessage.html)来格式化它。最后，如果都不匹配，那么使用[ParameterizedMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/ParameterizedMessage.html)来格式化它。

### LocalizedMessage

[LocalizedMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/LocalizedMessage.html)主要为了兼容Log4j 1.x。通常，本地化的最佳方法是让客户端UI里进行本地化处理。

LocalizedMessage包含一个ResourceBundle对象，让消息模式（message pattern）参数作为bundle里的key来获取本地化的消息模式。如果未指定任何bundle，LocalizedMessage将尝试查找相应的Logger名字的bundle。从bundle中检索的消息将使用FormattedMessage格式化。

### LoggerNameAwareMessage

[LoggerNameAwareMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/LoggerNameAwareMessage.html)是一个接口中，带setLoggerName方法。在事件构造期间将调用此方法，以便Message在格式化时可以使用到Logger的名字（其中LocalizedMessage实现了该接口）。

### MapMessage

[MapMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/MapMessage.html)包含key和value的映射。MapMessage实现FormattedMessage，可设置为输出成"XML"或"JSON"格式，在这种情况下，Map将格式化为XML或JSON。否则，Map将格式为“key1=value1 key2=value2 ...”。

### MessageFormatMessage

[MessageFormatMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/MessageFormatMessage.html)可处理符合`java.text.MessageFormat`[转换格式](http://docs.oracle.com/javase/6/docs/api/java/text/MessageFormat.html)的message。虽然它比ParameterizedMessage使用起来更灵活，但它的性能也更低，慢两倍左右。

### MultiformatMessage

[MultiformatMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/MultiformatMessage.html)接口包含一个getFormats方法和一个接受格式字符串数组的getFormattedMessage方法。getFormats方法可以被Layout调用，以提供Message支持哪些格式化信息。然后Layout使用这些格式化信息调用getFormattedMessage方法。如果Message不支持该格式，它将使用其默认格式简进行处理。比如，StructuredDataMessage，它接受格式为“XML”的字符串，这就决定了它将数据格式化为XML而不是RFC 5424格式。

### ObjectMessage

[ObjectMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/ObjectMessage.html)通过调用其toString方法来格式化对象。从Log4j 2.6开始，Layout调用`formatTo(StringBuilder)`方法来达到low-garbage或garbage-free的目的。

### ParameterizedMessage

[ParameterizedMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/ParameterizedMessage.html)处理包含“{}”的消息，其格式为表示可替换令牌和替换参数。

### ReusableObjectMessage

在garbage-free模式下，此Message用于将记录的对象传递到Layout和Appender上。功能等同于[ObjectMessage](./messages.md#ObjectMessage)。

### ReusableParameterizedMessage

在garbage-free模式下，此Message用于处理包含“{}”的消息，其格式为表示可替换令牌和替换参数。功能等同于[ParameterizedMessage](./messages.md#ParameterizedMessage)。

### ReusableSimpleMessage

在garbage-free模式下，此Message用于将记录的字符串和CharSequences传递到Layout和Appender上。功能上等同于[SimpleMessage](./messages.md#SimpleMessage)。

### SimpleMessage

[SimpleMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/SimpleMessage.html)包含不需要格式化的字符串或CharSequence。

### StringFormattedMessage

[StringFormattedMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/StringFormattedMessage.html)处理符合`java.lang.String.format()`的[转换格式](http://docs.oracle.com/javase/6/docs/api/java/util/Formatter.html#syntax)的message。虽然其比ParameterizedMessage更灵活，但它也慢5到10倍。

### StructuredDataMessage

[StructuredDataMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/StructuredDataMessage.html)允许应用程序向Map中添加项以及设置id，以允许根据RFC 5424将消息格式化为结构化数据元素。

### ThreadDumpMessage

[ThreadDumpMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/ThreadDumpMessage.html)将生成所有线程的堆栈跟踪信息。如果在Java 6+上运行，堆栈跟踪信息也包括hold住的任何锁。

### TimestampMessage

[TimestampMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/TimestampMessage.html)接口有一个getTimestamp方法，可以使其在事件构建期间被调用。将使用Message中的时间戳代替当前时间戳。
