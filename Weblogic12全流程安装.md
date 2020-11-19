# 1. weblogic12全流程安装

weblogic12c静默安装，配置免密登陆，创建Server，创建Domain，配置启停脚本，权限分离，控制台优化，打补丁。

## 1.1. 安装前检查工作

软件介质：

- jdk1.8.0_192.tar.gz
- fmw_12.2.1.3.0_wls.jar

### 1.1.1. 操作系统版本

注：做weblogic集群的节点操作系统版本需要保持一致

```shell
lsb_release -a
or
cat /etc/redhat-release
```

### 1.1.2. 检查hosts

/etc/hosts文件中配置的主机名必须和hostname一致，并且需要计入同一集群中所有server的IP和hostname的映射

```shell
# 类似下面这种
192.168.127.14  apphost
192.168.127.15  weblogicappli
192.168.127.16  localhost
```

### 1.1.3. 同步时间

```shell
# 先查看是否已经设定
crontab -l
# 若未设置，添加即可
crontab -e
# 同步自己的ntp服务器，下面是例子
*/10 * * * * /usr/sbin/ntpdate 10.1.1.1
# 注：若是编辑/etc/crontab文件来设定定时任务，需要在写入后执行crontab /etc/crontab来使之生效
```

## 1.2. 创建相关用户

```shell
# 权限分离
groupadd -g 500 admin
useradd -u 500 -g admin -G wheel -d /home/admin admin
useradd -u 530 -g admin -G wheel -d /home/weblogic weblogic
# 配置口令
echo "%TGB7ygvadmin110" | passwd --stdin admin
echo "%TGB7ygvweblogic110" | passwd --stdin weblogic
```

## 1.3. 目录赋权

```shell
# 创建相关目录并赋权
mkdir /app
chown root. /app
chmod 777 /app
mkdir -p /app/applications/testdomain/logs
mkdir -p /app/applications/testdomain/script
cd /app
mkdir install weblogic oraInventory
chown -R admin. install/ weblogic/ oraInventory/
chown -R weblogic. applications/
chmod -R 775 applications/
chmod -R 755 install/ weblogic/ oraInventory/
```

## 1.4. 部署jdk

```shell
mkdir /usr/java
tar -zxvf jdk1.8.0_192.tar.gz
chmod -R 755 /usr/java
```

## 1.5. 配置用户环境变量

```shell
vi /home/admin/.bash_profile
umask 022
ulimit -c unlimited
export JAVA_HOME=/usr/java/jdk1.8.0_192
export JRE_HOME=/usr/java/jdk1.8.0_192/jre
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export PATH=$JAVA_HOME/bin:$PATH

vi /home/weblogic/.bash_profile
alias kill='sudo -u admin kill'
alias netstat='sudo -u admin netstat'
umask 002
ulimit -c unlimited

source /home/admin/.bash_profile
source /home/weblogic/.bash_profile
```

## 1.6. 静默安装weblogic12c

```shell
cd /app/install

# 创建初始化文件
vi oraInst.loc
inventory_loc=/app/oraInventory
inst_group=admin

# 创建响应文件
vi wls.rsp
[ENGINE]
# DO NOT CHANGE THIS.
Response File Version=1.0.0.0
[GENERIC]
ORACLE_HOME=/app/weblogic
INSTALL_TYPE=weblogic Server
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false

# 安装
java -jar fmw_12.2.1.3.0_wls.jar -silent -responseFile /app/install/wls.rsp -invPtrLoc /app/install/oraInst.loc
# 注：如果因故需要重新安装，需要清空/app/oraInventory和/app/weblogic下面的文件，隐藏文件也要删除，否则重装会报错
# 验证安装版本
java -cp /app/weblogic/wlserver/server/lib/weblogic.jar weblogic.version
```

## 1.7. 创建domain

