# xxl-job-admin 的构建（docker）

1、从github上下载源码

```shell
wget https://github.com/xuxueli/xxl-job/archive/refs/tags/2.3.0.tar.gz
```

2、编译构建

```shell
cd xxl-job-admin/
mvn -f pom.xml clean package -DskipTests docker:build
```

3、启动镜像

```shell
/**
* 如需自定义 mysql 等配置，可通过 "-e PARAMS" 指定，参数格式 PARAMS="--key=value  --key2=value2" ；
* 配置项参考文件：/xxl-job/xxl-job-admin/src/main/resources/application.properties
* 如需自定义 JVM内存参数 等配置，可通过 "-e JAVA_OPTS" 指定，参数格式 JAVA_OPTS="-Xmx512m" ；
*/
docker run -e PARAMS="--spring.datasource.url=jdbc:mysql://182.61.42.24:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai --spring.datasource.username=wxb --spring.datasource.password=7421.*aA" -p 18080:8080 -v /tmp:/data/applogs --name xxl-job-admin  -d xxl-job-admin:2.3.0
```

