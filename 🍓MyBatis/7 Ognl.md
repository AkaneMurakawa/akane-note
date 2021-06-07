# Ongl
OGNL是Object Graphic Navigation Language(对象图导航语言)的缩写，是Java中的一个开源的表达式语言，用于访问对象数据。在Mybatis中使用了Ognl来处理动态SQL，例如：
- if
- choose (when, otherwise)
- trim (where, set)
- foreach
```
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
</select>

<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <choose>
    <when test="title != null">
      AND title like #{title}
    </when>
    <when test="author != null and author.name != null">
      AND author_name like #{author.name}
    </when>
    <otherwise>
      AND featured = 1
    </otherwise>
  </choose>
</select>
```

## Ognl有趣的表达式
除了上面比较简单的表达式外，Ognl中还有其他比较有趣的特性
- e.method(args)调用对象方法
- e.property对象属性值
- e1[ e2 ]按索引取值，List,数组和Map
- @class@method(args)调用类的静态方法
- @class@field调用类的静态字段值

## @Ognl@isNotBlank
当我们需要对字符串判空的时候，如果每次都像下面这样写，其实是很麻烦的
```
<if test="title != null and tile !=''">
```
那么我们就可以封装成一个方法，例如新建一个类Ognl，那么在判断的时候就可以这样的形式：`@Ognl@isNotBlank(title)`
```
// package default 注意:这里设置成默认包名，不然在xml引用的时候，需要带上包名，例如@com.example.Ognl@isNotBlank(title)
import java.lang.reflect.Array;
import java.util.Collection;
import java.util.Map;

/**
 * Ognl工具类，方便mybatis xml使用
 */
@SuppressWarnings("ALL")
public class Ognl {

    public static boolean isNull(Object o) {
        return o == null;
    }

    public static boolean isNotNull(Object o) {
        return !isNull(o);
    }

    /**
     * 可以用于判断String,Map,Collection,Array是否为空
     */
    public static boolean isEmpty(Object o) throws IllegalArgumentException {
        if(o == null) {return true;}

        if(o instanceof String) {
            if(((String)o).length() == 0){
                return true;
            }
        } else if(o instanceof Collection) {
            if(((Collection)o).isEmpty()){
                return true;
            }
        } else if(o.getClass().isArray()) {
            if(Array.getLength(o) == 0){
                return true;
            }
        } else if(o instanceof Map) {
            if(((Map)o).isEmpty()){
                return true;
            }
        }else {
            return false;
        }
        return false;
    }

    public static boolean isNotEmpty(Object o) {
        return !isEmpty(o);
    }

    public static boolean isNumber(Object o) {
        if(o == null) {return false;}
        if(o instanceof Number) {
            return true;
        }
        if(o instanceof String) {
            String str = (String)o;
            if(str.length() == 0) {return false;}
            if(str.trim().length() == 0) {return false;}
            return org.apache.commons.lang.StringUtils.isNumeric(str);
        }
        return false;
    }

    public static boolean isNotBlank(Object o) {
        return !isBlank(o);
    }

    public static boolean isBlank(Object o) {
        if(o == null) {return true;}
        if(o instanceof String) {
            String str = (String)o;
            return isBlank(str);
        }
        return false;
    }

    public static boolean isBlank(String str) {
        int strLen;
        if (str != null && (strLen = str.length()) != 0) {
            for(int i = 0; i < strLen; ++i) {
                if (!Character.isWhitespace(str.charAt(i))) {
                    return false;
                }
            }

            return true;
        } else {
            return true;
        }
    }

}

```

