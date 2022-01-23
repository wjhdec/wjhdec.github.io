# Jackson自动对类型格式化


　　现在很多框架都是自动配置，方便了许多，但是也因此带来了麻烦。例如一些特殊的情况下，自动完成的不能满足需求。Jackson也有类似的问题，这时就需要自定义格式化类型。

<!-- more -->

　　实际操作中有这样一个问题，postgresql较新的版本支持json格式，当数据库中有jsonb或json字段时，用mybatis等框架读取出来的并不是json字符串，而是PGData的格式，这时就需要进行数据的格式化了。

　　首先模拟出一个PGData

``` java
/**
 * 模拟Postgresql中传入的jsonb
 */
public class PGData {
    private String type;
    private Object value;

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public Object getValue() {
        return value;
    }

    public void setValue(Object value) {
        this.value = value;
    }
}
```

简单演示，方法和测试就都写在一起了。

``` java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.ser.BeanPropertyWriter;
import com.fasterxml.jackson.databind.ser.BeanSerializerModifier;

import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.List;

public class JacksonModifier {
    // 添加时间及pgdata修改内容
    private static final ObjectMapper objectMapper = new ObjectMapper();

    // 不添加任何内容，做对比用
    private static final ObjectMapper objectMapper1 = new ObjectMapper();

    static {
        // 添加修改
        objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd hh:mm:ss"))
                .setSerializerFactory(objectMapper.getSerializerFactory().withSerializerModifier(getJsobModifier()));
    }

    /**
     * 获取 修改类的实例
     * @return
     */
    private static BeanSerializerModifier getJsobModifier() {
        return new BeanSerializerModifier() {
            @Override
            public List<BeanPropertyWriter> changeProperties(SerializationConfig config, BeanDescription beanDesc, List<BeanPropertyWriter> beanProperties) {
                for (BeanPropertyWriter w : beanProperties) {
                    if (w.getType().getRawClass().equals(PGData.class)) {
                        w.assignSerializer(new JsonSerializer<Object>() {
                            @Override
                            public void serialize(Object value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
                                gen.writeObject(((PGData) value).getValue());
                            }
                        });
                    }
                }
                return beanProperties;
            }
        };
    }

    public static void main(String[] args) {
        try {
            TestData td = new TestData();
            td.setDt(new Date());
            PGData pgdt = new PGData();
            pgdt.setType("jsonb");
            pgdt.setValue("value");
            td.setPgData(pgdt);
            System.out.println("修改后");
            System.out.println(objectMapper.writeValueAsString(td));
            System.out.println("原始输出");
            System.out.println(objectMapper1.writeValueAsString(td));
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
    }
}
```

　　最近写scala有点多，成习惯了，getJsobModifier方法直接串下来了……
　　getJsobModifier方法生成转换的实例会把type去掉，只留下value，作为最终结果。

　　测试输出：

``` log
修改后
{"dt":"2018-01-13 08:04:07","pgData":"value"}
原始输出
{"dt":1515845047360,"pgData":{"type":"jsonb","value":"value"}}
```

　　可以看到时间格式转换和PGData类型的数据都有了改变
