


## 1、创建目录,手动上传文件到目录

mkdir /opt


## 2、修改主机名,所有节点
hostnamectl set-hostname cdh01

## 3、配置主机名映射, 所有节点
echo -e '192.168.80.xxx cdh01\n192.168.80.xxx cdh02\n192.168.80.xxx cdh03'  >> /etc/hosts

## 4、免密登录
    # 生成秘钥
    ssh-keygen -t rsa
    # 拷贝秘钥
    ssh-copy-id cdh01

## 5、关闭防火墙，所有节点
    systemctl stop firewalld
    systemctl disable firewalld

## 6. 关闭SElinux
    # 查看状态： 
    getenforce
    # 临时关闭： 
    setenforce 0
    # 永久关闭： 
        sed -i 's/SELINUX=enforcing/SELINUX=disable/' /etc/selinux/config
    # 验证: cat /etc/selinux/config

## 7、禁用透明大页,所有节点
    # 查看: 透明大页设置和启动状态
    cat sys/kernel/mm/transparent_hugepage/defrag
    cat sys/kernel/mm/transparent_hugepage/enabled
    # 操作: 临时关闭
    echo never > sys/kernel/mm/transparent_hugepage/defrag
    echo never > sys/kernel/mm/transparent_hugepage/enabled
    # 永久关闭
    echo 'echo never > sys/kernel/mm/transparent_hugepage/defrag' >> /etc/rc.d/rc.local
    echo 'echo never > sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.d/rc.local
    ## 验证: 
    cat  /etc/rc.d/rc.local

## 8、修改linux swappiness参数， 所有节点
    # 为了避免服务器使用swap功能而影响服务器性能，一般会把vm.swappiness修改为0,(cloudera建议10以下)
    # 查看
    cd /usr/lib/tuned
    grep "vm.swappiness" * -R
    # 操作
    sed -i s/"vm.swappiness = 30"/"vm.swappiness = 10"/g /usr/lib/tuned/virtual-guest/tuned.conf

## 9、安装jdk 所有节点
    ## 注意CDH的安装要求使用指定版本 oracle-j2SDK1.8
    # 查看安装， 如果有安装需要删除以前的；
    rpm -qa |grep java
    # 删除jdk： yum remove java*
    # 上传， 安装命令
    mkdir -p /opt/cdh
    
    scp oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm  centos02:/opt/cdh
    scp oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm  centos03:/opt/cdh
    rpm -ivh ./oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm

    # 查找jdk安装路径
    find / -name java
    # 配置环境变量
    echo 'export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera' >> /etc/profile
    echo 'export PATH=.:$JAVA_HOME/bin:$PATH' >> /etc/profile
    source /etc/profile
    # 验证 java -version

## 10、上传jdbc依赖包 所有节点
    # 创建目录
    mkdir -p /usr/share/java
    # 重命名 
    mv mysql-connector-java-8.0.12.jar mysql-connector-java.jar
    # 将mysql-connector-java.jar移动和复制到所有节点/usr/share/java
    cp mysql-connector-java.jar /usr/share/java
    scp mysql-connector-java.jar cdh01:/usr/share/java
    ...

## 11、安装mysql(master节点在cdh01)
    # 查询出来已安装的mariadb
    rpm -qa |grep mariadb
    # 卸载mariadb 文件名是上面查出来的
    rpm -e --nodeps 文件名
    # 上传安装包到 /opt目录
    mysql-8.0.18-linux-glibc2.12-x86_64.tar.xz
    # 解压安装包
    tar Jxvf mysql-8.0.18-linux-glibc2.12-x86_64.tar.xz
    # 重命名
    mv mysql-8.0.18-linux-glibc2.12-x86_64 mysql
    mv mysql /usr/local
    # 创建数据目录
    mkdir /usr/local/mysql/data
    # 创建编辑my.cnf
    vim /etc/my.cnf
    <<EOF
        [client]
        port = 3306
        socket = /tmp/mysql.sock
        [mysqld]
        port = 3306
        user = mysql
        socket = /tmp/mysql.sock
        basedir = /usr/local/mysql
        datadir = /usr/local/mysql/data
        log-error = /usr/local/mysql/error.log
        pid-file = /usr/local/mysql/mysql.pid
        transaction_isolation = READ-COMMITTED
        character-set-server = utf8
        collation-server=utf8_general_ci
        lower_case_table_names = 1
        sql_mode="STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ENGINE_SUBSTITUTION,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO"
EOF
    # 创建组
    groupadd mysql
    # 创建用户
    useradd -g mysql mysql 
    # 授权
    chown -R mysql:mysql /usr/local/mysql
    chown -R 755 /usr/local/mysql
    cd /usr/local/mysql
    # 初始化mysql
    ./bin/mysqld --initialize --user=mysql
    # 启动mysql
    ./support-files/mysql.server start

    #将mysql添加为服务
    cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
    # 设置开机启动
    cd /etc/init.d
    chmod 755 /etc/init.d/mysql
    chkconfig --add mysql
    chkconfig --level 345 mysql on
    service mysql restart
    # 配置环境变量
    echo 'export MYSQL_HOME=/usr/local/mysql' >> /etc/profile
    echo 'export PATH=.:$MYSQL_HOME/bin:$PATH' >> /etc/profile
    source /etc/profile
    # 使用 默认密码登录mysql, 查看日志中有password
    cat error.log 
    # mysql -uroot -pxxxx
    # 第一次登录需要重新设置root密码
    alter user 'root'@'localhost' identified by '123456';
    # 开启远程访问
    create user 'root'@'%' identified with mysql_native_password by '123456' ;
    grant all on *.* to 'root'@'%';
    flush privileges;