```shell
# 添加随机数，解决创建过程过慢的问题
vi /app/weblogic/oracle_common/common/bin/config_internal.sh
# 对应配置添加 -Djava.security.egd=file:///dev/urandom
JVM_ARGS=" -Djava.security.egd=file:///dev/urandom -Dpython...${CONFIG_JVM_ARGS}"
# 创建脚本，注意admin.jar文件若提示不存在，查找出来换一下路径即可
vi create_domain.py
readTemplate('/app/weblogic/wlserver/common/templates/wls/wls.jar')
cd('Server/AdminServer')
set('ListenPort',7001)
cmo.setName('AdminServer')
cd('/')
cd('Security/base_domain/User/weblogic')
cmo.setName('admin')
cmo.setPassword('admin123')
setOption('ServerStartMode','prod')
setOption('OverwriteDomain','true')
writeDomain('/app/weblogic/user_projects/domains/testdomain')
closeTemplate()
exit()
# 执行脚本
sh /app/weblogic/oracle_common/common/bin/wls.sh create_domain.py
# 注：若还是创建过慢，可以输入命令 export CONFIG_JVM_ARGS='-Djava.security.egd=file:///dev/urandom' 设置临时变量加速
```

## 1.8. 创建后的优化

```shell
# 配置默认切换路径
vi /home/admin/.bash_profile
cd /app/weblogic/user_projects/domains/testdomain

vi /home/weblogic/.bash_profile
cd /app/applications/testdomain/script

source /home/admin/.bash_profile
source /home/weblogic/.bash_profile

# 为weblogic用户添加权限
visudo
# 下面一行找到注释掉
#%wheel ALL=(ALL) ALL
Cmnd Alias COMMAND=/bin/kill,/bin/netstat,/app/weblogic/user_projects/domains/testdomain/*.sh
weblogic  ALL=(admin) NOPASSWD:COMMAND

# 设置文件句柄数
vi /etc/security/limits.conf
admin hard  nofile  65535
admin soft  nofile  65535

# 配置服务开机自启动，以下是针对一个受管的情况
vi /etc/rc.local
su - weblogic -c "sh /app/applications/testdomain/script/strAdminServer.sh"
sleep 30
su - weblogic -c "sh /app/applications/testdomain/script/startappserver1.sh"

# 创建软连接
cd /app/weblogic/user_projects/domains/testdomain
mkdir logs
ln -s /app/weblogic/user_projects/domains/testdomain/logs /app/applications/testdomain/logs

# 修改ulimit
vi /app/weblogic/oracle_common/common/bin/commBaseEnv.sh
# 找到ulimit并修改
ulimit -n 10240

# 禁用12c启动后产生的多余进程
vi /app/weblogic/user_projects/domains/testdomain/bin/setDomainEnv.sh
# 找到DERBY_FLAG="true"并修改
DERBY_FLAG="false"

# 加速weblogic启停优化
cd /app/weblogic/user_projects/domains/testdomain/bin
# 下面一大段直接复制执行即可
echo 'JAVA_OPTIONS="${JAVA_OPTIONS} -Dfile.encoding=utf-8 -Djava.security.egd=file:///dev/urandom"' >> setDomainEnv.sh &&
echo "export JAVA_OPTIONS" >> setDomainEnv.sh && sed -i "/umask/ s/umask 027/umask 002/" startWeblogic.sh &&
sed -i "/ignoreSessions s/'true'/'true',timeOut=0,force='true'" stopWeblogic.sh
```

## 1.9. 创建Server

