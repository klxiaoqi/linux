
# 离线安装

- 下载
- 解压

``` bash
sudo tar -zxvf jdk-8u171-linux-x64.tar.gz
sudo mv jdk1.8.0_171 /usr/lib/xxx
```

# 在线安装

- 安装ppa
``` bash
sudo add-apt-repository ppa:webupd8team/java 
sudo apt-get update
```  

- 安装 jdk

``` bash
sudo apt-get install oracle-java8-installer
```

- 验证安装是否成功

```
java -version

```
> 成功后会出现：
```
java version “1.8.0_171” 
Java(TM) SE Runtime Environment (build 1.8.0_171-b11) 
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)
```
> 如果系统中安装有多个JDK版本，则可以通过如下命令设置系统默认JDK为Oracle JDK 8：

```
sudo update-java-alternatives -s java-8-oracle
```

# 配置环境变量

> 编辑/etc/profile文件
> 假设  JAVA_HOME对应的位置在 `/usr/lib/jvm/java-8-oracle` 处

```
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export JRE_HOME=/usr/lib/jvm/java-8-oracle/jre
export PATH=${JAVA_HOME}/bin:$PATH
```

> 然后执行 

```
source /etc/profile
```
> 验证：

```
echo $JAVA_HOME
```