## 12、安装 apache httpd服务（master节点）
    # 安装
    yum install httpd -y 
    # 启动. 启动之后就可以使用ip去页面上访问文件夹了
    systemctl start httpd 
    # 设置开机启动
    systemctl enable httpd 
## 13、配置cloudera manager 安装包yum源（master ）
    # 创建目录
    mkdir -p /var/www/html/cloudera-repos/cm6
    # 创建仓库
    cd /var/www/html/cloudera-repos/cm6
    # 上传安装包    *.rpm和授权文件

    yum install -y creterepo
    createrepo .
    # 创建repo (所有节点)
    mkdir -p /etc/yum.repos.d
    vim /etc/yum.repos.d/cloudera-manager.repo
    <<EOF
    [cloudera-manager]
    name=Cloudera Manager 6.3.1
    baseurl=http://cdh01/cloudera-repos/cm6
    gpgkey=https://archive.cloudera.com/cm6/6.3.1/redhat7/yum/RPM-GPG-KEY-cloudera
    gpgcheck=1
    enabled=1
    autorefresh=0
    type=rpm-md
EOF
    # 拷贝
    scp /etc/yum.repo.d/cloudera-manager.repo centos03:/etc/yum.repo.d/
    # 清理并缓存 所有节点
    yum clean all
    yum makecache

## 14、 安装cloudera manager （master节点）
    # 执行安装
    yum install -y cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
    # 如果缺少yum源则上传安装包, 已准备好  执行以下安装命令
    rpm -pql cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
    rpm -ivh cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
    rpm -ivh cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
    rpm -ivh cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
    #如果报错需要 执行yum -y install 对应缺少的内容
    yum -y install mod_ssl python-psycopg2 MySQL-python postgresql-libs.x86_64 /lib/lsb/init-functions openssl-devel
    rpm -ivh cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
    # 其他节点执行 
    yum install -y cloudera-manager-agent cloudera-manager-daemons
    或者 rpm -ivh cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm

    # 将 CDH-6.3.1-1CDH6.3.2.P0XXXXparcel 文件放到目录 /opt/cloudera/parcel-repo路径下
    mkdir -p /opt/cloudera/parcel-repo
    # 执行校验
    sha1sum CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel | awk '{print $1}' > CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha

    # 创建mysql 库 后边会用到

    # 执行CM 初始化脚本
    cp ./mysql-connector-java-8.0.18.jar /opt/cloudera/cm/lib
    /opt/cloudera/cm/schema/scm_prepare_database.sh mysql cdh root 123456

    # 启动服务
    systemctl start cloudera-scm-server.service

    # 查看状态
    systemctl status cloudera-scm-server.service -l

    # master 查看服务是否自动起了
    netstat -anp | grep 7180

## 15 页面查看
    ip:7180

# FQA
## 16、 yum install ntp
     # 设置开机启动
     chkconfig ntpd on
     # 服务启动
     systemctl start ntpd
 
     # 手动同步时钟的方法
    ntpdate -u 0.cn.pool.ntp.org



## 报错相关
    # 如果报错: downloader   ERROR    Failed op: 'ascii' codec can't decode byte 0xe5 in position 5: ordinal not in range(128)
    vim /opt/cloudera/cm-agent/lib/python2.7/site-packages/cmf/downloader.py
    ## 添加: sys.setdefaultencoding('utf8')
    # 然后执行
    systemctl stop cloudera-scm-agent
    rm -rf /opt/cloudera/parcels
    rm -rf /opt/cloudera/parcel-cache/
    systemctl start cloudera-scm-agent
    systemctl status cloudera-scm-agent


## python3 安装

    # 创建目录
     mkdir  /usr/local/python3/
     wget https://www.python.org/ftp/python/3.7.1/Python-3.7.1rc2.tgz
       tar -zxf  Python-3.7.1rc2.tgz

      mv ./Python-3.7.1rc2 /usr/local/python3/
      cd /usr/local/python3/

      cd Python-3.7.1rc2/
    # 安装依赖
      yum install -y gcc
      # 初始化
      ./configure --with-ssl
    # 安装
      sudo yum install libffi-devel
      make
      make install
    # 设置默认py3
      ln -s /usr/local/python3  /usr/bin/python
      sudo rm -rf /usr/bin/python
      ln -s /usr/local/python3  /usr/bin/python

## 设置yum使用py版本
    # 修改下面两个文件中的脚本头
    /usr/bin/yum
    /usr/libexec/urlgrabber-ext-down

## com.cloudera.cmon.MgmtServiceLocatorException: Could not find a HOST_MONITORING nozzle from SCM.
    # 在主节点上，修改/opt/cm-5.15.1/etc/cloudera-scm-agent/config.ini文件：
    # //查看文件句柄数，显示1024，显然太小
    # ulimit -n 
    1024
    # //修改限制
    #vi /etc/security/limits.conf 
    #//在文件后加入下面内容：
    * soft nofile 100000
    * hard nofile 100000
    # 此步骤需要重启机器生效，可以设置完后再重启。

## 报错: MainThread agent        ERROR    Heartbeating to 192.168.226.129:7182 failed.
    # 检查端口是否通过: nc -w 1 10.10.40.91 7182

## 修改serverhost地址
    vi /etc/cloudera-scm-agent/config.ini
## 卸载
    rpm -qa | grep cloudera

##  cdh 主机运行状况不良。 - cdh03.com
    rm -rf /var/lib/cloudera-scm-agent/cm_guid
    systemctl restart cloudera-scm-agent

## 查看目录大小
 du -h -x --max-depth=1


 ## 删除文件
 /var/lib/cloudera-scm-server/search 
 /tmp/
