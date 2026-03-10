# 跨域配置功能

## 1. 跨域及单点登录
1.1 关闭Apache的Basic认证
```text
  # 将以下内容注释
  #<LocationMatch ^/+Windchill/+(;.*)?>
  #  AuthName "Windchill"
  #  AuthType Basic
  #  AuthBasicProvider Windchill-LDAP 
  #  Require valid-user
  #</LocationMatch>
  
  # 对指定的路径进行拦截
  <LocationMatch ^/+Windchill/+netmarkets/+jsp/+com/+hihonor/+login/+(;.*)?$>
    AuthName "Windchill"
    AuthType Basic
    AuthBasicProvider Windchill-LDAP
    Require valid-user
  </LocationMatch>
```

1.2 修改codebase/WEB-INF/web.xml文件，添加跨域配置以及拦截器配置
```xml
# 在web.xml的最开始的位置增加以下内容
#<filter>
    #<filter-name>CORSFilter</filter-name>
        #<filter-class>com.hihonor.bop.wp.login.filter.CORSFilter</filter-class>
    #</filter>

    #<filter-mapping>
        #<filter-name>CORSFilter</filter-name>
        #<url-pattern>/*</url-pattern>
    #</filter-mapping>
        
#<filter>

<filter-name>SsoFilter</filter-name>
    <filter-class>com.hihonor.login.filter.SsoFilter</filter-class>
</filter>

<filter-mapping> 
    <filter-name>SsoFilter</filter-name>
    <url-pattern>/*</url-pattern>
      <!-- 
          <url-pattern>/servlet/hwrest/*</url-pattern>
          -->
</filter-mapping>
```
> 注意：Windchill的跨域拦截配置，可以不再Windchill中的codebase/WEB-INF/web.xml中配置CORSFilter

针对未启用HTTPS的，可以在HTTPServer/conf/httpd.conf文件中增加以下内容
`待补充`

针对启用HTTPS的, 需要在conf/conf.d/20-mod_ssl.conf文件中配置以下内容
```conf
# 找到 <VirtualHost _default_:443> 字样
# 在SSLEngine on下增加
# ========================start==============================


# 确保 OPTIONS 请求被 Apache 立即处理
RewriteEngine On
RewriteCond %{REQUEST_METHOD} OPTIONS
RewriteRule ^(.*)$ $1 [R=200,L]

# 设置 CORS 响应头
SetEnvIf Origin "http(s)?://(wp.+.test.hihonor.com|workplace.hihonor.com|localhost.hihonor.com:8000)$" AccessControlAllowOrigin=$0
Header always set Access-Control-Allow-Origin %{AccessControlAllowOrigin}e env=AccessControlAllowOrigin
Header always set Access-Control-Allow-Methods "POST,GET,OPTIONS,PUT,DELETE"
Header always set Access-Control-Allow-Headers "Origin, Content-Type, Accept, Authorization, X-Requested-With"                   
Header always set Access-Control-Allow-Credentials "true"

# ========================end==============================
```

1.3 部署对应的类
直接通过ide进行打成jar包

## 2. 部署配置文件

2.1 HomeService.json
功能地图的配置
主题工具栏的配置

2.2 CommonConfig.json
app.home 项目根路径，在导出报表的时候有用到
运维人员信息配置
系统域名配置
注意：需要调整下载目录；由于UAT、生产环境是Windows环境，同时Method和BackgroundMethod是两台服务器为了解决文件互通，需要将下载的目录改为共享盘目录

2.3 onlineSwitchConfig.json
```text
WeLink开关配置
上线时，需要调整
{
  "configValue": "DEV",
  "configKey": "env",
  "description": "当前系统环境"
},
configValue不能是DEV, DEV会固定的发给湧鑫的账号
```
2.4 RequisitionTypeConfig.json
检索数据时，排除的流程模板的名称配置

2.5 systemIntegrationConfig.json
WeLink的Token校验，消息发送的配置

2.6 TableHeaderCustom.json
搜索结果以及高级搜索结果的表格的默认配置

## 3. 解决Background中使用Spring注解的问题

