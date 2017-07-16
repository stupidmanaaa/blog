# mybatis 源码学习之getMapper过程分析
## 一、 简介
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这篇文章分析mybatis3.2.8中getMapper的过程，分为两个部分，一是剖析mybatis初始化的过程（这里采用加载xml配置文件的方式）</p>

## 二、 mybatis初始化

### 1. 配置文件
首先我们看下这里使用的配置文件，在mappers标签中利用class的形式配置（如果不清楚的话，可以[戳下这](http://www.mybatis.org/mybatis-3/zh/configuration.html#mappers)）。
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"></transactionManager>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"></property>
                <property name="url" value="jdbc:mysql://localhost:3306/test2"></property>
                <property name="username" value="root"></property>
                <property name="password" value="root"></property>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper class="com.hwb.study.smybatis.mapper.EmployeesMapper"></mapper>
    </mappers>
</configuration> 
```

### 2. 初始化
这里利用以上配置文件进行初始化
```
public static SqlSessionFactory buildSqlSessionFactory() throws IOException {
        // 配置文件
        String resource = "mybatis-config.xml";
        // 讲配置文件转化为输入流
        InputStream inputStream = Resources.getResourceAsStream(resource);
        // 利用SqlSessionFactoryBuilder进行初始化
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        return sqlSessionFactory;
    }
```

简单介绍SqlSessionFactoryBuilder,来看一下它的源码
```
public class SqlSessionFactoryBuilder {
	public SqlSessionFactory build(Reader reader) {
	    return build(reader, null, null);
	}
	
	...
	
	public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
	    try {
	      // 真正生成sqlsessionfactory
	      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
	      return build(parser.parse());
	    } catch (Exception e) {
	      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
	    } finally {
	      ErrorContext.instance().reset();
	      try {
		reader.close();
	      } catch (IOException e) {
		// Intentionally ignore. Prefer previous error.
	      }
	    }
    }
    
    ...
}
```
SqlSessionFactoryBuilder 采用builder设计模式，它的作用就是为了帮助生成sqlsessionfactory，关键还是利用XMLConfigBuilder类来真正解析Xml配置文件，返回sqlsessionfactory,这里的关键代码就是
```
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      return build(parser.parse());
```
接下来我们可以看下XMLConfigBuilder的源码
```
public class XMLConfigBuilder extends BaseBuilder {
	...
	public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
	    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
	}
	...
	private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
	    super(new Configuration());
	    ErrorContext.instance().resource("SQL Mapper Configuration");
	    this.configuration.setVariables(props);
	    this.parsed = false;
	    this.environment = environment;
	    this.parser = parser;
	}
	...
	public Configuration parse() {
	    if (parsed) {
	      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
	    }
	    parsed = true;
	    parseConfiguration(parser.evalNode("/configuration"));
	    return configuration;
	  }

	private void parseConfiguration(XNode root) {
	    try {
	      Properties settings = settingsAsPropertiess(root.evalNode("settings"));
	      //issue #117 read properties first
	      propertiesElement(root.evalNode("properties"));
	      loadCustomVfs(settings);
	      typeAliasesElement(root.evalNode("typeAliases"));
	      pluginElement(root.evalNode("plugins"));
	      objectFactoryElement(root.evalNode("objectFactory"));
	      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
	      reflectorFactoryElement(root.evalNode("reflectorFactory"));
	      settingsElement(settings);
	      // read it after objectFactory and objectWrapperFactory issue #631
	      environmentsElement(root.evalNode("environments"));
	      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
	      typeHandlerElement(root.evalNode("typeHandlers"));
	      mapperElement(root.evalNode("mappers"));
	    } catch (Exception e) {
	      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
	    }
	}
}
```
其中XPathParser工具类作用是利用javax.xml中工具类对xml配置文件进行解析（这个javax.xml的类大家可以自行百度）。在完成XMLConfigBuilder的实例化之后，<font color='red'>关键来了，在XMLConfigBuilder的parse的方法中,在这个方法中解析xml中对mapper的配置。</font>
```
mapperElement(root.evalNode("mappers"));
```
接下来我们看这个mapperElement()方法；

```
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
      }
   }
}
```
在这段代码中，最后一个else if(resource = = null && url = = null && mapperClass != null),这里就是判断mapper是以那种方式定义，我们的配置文件中以class(interface)的方式配置，所以会进入到这个代码块中。这里的重点就是 configuration.addMapper(mapperInterface); 将这个配置加载如configuration中的mapperRegistry。至此基本的Mapper解析算是完成了。接下来我们看看sqlsession是如何获取已经注册好的mapper的。

###三、getMapper 剖析
从这里开始分析sqlsession.getMapper的过程。经过第二步中的初始化之后，我们可以得到SqlSessionFactory,然后利用其创建一个sqlsession,利用这个sqlsession获得其mapper的代理类。代码如下
```
/**
     * 研究mybatis mapper 过程
     *
     * @throws IOException
     */
    public static void testGetMapper() throws IOException {
        SqlSessionFactory factory = buildSqlSessionFactory();
        SqlSession sqlSession = factory.openSession();

        // get mapper
        EmployeesMapper mapper = sqlSession.getMapper(EmployeesMapper.class);

        System.out.println(mapper.getClass());

        // bulid param
        Map<String,Object> paramMap = new HashMap<String, Object>();
        paramMap.put("employee_id", 100);

        // execute query
        List<Map<String, Object>> list = mapper.selectByPrimary(paramMap);

        // print result
        System.out.println(list);

        sqlSession.close();
    }
```
 我们进入到sqlsession.getmapper的源码中，能够发现其实它只是调用了其内部的configuration.getMapper()方法。
```public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }```
  回顾一下第二步中的最后，mapperElement解析方法中对class类型的mapper的处理，
```
  if (resource == null && url == null && mapperClass != null) {
	Class<?> mapperInterface = Resources.classForName(mapperClass);
	configuration.addMapper(mapperInterface);
  } 
```
  看到这里大家应该大致明白了这个mybatis中get/add mapper的流程了吧，但是configuration 内部又是怎么处理管理这个mapper的呢。
  
### 最后

关于configuration 中管理mapper的MapperRegistry流程，之后我会再写一篇博客，继续学习！！


希望大家能一起交流学习。联系我997239778(qq)。