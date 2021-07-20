# 好用的工具类收集

## 使用{0},{1}占位符的方式格式化字符串

java.text.MessageFormat.format\(\)

## Spring反射工具

org.springframework.util.ReflectionUtils

## Spring-Bean工具

org.springframework.beans.BeanUtils

## 获取Mybatis-Plus租户注解表名
```
List<TableInfo> tableInfos = TableInfoHelper.getTableInfos();
        TABLE_TENANT.addAll(ListUtils.forEachFunc(
                tableInfos,
                info -> AnnotationUtils.isAnnotationDeclaredLocally(TableTenant.class, info.getEntityType()),
                info -> info.getTableName()));
```                

## Spring注解工具类
org.springframework.core.annotation.AnnotationUtils#getAnnotation(java.lang.reflect.AnnotatedElement, java.lang.Class<A>)
