---
layout: post
title: mybaits-generator生成器
date: 2018-06-20 11:48:00
categories:
    - 日常记录
tags:
    - mybaits-generator
    - mybatis
---
# mybaits-generator生成器

## 说明

使用java类+yml配置+JUnit搭建mybatis生成器项目, 能让其他项目快速接入生成mybatis使用的类和文件。

1. java类用于代替传统的xml配置, 简化配置成本和上手成本。
2. 配置文件使用YAML语法, 相比xml能大大增强阅读性。
3. 使用JUnit启动运行, 能有效隔离项目, 区分业务和工具, 降低对项目的影响。

## 需要引入的jar包和版本说明

| jar包                           | 版本    | 作用                                                         | 是否必要                         |
| ------------------------------- | ------- | ------------------------------------------------------------ | -------------------------------- |
| lombok                          | 1.16.20 | 自动生成getter/setter方法, 加强源码阅读性                    | 可选                             |
| fastjson                        | 1.2.47  | JSON序列化框架                                               | 必要, 也可以用其他序列化框架代替 |
| snakeyaml                       | 1.17    | 用于读取解析YAML文件配置                                     | 必要                             |
| mysql-connector-java            | 5.1.46  | 用于连接数据库                                               | 必要                             |
| ⭐mybatis-generator-core         | 1.3.6   | 生成器的核心包, 建议升级到1.3.6以上版本,这个版本支持覆盖文件 | 必要                             |
| ⭐mybatis-generator-maven-plugin | 1.3.6   | 生成器maven插件(内包含了生成器核心包)                        | 必要                             |
| junit                           | 4.12    | 单例测试, 用于启动                                           | 必要                             |

## 项目结构

### 生成文件

1. entity.java
2. mapper.java
3. mapper.xml

### 目录结构

```text

生成文件目录
src
    main
        java
            entity          -------- 生成的实体
            mapper          -------- 生成的mapper
            po              -------- 自定义po(用于自定义sql映射的实体)
            mapperExt       -------- 自定义mapper
        resources
            mapperXml       -------- 自动生成sql
            mapperXmlExt    -------- 自定义sql

运行目录
    test
        java
            generator   -------- 存放生成器的启动类(单例测试)
        resource    --------- 存放生成器YMAL配置文件`generator.yml`

```

## 配置和类

### 创建配置文件`generator.yml`

```yaml
#需要生成的表名
tableNames :
    - user_table

#------------------------- 数据库连接 ----------------------------------

url : jdbc:mysql://host:port/dbname
user : username
password : password

#------------------------- 个人路径(改这里就可以) ----------------------------------

#项目所在的地址路径(默认根据target/自动获取所在根目录, 目录下mybatis-generator文件夹, 也可自定义设置绝对路径)
#projectPath : D:\Project\
#jar包的绝对路径(默认根据driverClass包名自动获取, 也可自定义设置绝对路径)
#classPath : D:\mysql\mysql-connector-java-5.1.40.jar

# ------------------------- 项目配置 ----------------------------------

driverClass : com.mysql.jdbc.Driver

#实体包名和位置
javaModelGeneratorPackage : com.user.entity
javaModelGeneratorProject : src\main\java

#mapper包名和位置
javaClientGeneratorPackage : com.user.mapper
javaClientGeneratorProject : src\main\java

#mapperXml位置
sqlMapGeneratorPackage : mapperXml
sqlMapGeneratorProject : src\main\resources

```

### java类

1. 配置覆盖参数

生成规则默认使用IntrospectedTableMyBatis3Impl, 但是没有isMergeable(覆盖)的可设置方法, 改一下
注意:1.3.6之前的版本都不能设置覆盖, 只能追加(可能我不会), 1.3.6版本开放了这个参数
```java
public class MyIntrospectedTableMyBatis3Impl extends IntrospectedTableMyBatis3Impl {

    @Override
    public List<GeneratedXmlFile> getGeneratedXmlFiles() {
        List<GeneratedXmlFile> answer = new ArrayList<>();

        if (xmlMapperGenerator != null) {
            Document document = xmlMapperGenerator.getDocument();
            GeneratedXmlFile gxf = new GeneratedXmlFile(document,
                    getMyBatis3XmlMapperFileName(), getMyBatis3XmlMapperPackage(),
                    context.getSqlMapGeneratorConfiguration().getTargetProject(),
                    false, context.getXmlFormatter());
            if (context.getPlugins().sqlMapGenerated(gxf, this)) {
                answer.add(gxf);
            }
        }

        return answer;
    }
}
```

1. 配置信息,主要获取yml配置文件