```shell
cd /app/weblogic/user_projects/domains/testdomain
./startWeblogic.sh

# 浏览器登陆console，地址为http://IP:7001/console 账户密码是创建domain时create_domain.py脚本内设定的密码

主页>安全领域>myrealm>领域角色>用户和组>按需求更改admin的口令
创建受管appserver1，服务器监听地址，端口从8001开始
重启AdminServer后在bin目录下启动受管
./startManagedWeblogic.sh appserver1 t3://IP:7001 输入create_domain.py中配置的密码，控制台观察是否成功启动

# 配置免密启动
cd /app/weblogic/user_projects/domains/testdomain/servers/AdminServer
mkdir security
cd security
vi boot.properties
username=admin
password=更改后的admin账户口令
cd ..
cp -r security/ /app/weblogic/user_projects/domains/testdomain/servers/appserver1

# 创建启停脚本
cd /app/weblogic/user_projects/domains/testdomain

# AdminServer启动脚本
vi strAdminServer.sh
export DOMAIN_DIR=/app/weblogic/user_projects/domains/testdomain
export DEPLOY_DIR=/app/applications/testdomain
mv ${DEPLOY_DIR}/logs/AdminServer.out ${DEPLOY_DIR}/logs/AdminServer`date +%Y-%m-%d-%H:%M:%S`.out
mv ${DEPLOY_DIR}/logs/AdminServer.err ${DEPLOY_DIR}/logs/AdminServer`date +%Y-%m-%d-%H:%M:%S`.err
mv ${DEPLOY_DIR}/logs/AdminServer_gc.out ${DEPLOY_DIR}/logs/AdminServer_gc`date +%Y-%m-%d-%H:%M:%S`.out
export USER_MEM_ARGS="-Xms2048m -Xmx2048m -Xmn768m -XX:+UseConcMarkSweepGC -XX:+HeapDumpOutOfMemoryError -verbose:gc -Xloggc:./logs/AdminServer_gc.out -XX:+PrintGCDetails -XX:PrintGCTimeStamps -Dweblogic.threadpool.MinPoolSize=100 -Dweblogic.threadpool.MaxPoolSize=300 -Dweblogic.management.disableManagedServerNotifications=true"
nohup sh ${DOMAIN_DIR}/bin/startWeblogic.sh 1> ${DEPLOY_DIR}/logs/AdminServer.out 2> ${DEPLOY_DIR}/logs/AdminServer.err &
echo "nohup sh ${DOMAIN_DIR}/bin/startWeblogic.sh 1> ${DEPLOY_DIR}/logs/AdminServer.out 2> ${DEPLOY_DIR}/logs/AdminServer.err &"

# AdminServer停止脚本
vi stpAdminServer.sh
cd /app/weblogic/user_projects/domains/testdomain/bin
./stopAdminServer.sh

# ManagedServer启动脚本
vi startappserver1.sh
export DOMAIN_DIR=/app/weblogic/user_projects/domains/testdomain
export DEPLOY_DIR=/app/applications/testdomain
mv ${DEPLOY_DIR}/logs/appserver1.out ${DEPLOY_DIR}/logs/appserver1`date +%Y-%m-%d-%H:%M:%S`.out
mv ${DEPLOY_DIR}/logs/appserver1.err ${DEPLOY_DIR}/logs/appserver1`date +%Y-%m-%d-%H:%M:%S`.err
mv ${DEPLOY_DIR}/logs/appserver1_gc.out ${DEPLOY_DIR}/logs/appserver1_gc`date +%Y-%m-%d-%H:%M:%S`.out
export USER_MEM_ARGS="-Xms2048m -Xmx2048m -Xmn768m -XX:+UseConcMarkSweepGC -XX:+HeapDumpOutOfMemoryError -XX:+HeapDumpOutOfMemoryError=${DOMAIN_DIR}/threaddump1.sh -verbose:gc -Xloggc:./logs/appserver1_gc.out -XX:+PrintGCDetails -XX:PrintGCTimeStamps -Dweblogic.threadpool.MinPoolSize=100 -Dweblogic.threadpool.MaxPoolSize=300 -Dweblogic.management.disableManagedServerNotifications=true"
nohup sh ${DOMAIN_DIR}/bin/startManagedWeblogic.sh appserver1 t3://替换为主机IP:7001 1> ${DEPLOY_DIR}/logs/appserver1.out 2> ${DEPLOY_DIR}/logs/appserver1.err &

# ManagedServer停止脚本
vi stopappserver1.sh
/app/weblogic/user_projects/domains/testdomain/bin
./stopManagedWeblogic.sh appserver1 t3://IP:7001

# 线程快照脚本
vi threaddump1.sh
pid=`ps -ef | grep appserver1 | grep java | grep -v grep | awk '{print $2}'`
/usr/java/jdk1.8.0_192/bin/jstack $pid > logs/thread_appserver1`date +%Y-%m-%d-%H:%M:%S`.logs

# 以下四个脚本配置在weblogic用户下，供应用发布使用
cd /app/applications/testdomain/script
vi strAdminServer.sh
sudo -u admin /app/weblogic/user_projects/domains/testdomain/strAdminServer.sh
vi stpAdminServer.sh
sudo -u admin /app/weblogic/user_projects/domains/testdomain/stpAdminServer.sh
vi startappserver1.sh
sudo -u admin /app/weblogic/user_projects/domains/testdomain/startappserver1.sh
vi stopappserver1.sh
sudo -u admin /app/weblogic/user_projects/domains/testdomain/stopappserver1.sh
```

