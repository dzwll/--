#  启动脚本
branch=dev
service=spms-digital-service-sdpms
group=spms-digital
code_pull_cmd_str="git clone -b ${branch} http://wujiang003%40chinasofti.com@10.88.40.177:8099/${group}/${service}.git ./;   "
sh /cicd/k8s-java.3.0.quality.sh "${code_pull_cmd_str}"   ${service} 

#  执行脚本
version="latest"
echo "version " ${version}
else
version=`date +%Y%m%d%H%M`
echo "version " ${version}
fi

#集成工作路径（创建）
rm -rf ${code_path}
mkdir -p ${code_path}
cd ${code_path}

#最新仓库代码（获取）
echo "${project_path}"
eval "${project_path}"
#svn checkout ${project_path} ./

#最新仓库代码（编译）
echo "mvn clean install -U -Dmaven.test.skip=true    -Djava.net.preferIPv4Stack=true"
mvn clean install -U    -Djava.net.preferIPv4Stack=true


if [ $? -ne 0 ]; then
        echo "failed"
        exit 1
else
        echo "success!"
fi

if [ "${docker_is_image}" = "0" ]; then
        echo "not build images"
        echo "succeed"
        exit 0
else
        echo "continute"
fi


#最新发布镜像（定义）
APP_OPTIONS='$APP_OPTIONS'
server_port='$shark_server_port'
cat >runtmp <<EOF
#!/bin/bash
umask 0027
java  $APP_OPTIONS  -Djava.net.preferIPv4Stack=true  -jar /apps/app.jar
EOF
#cp -rf /opt/cicd/agent ./agent
cat runtmp
#cp `ls target/*jar |grep -v "\-sources" ` app.jar
/bin/cp -f `ls target/*.[jw]ar |grep -v "\-sources" ` app.jar
#cp datax
/bin/cp -a /home/dataxnew2.tar.gz ${code_path}
cat > Dockerfile <<EOF
FROM 192.168.80.201/library/jdk1.8:20220613
ENV  TZ="Asia/Shanghai"
ENV  APP_OPTIONS="-Xms128m -Xmx512m -Xss512k"
COPY app.jar /apps/app.jar
COPY runtmp /apps/run.sh
ADD dataxnew2.tar.gz /apps/
RUN chmod 550 -R /apps/*  && chown cig -R /apps/*
EXPOSE 8080
USER cig
CMD ["/apps/run.sh"]
EOF

#最新发布镜像（打包）
echo "docker build -t ${docker_namespaces}/${project_main}:${version} ."
if [ "${docker_server_ip}" = "-1" ]; then
        docker build -t ${docker_namespaces}/${project_main}:${version} .
else
        docker build -t ${docker_server_ip}/${docker_namespaces}/${project_main}:${version} .
fi

if [ $? -ne 0 ]; then
        echo "build failed"
        exit 1
else
        echo "build success!"
fi

#最新发布镜像（上传）
if [ "${docker_server_ip}" = "-1" ]; then
        docker login -u ${docker_server_username} -p${docker_server_password}
        docker push  ${docker_namespaces}/${project_main}:${version}
        docker rmi ${docker_namespaces}/${project_main}:${version}
        echo "image is:"${docker_namespaces}/${project_sub}:${version}
else
        docker login -u ${docker_server_username} -p${docker_server_password} ${docker_server_ip}
        docker push  ${docker_server_ip}/${docker_namespaces}/${project_main}:${version}
        docker rmi ${docker_server_ip}/${docker_namespaces}/${project_main}:${version}
        echo "image is:"${docker_server_ip}/${docker_namespaces}/${project_main}:${version}
fi
exit 0