```java
@Data
@Accessors(chain = true)
public class MyGeneratorConfig {

    private  String[] tableNames;
    private  String classPath;
    private  String driverClass;
    private  String url;
    private  String user;
    private  String password;
    private  String schema;
    /**
     * 绝对路径项目根目录
     */
    private  String projectPath;
    private  String javaModelGeneratorPackage;
    private  String javaModelGeneratorProject;
    private  String javaClientGeneratorPackage;
    private  String javaClientGeneratorProject;
    private  String sqlMapGeneratorPackage;
    private  String sqlMapGeneratorProject;

    MyGeneratorConfig() {

    }

    /**
     * 配置文件"generator.yml"
     * @param ymlName
     * @return
     */
    public MyGeneratorConfig getConfig(String ymlName) {

        Map<String, Object> map = new HashMap<>();
        Yaml yaml = new Yaml();
        InputStream is = this.getClass().getClassLoader().getResourceAsStream(ymlName);
        if (is != null) {
            Object obj = yaml.load(is);
            if (obj != null) {
                map = (Map<String, Object>) obj;
            }
        }

        MyGeneratorConfig myGeneratorConfig = JSON.parseObject(JSON.toJSONString(map), this.getClass());

        if (StringUtils.isBlank(myGeneratorConfig.getClassPath())) {
            // 获取驱动包路径
            try {
                String driverClassPath = Class.forName(myGeneratorConfig.getDriverClass())
                        .getProtectionDomain()
                        .getCodeSource()
                        .getLocation()
                        .getPath()
                        .replace("/", "\\").substring(1);
                myGeneratorConfig.setClassPath(driverClassPath);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        }

        if (StringUtils.isBlank(myGeneratorConfig.getProjectPath())) {
            // 默认获取项目路径
            String[] projectPaths = MyGeneratorConfig.class.getResource("/").getPath().split("/target");
            projectPath = projectPaths[0].replace("/", "\\").substring(1) + "\\" + "mybatis-generator" + "\\";
            // 处理中文路劲
            try {
                projectPath = java.net.URLDecoder.decode(projectPath,"utf-8");
            } catch (UnsupportedEncodingException e) {
                e.printStackTrace();
            }
            myGeneratorConfig.setProjectPath(projectPath);
        }

        projectPath = myGeneratorConfig.getProjectPath();
        myGeneratorConfig.setJavaModelGeneratorProject(projectPath + myGeneratorConfig.getJavaModelGeneratorProject());
        myGeneratorConfig.setJavaClientGeneratorProject(projectPath + myGeneratorConfig.getJavaClientGeneratorProject());
        myGeneratorConfig.setSqlMapGeneratorProject(projectPath + myGeneratorConfig.getSqlMapGeneratorProject());

        // 创建文件夹
        new File(myGeneratorConfig.getProjectPath()).mkdirs();
        new File(myGeneratorConfig.getJavaModelGeneratorProject()).mkdirs();
        new File(myGeneratorConfig.getJavaClientGeneratorProject()).mkdirs();
        new File(myGeneratorConfig.getSqlMapGeneratorProject()).mkdirs();

        System.out.println("entity path:" + myGeneratorConfig.getJavaModelGeneratorProject());
        System.out.println("mapperJava path:" + myGeneratorConfig.getJavaClientGeneratorProject());
        System.out.println("mapperXml path:" + myGeneratorConfig.getSqlMapGeneratorProject());

        return myGeneratorConfig;
    }
}
```

1. 构建生成配置

