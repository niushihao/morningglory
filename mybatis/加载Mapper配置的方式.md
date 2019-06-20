### mybatis加载Mapper配置的5种方式
1、 根据Mapper类具体路径加载Mapper

```java
<configuration>
	<mappers>		
		<!-- class 级别的指定 -->
		<mapper class="com.nsh.mapper.UserModelMapper"/>
		<mapper class="com.nsh.mapper.UserModelTwoMapper"/>
	</mappers>
</configuration>
```
这种方式可以使用注解的方式，也可以使用xml的方式，但是如果使用xml的方式,mybatis会根据同名规则匹配，即在相同的包下找名称相同且为.xml格式的文件

2、 根据Mapper类所在包路径加载Mapper

```java
<configuration>
	<mappers>
		<package name="com.nsh.mapper"/>
	</mappers>
</configuration>
```
这种配置和第一种一样，即使用xml的方式时，Mapper.xml必须与Mapper.class同名，且在同一个包下。

3、 把Mapper.xml放到resource中,与Mapper.class分开

```java
<configuration>
	<mappers>
		<!-- 使用这个方案，可以单独指定Mapper的位置 -->
		<mapper resource="mybatis/mappings/UserModelMapper.xml"/>
		<mapper resource="mybatis/mappings/UserModelTwoMapper.xml"/>
	</mappers>
</configuration>
```
这种方式打破了Mapper.class和Mapper.xml必须同名且在同一目录的限制，可以在任意位置任意命名，但是有几个Mapper.xml就需要加几行

4、 使用spring的方式

```java
<!-- MustConfigPoint MyBatis begin -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	<property name="dataSource" ref="dataSource" />
	<property name="typeAliasesPackage" value="实体类包路径" />
	<property name="typeAliasesSuperType" value="实体类顶级包路径" />
	<property name="mapperLocations" value="classpath:/mybatis/mappings/**/*.xml" />
	<property name="configLocation" value="classpath:/mybatis/mybatis-config.xml"></property>
</bean>

<!-- MustConfigPoint 扫描basePackage下所有以@MyBatisDao注解的接口 -->
<bean id="mapperScannerConfigurer" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
	<property name="basePackage" value="mapper类的包路径" />
	<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
	<property name="annotationClass" value="com.msyd.framework.common.persistence.annotation.MyBatisDao" />
</bean>
```

5、 使用springboot的方式

```java
// yml方式
mybatis:
  config-location: classpath:config/mybatis-config.xml
  mapper-locations: classpath:mapper/*.xml
  
// 注解方式

```
