# 解决weblogic启动慢的问题

问题描述：中间件weblogic11g或12c启动很慢，控制台加载很慢，并没有报错，只是单纯的启动慢

解决方案1：这种方法不是全局设置，针对每个域进行配置

在weblogic domain的bin目录下的setDomainEnv.sh的文件末尾加入JAVA启动参数  
echo 'JAVA_OPTIONS="{JAVA_OPTIONS} -Djava.security.egd=file:/dev/./urandom"' >> setDomainEnv.sh  
echo 'export JAVA_OPTIONS' >> setDomainEnv.sh

解决方案2：全局设置随机数，修改对应jdk中的java文件

vi $JAVA_HOME/jre/lib/security/java.security  
将securerandom.source=file:/dev/urandom 修改为 securerandom.source=file:/dev/./urandom