```java
public class MybatisGeneratorMain {

    private Log logger = LogFactory.getLog(getClass());

    private Context context = new Context(ModelType.FLAT);
    private List<String> warnings = new ArrayList<>();
    /**
     * 获取配置
     */
    private MyGeneratorConfig myGeneratorConfig = null;

    public MybatisGeneratorMain(String ymlPath) {

        // 设置配置文件
        this.myGeneratorConfig = new MyGeneratorConfig().getConfig(ymlPath);
        // 生成
        generator();
    }

    /**
     * 生成代码主方法
     */
    private void generator() {

        if (StringUtils.isNotBlank(myGeneratorConfig.getSchema())) {
            myGeneratorConfig.setUrl(myGeneratorConfig.getUrl() + "/" + myGeneratorConfig.getSchema());
        }

        context.setId("prod");
//        context.setTargetRuntime("MyBatis3");
        context.setTargetRuntime(MyIntrospectedTableMyBatis3Impl.class.getName());

        // ---------- 配置信息 start ----------

        pluginBuilder(context, "org.mybatis.generator.plugins.ToStringPlugin");
        pluginBuilder(context, "org.mybatis.generator.plugins.FluentBuilderMethodsPlugin");

        commentGeneratorBuilder(context);

        jdbcConnectionBuilder(context);

        javaTypeResolverBuilder(context);

        javaModelGeneratorBuilder(context);

        sqlMapGeneratorBuilder(context);

        javaClientGeneratorBuilder(context);

        tableBuilder(context, myGeneratorConfig.getSchema(), myGeneratorConfig.getTableNames());

        // ---------- 配置信息 end ----------


        // --------- 校验,执行 ---------
        Configuration config = new Configuration();
        config.addClasspathEntry(myGeneratorConfig.getClassPath());
        config.addContext(context);
        DefaultShellCallback callback = new DefaultShellCallback(true);

        try {
            MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config, callback, warnings);
            myBatisGenerator.generate(null);
        } catch (Exception e) {
            e.printStackTrace();
        }
        logger.warn("warnings=" + warnings);
    }

    /**
     * plugin
     * @param context
     */
    private void pluginBuilder(Context context, String configurationType) {
        PluginConfiguration pluginConfiguration = new PluginConfiguration();
        pluginConfiguration.setConfigurationType(configurationType);
        context.addPluginConfiguration(pluginConfiguration);
    }

    /**
     * commentGenerator
     * @param context
     */
    private void commentGeneratorBuilder(Context context) {
        CommentGeneratorConfiguration commentGeneratorConfiguration = new CommentGeneratorConfiguration();
        commentGeneratorConfiguration.addProperty("suppressDate", "true");
        commentGeneratorConfiguration.addProperty("suppressAllComments", "false");
        commentGeneratorConfiguration.addProperty("addRemarkComments", "true");
        commentGeneratorConfiguration.addProperty("javaFileEncoding", "UTF-8");
        context.setCommentGeneratorConfiguration(commentGeneratorConfiguration);
    }

    /**
     * jdbcConnection
     * @param context
     */
    private void jdbcConnectionBuilder(Context context) {
        JDBCConnectionConfiguration jdbc = new JDBCConnectionConfiguration();
        jdbc.setConnectionURL(myGeneratorConfig.getUrl());
        jdbc.setDriverClass(myGeneratorConfig.getDriverClass());
        jdbc.setUserId(myGeneratorConfig.getUser());
        jdbc.setPassword(myGeneratorConfig.getPassword());
        context.setJdbcConnectionConfiguration(jdbc);
    }

    /**
     * javaTypeResolver
     * @param context
     */
    private void javaTypeResolverBuilder(Context context) {
        JavaTypeResolverConfiguration javaTypeResolverConfiguration = new JavaTypeResolverConfiguration();
        javaTypeResolverConfiguration.addProperty("forceBigDecimals", "true");
        context.setJavaTypeResolverConfiguration(javaTypeResolverConfiguration);
    }

    /**
     * javaModelGenerator
     * @param context
     */
    private void javaModelGeneratorBuilder(Context context) {
        JavaModelGeneratorConfiguration javaModel = new JavaModelGeneratorConfiguration();
        javaModel.setTargetPackage(myGeneratorConfig.getJavaModelGeneratorPackage());
        javaModel.setTargetProject(myGeneratorConfig.getJavaModelGeneratorProject());
        javaModel.addProperty("trimStrings", "true");
        javaModel.addProperty("enableSubPackages", "true");
        context.setJavaModelGeneratorConfiguration(javaModel);
    }

    /**
     * sqlMapGenerator
     * @param context
     */
    private void sqlMapGeneratorBuilder(Context context) {
        SqlMapGeneratorConfiguration sqlMapGeneratorConfiguration = new SqlMapGeneratorConfiguration();
        sqlMapGeneratorConfiguration.setTargetPackage(myGeneratorConfig.getSqlMapGeneratorPackage());
        sqlMapGeneratorConfiguration.setTargetProject(myGeneratorConfig.getSqlMapGeneratorProject());
        sqlMapGeneratorConfiguration.addProperty("enableSubPackages", "true");
        context.setSqlMapGeneratorConfiguration(sqlMapGeneratorConfiguration);
    }

    /**
     * javaClientGenerator
     * @param context
     */
    private void javaClientGeneratorBuilder(Context context) {
        JavaClientGeneratorConfiguration javaClientGeneratorConfiguration = new JavaClientGeneratorConfiguration();
        javaClientGeneratorConfiguration.setTargetPackage(myGeneratorConfig.getJavaClientGeneratorPackage());
        javaClientGeneratorConfiguration.setTargetProject(myGeneratorConfig.getJavaClientGeneratorProject());
        javaClientGeneratorConfiguration.setConfigurationType("XMLMAPPER");
        javaClientGeneratorConfiguration.addProperty("enableSubPackages", "true");
        context.setJavaClientGeneratorConfiguration(javaClientGeneratorConfiguration);

    }

    /**
     * table
     * @param context
     * @param schema        添加SQL表名前面的库名
     * @param tableName
     */
    private void tableBuilder(Context context, String schema, String...tableName) {
        for (String table : tableName) {
            TableConfiguration tableConfiguration = new TableConfiguration(context);
            tableConfiguration.setTableName(table);
            tableConfiguration.setCountByExampleStatementEnabled(false);
            tableConfiguration.setUpdateByExampleStatementEnabled(false);
            tableConfiguration.setDeleteByExampleStatementEnabled(false);
            tableConfiguration.setSelectByExampleStatementEnabled(false);
            if (StringUtils.isNotBlank(schema)) {
                tableConfiguration.setSchema(schema);
                tableConfiguration.addProperty("runtimeSchema", schema);
            }
            context.addTableConfiguration(tableConfiguration);
        }
    }
}
```

1. 运行

```java
public class MainGenerator {

    @Test
    public void generator() {
        new MybatisGeneratorMain("generator.yml");
    }
}
```