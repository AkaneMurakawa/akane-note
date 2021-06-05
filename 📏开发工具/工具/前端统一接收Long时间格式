# 前端统一接收Long时间格式

```java
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.core.convert.converter.Converter;
import org.springframework.format.FormatterRegistry;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import com.alibaba.fastjson.serializer.SerializeConfig;
import com.alibaba.fastjson.serializer.SerializerFeature;
import com.alibaba.fastjson.support.config.FastJsonConfig;
import com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter4;

/**
 * 覆盖MVCAdapter时间格式转换器，保证web页面提交的时间参数都为Long类型，否则抛出异常!
 */
@Configuration
@Order(2000)
public class WebMVCAdapter implements WebMvcConfigurer {

	@Override
	public void addFormatters(FormatterRegistry registry) {
		registry.addConverter(new DateConvertor());
	}
	
	@Override
	public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
		converters.add(fastsonHttpMessageConverter());
	}	

	@SuppressWarnings("rawtypes")
	public HttpMessageConverter fastsonHttpMessageConverter(){
		FastJsonHttpMessageConverter4 fastConverter = new FastJsonHttpMessageConverter4();
		FastJsonConfig fastJsonConfig = new FastJsonConfig();
		fastJsonConfig.setSerializerFeatures(SerializerFeature.WriteMapNullValue,SerializerFeature.DisableCircularReferenceDetect);

		fastJsonConfig.setSerializeConfig(serializeConfig());
		List<MediaType> supportedMediaTypes = new ArrayList<>();
		supportedMediaTypes.add(MediaType.APPLICATION_JSON);
		supportedMediaTypes.add(MediaType.APPLICATION_JSON_UTF8);
		supportedMediaTypes.add(MediaType.APPLICATION_ATOM_XML);
		supportedMediaTypes.add(MediaType.APPLICATION_FORM_URLENCODED);
		supportedMediaTypes.add(MediaType.APPLICATION_OCTET_STREAM);
		supportedMediaTypes.add(MediaType.APPLICATION_PDF);
		supportedMediaTypes.add(MediaType.APPLICATION_RSS_XML);
		supportedMediaTypes.add(MediaType.APPLICATION_XHTML_XML);
		supportedMediaTypes.add(MediaType.APPLICATION_XML);
		supportedMediaTypes.add(MediaType.IMAGE_GIF);
		supportedMediaTypes.add(MediaType.IMAGE_JPEG);
		supportedMediaTypes.add(MediaType.IMAGE_PNG);
		supportedMediaTypes.add(MediaType.TEXT_EVENT_STREAM);
		supportedMediaTypes.add(MediaType.TEXT_HTML);
		supportedMediaTypes.add(MediaType.TEXT_MARKDOWN);
		supportedMediaTypes.add(MediaType.TEXT_XML);
		fastConverter.setSupportedMediaTypes(supportedMediaTypes);
		fastConverter.setFastJsonConfig(fastJsonConfig);
		return fastConverter;
	}

	/**
	 * 配置序列化文件
	 * @return
	 */
	private SerializeConfig serializeConfig(){
		SerializeConfig serializeConfig = SerializeConfig.globalInstance;
		serializeConfig.put(Long.class, LongToStringSerializer.instance);
		return serializeConfig;
	}

	/**
	 * 设置强制使用Long类型的时间格式，非Long格式则抛出异常，本设置应用于单参数请求
	 */
	class DateConvertor implements Converter<String, Date>{
		@Override
		public Date convert(String source) {
			Long dataLong = null;
			try {
				if(source!=null && !"".equals(source)) {
					dataLong = Long.parseLong(source);
					return new Date(dataLong);
				}
			} catch (NumberFormatException e) {
				throw new BusinessException("时间转换异常: Date参数必须为Long类型!");
			}
			return null;
		}
	}
}
```

LongToStringSerializer
```java

import com.alibaba.fastjson.serializer.JSONSerializer;
import com.alibaba.fastjson.serializer.ObjectSerializer;
import com.alibaba.fastjson.serializer.SerializeWriter;
import java.io.IOException;
import java.lang.reflect.Type;

public class LongToStringSerializer implements ObjectSerializer {

    public static final LongToStringSerializer instance = new LongToStringSerializer();

    private String converseFileName="tid";


    /**
     * fastjson invokes this call-back method during serialization when it encounters a field of the
     * specified type.
     *
     * @param serializer
     * @param object     src the object that needs to be converted to Json.
     * @param fieldName  parent object field name
     * @param fieldType  parent object field type
     * @param features   parent object field serializer features
     * @throws IOException
     */
    @Override
    public void write(JSONSerializer serializer, Object object, Object fieldName, Type fieldType, int features) throws IOException {
        SerializeWriter out = serializer.out;

        if (object == null) {
            out.writeNull();
            return;
        }
        if(fieldName==converseFileName){
            String strVal = object.toString();
            out.writeString(strVal);
        }else {
            out.writeLong((Long) object);
        }

    }
}
```