## 1.10. 控制台优化

```shell
# 优化testdomain.log
锁定并编辑>testdomain>配置>日志记录>活动类型(时间)>限制文件保留数(勾选)>保留数30>保存>激活更改

# 优化appserver1.log和access.log
更改一般信息和HTTP选项卡的日志配置，同上
http高级>access.log格式改为“扩展”>日志记录格式参数为“x-GWXFF c-ip date time cs-method cs-uri sc-status”>保存>激活更改
同时把GWXFF的jar包放在/app/weblogic/user_projects/domains/testdomain/lib下

# 修改stage模式
修改stage模式的前提是domain采用1个AdminServer+N个ManagedServer的模式（n>=1）
选择appserver1>配置>部署>临时模式>调整为不存放(no stage)>保存>激活更改

# 配置账户
安全领域>myrealm>用户和组>用户
创建weblogic用户并加入Deployer组，设置口令等

# 配置SNMP
点击中间左上角的主页（注销旁边），选择右下角SNMP
SNMP配置，端口改为8161和7050  社区前缀TESTpub  陷阱版本V2

# 添加t3协议
testdomain>安全>筛选器>配置如下
连接筛选器：weblogic.security.net.ConnectionFilterImpl
规则：(t3网段根据实际配置来,下面仅是例子)

# 允许10和12网段访问，屏蔽其他所有网段
10.0.0.0/8  * * allow t3
12.0.0.0/8  * * allow t3
* * * deny  t3

# 禁用IIOP协议，每个服务都要禁用
Admin和Managed服务器>协议>取消勾选IIOP

# 添加数据源
服务>数据源>新建
需要准备好数据库IP  用户名密码以及连接串等信息
```

## 1.11. 打补丁

```shell
补丁包分为OPatch升级工具包和补丁本身包
假设p28000_Generic.zip为工具包  p35000_Generic.zip为补丁本身
unzip p28000_Generic.zip
mv 6880880/ /app/weblogic
cd /app/weblogic
chown admin. -R 6880880/
java -jar /app/weblogic/6880880/opatch_generic.jar -silent oracle_home/app/weblogic/ -invPtrLoc /app/install/oraInst.loc
# 或者直接使用生成的oraInst.loc文件
# java -jar /app/weblogic/6880880/opatch_generic.jar -silent oracle_home/app/weblogic/ -invPtrLoc /app/weblogic/oraInst.loc
cd /app/weblogic/OPatch
./opatch lsinventory -jre /usr/java/jdk1.8.0_192/jre
输出过程已省略...
mkdir PATCH_TOP
cd PATCH_TOP
unzip p35000_Generic.zip
cd 325524/
/app/weblogic/OPatch/opatch apply -jre /usr/java/jdk1.8.0_192/jre
输出过程已省略...
```