3.1 编写servlet.xml，用于在BackgroudMethod中使用Spring注解
在codebase/WEB-INF/目录下新建文件 HonorDispatcher-servlet.xml 文件
文件内容如下:
```xml
<?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:mvc="http://www.springframework.org/schema/mvc"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
          xmlns:context="http://www.springframework.org/schema/context"
          xmlns:aop="http://www.springframework.org/schema/aop"
          xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.3.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.3.xsd
                http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd">
        <context:annotation-config />
        <bean class="org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor"/>
        <!-- 把标记了@Controller注解的类转换为bean -->
        <context:component-scan base-package="com.hihonor.bop" use-default-filters="false">
                <context:include-filter type="annotation"
                                                                expression="org.springframework.stereotype.Service"/>
                <context:include-filter type="annotation"
                                                                expression="org.springframework.stereotype.Component"/>
                <context:include-filter type="annotation"
                                                                expression="org.springframework.stereotype.Controller"/>
        </context:component-scan>
         <import resource="classpath:/com/hihonor/bop/business/core/resource/config/*.exceptions.xml"/>
    <import resource="classpath:/com/hihonor/bop/adapter/core/resource/config/*.exceptions.xml"/>
</beans>
```

3.2 编写honor-bop-wp-configs.xml文件，用于扫描和XML注入Bean
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.ptc.com/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                        http://www.ptc.com/schema/mvc http://www.ptc.com/schema/mvc/mvc-10.0.xsd">

    <!--工艺参数模板编辑器控制器-->
    <context:component-scan base-package="com.hihonor.bop"/>
    
        <import resource="classpath:/com/hihonor/bop/business/core/resource/config/*.exceptions.xml"/>
    <import resource="classpath:/com/hihonor/bop/adapter/core/resource/config/*.exceptions.xml"/>
</beans>
```

## 4. 执行sql
```sql
-- 创建用户自定义菜单功能
CREATE TABLE home_custom_function_menu(
    ID varchar(50) PRIMARY KEY ,
    MENU_LIST varchar(4000),
    "CREATE" varchar(50),
    CREATE_TIME timestamp,
    "UPDATE" varchar(50),
    UPDATE_TIME timestamp,
    DEL_FLAG int DEFAULT 0
);

CREATE INDEX composite$creator ON home_custom_function_menu("CREATE") TABLESPACE indx;
CREATE INDEX composite$updator ON home_custom_function_menu("UPDATE") TABLESPACE indx;
```
执行MyFavor数据库表创建
```shell
cd <WT_HOME>/db
execute_sql_script.bat com/hihonor/myfavor/Make_pkg_myfavor_Table.sql bop honor123
execute_sql_script.bat com/hihonor/myfavor/Make_pkg_myfavor_Index.sql bop honor123

execute_sql_script.bat com/hihonor/wftask/Make_pkg_wftask_Table.sql bop honor123
execute_sql_script.bat com/hihonor/wftask/Make_pkg_wftask_Index.sql bop honor123
```
## 5. 注册拉模文件

文件：codebase/descendentRegistry.properties
```properties
wt.fc.WTObject=com.hihonor.myfavor.model.MyFavor
wt.fc.WTObject=com.hihonor.wftask.model.TaskCloseNotice
```
文件: codebase/modelRegistry.properties
```properties
com.hihonor.myfavor.model=MyFavor
com.hihonor.wftask.model=TaskCloseNotice
```

5.1 MyFavor
目前用来存储关闭的公告弹窗
暂时无法在HONOR E Link文档外展示此内容

5.2 TaskCloseNotice
暂时无法在HONOR E Link文档外展示此内容

## 6. 注册MVC控制器
在codebase/config/mvc下新建文件 honor-bop-wp-configs.xml 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.ptc.com/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                        http://www.ptc.com/schema/mvc http://www.ptc.com/schema/mvc/mvc-10.0.xsd">

    <!--工艺参数模板编辑器控制器-->
    <context:component-scan base-package="com.hihonor.bop"/>
    <!--异常处理-->
    <import resource="classpath:/com/hihonor/bop/business/core/resource/config/*.exceptions.xml"/>
    <import resource="classpath:/com/hihonor/bop/adapter/core/resource/config/*.exceptions.xml"/>
    
</beans>
```
## 7. Jar包以及升级替换
```text
#依赖的jar包
jackson-core-asl-1.9.13.jar
jackson-mapper-asl-1.9.13.jar
commons-httpclient-3.1.jar
# 需要删除旧版本的jar包，升级到新版本的jar包
fastjson-1.2.5.jar    ->   fastjson-1.2.83.jar
```
## 8. 创建队列
commonExportQueue   流程   类型
## 9. 创建清理队列
定时清理数据库中记录的数据
## 10. 属性创建与关联
```text
1. 作业指导书类别【关联到 排拉图、作业指导书】
名称：作业指导书类别
内码：DocCategory
合法值：工艺方案|工艺路线
2. 工艺变更上新增属性  关联影响工艺路线
名称：关联影响工艺路线
内码: associateRefPlan
合法值: 对象的oid
```
