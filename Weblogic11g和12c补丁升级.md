# Weblogic补丁升级

本文涉及补丁是2021年1月的最新补丁

```shell
# 说明
-prod_dir     weblogic的家目录
-patchlist    补丁ID,就是补丁包里.jar文件的文件名
```

## 11g版本

此处weblogic安装目录是`/app/weblogic/wlserver_10.3`, 根据实际情况更改即可

### 查看现有的补丁

```shell
cd /app/weblogic/utils/bsu
./bsu.sh -prod_dir=/app/weblogic/wlserver_10.3 -status=applied -verbose -view
```

### 卸载旧补丁(若有)

此处假设已存在旧补丁, 补丁号是NA7A

```shell
./bsu.sh -install -verbose -patchlist=NA7A -prod_dir=/app/weblogic/wlserver_10.3
```

### 打最新的补丁

将最新补丁`p32052267_1036_Generic.zip`上传到`/app/weblogic/utils/bsu/cache_dir`, 并赋予和weblogic同样的用户权限

```shell
cd /app/weblogic/utils/bsu/cache_dir
unzip p32052267_1036_Generic.zip
cd ..
# 这个补丁号1YWL, 在上面解压后就能看到一个对应的.jar文件
./bsu.sh -install -verbose -patchlist=1YWL -prod_dir=/app/weblogic/wlserver_10.3
```

### 升级bsu打补丁工具(若需)

如果升级补丁失败并提示bsu版本较低, 可采用下面方式升级bsu, 然后重新打补丁即可

```shell
# 将bsu升级工具包(比如: p26664545_1036_Generic.zip)上传到/app/weblogic/utils/bsu下, 解压
# 下面这些步骤在README.txt都能查看到
# 安装
sh bsu_update.sh install
# 回退
sh bsu_update.sh rollback
```

## 12c版本

Weblogic安装目录为`/app/weblogic`, 12c不需要卸载旧补丁, 直接升级即可

### 升级新补丁

将最新补丁`p32300397_122130_Generic.zip`上传到`/app/weblogic/OPatch/PATCH_TOP/`, 并赋予和weblogic同样的用户权限

```shell
cd /app/weblogic/OPatch/PATCH_TOP/
unzip p32300397_122130_Generic.zip
cd 32300397/
/app/weblogic/OPatch/opatch apply -jre /usr/java/jdk1.8.0_192/jre
```

> -jre /usr/java/jdk1.8.0_192/jre 这个参数是指定使用的jdk, 如果环境变量配置的jdk版本太低无法打补丁, 可以像这样指定一下; 若配置好的jdk版本较高,不指定亦可

### 升级OPatch工具(若需)

```shell
# 将OPatch工具包(比如: p28186730_139424.zip)上传到/app/weblogic, 解压
# 我这里-invPtrLoc /app/install/oraInst.loc文件是事先准备好的. 可不加该参数, 默认去oracle_home下面也能找到对应文件
/usr/java/jdk1.8.0_192/bin/java -jar /app/weblogic/6880880/opatch_generic.jar -slient oracle_home=/app/weblogic -invPtrLoc /app/install/oraInst.loc
# 查看OPatch版本
cd /app/weblogic/OPatch
./opatch version
```
