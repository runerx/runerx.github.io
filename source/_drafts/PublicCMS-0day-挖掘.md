---
title: PublicCMS 0day 挖掘
tags:
---



```
git clone https://github.com/sanluan/PublicCMS.git

```

### mysql构建

```
docker run --rm -itd --name my-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:8.0.15

docker exec -it my-mysql bash

mysql -u root -p

开启远程访问
use mysql;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
flush privileges;



#使用sql创建数据库（数据库名称publiccms）
/PublicCMS/data/publiccms/publiccms.sql
/PublicCMS/publiccms-parent/publiccms-core/src/main/resources/initialization/sql



```

运行

```
cd publiccms-parent
mvn clean package
cd publiccms/target
java -jar publiccms.war
```

访问：http://172.16.42.151:8080/admin





pom

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	........
    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>
```





修改密码

vim data/publiccms/database.properties

```
...
jdbc.password=123456
```



codeql

```
codeql database create publiccms-qldb -l java --command="mvn clean install -Dmaven.test.skip=true" --overwrite
```

