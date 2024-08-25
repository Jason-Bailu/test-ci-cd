- build

  ```shell
  # Jenkins机器：编译完成后，build生成一个新版本的镜像，push到远程docker仓库
  # Variables
  JENKINS_WAR_HOME='/var/lib/jenkins/workspace/maven-docker/target'
  DOCKERFILE_HOME='/var/lib/jenkins/docker-file/maven-docker-war'
  HARBOR_IP='172.28.29.243:80'
  REPOSITORIES='jenkins/maven-docker'
  HARBOR_USER='bailu'
  HARBOR_USER_PASSWD='Dxy000216'
  HARBOR_USER_EMAIL='1242857354@qq.com'
  
  # Copy the newest war to docker-file directory
  \cp -f ${JENKINS_WAR_HOME}/maven-docker.war ${DOCKERFILE_HOME}/maven-docker.war
  
  # Delete image early version
  sudo docker login -u ${HARBOR_USER} -p ${HARBOR_USER_PASSWD} ${HARBOR_IP} 
  IMAGE_ID=`sudo docker images | grep ${REPOSITORIES} | awk '{print $3}'`
  if [ -n "${IMAGE_ID}" ]; then 
  		sudo docker rmi ${IMAGE_ID}
  fi
  
  # Build image
  cd ${DOCKERFILE_HOME}
  TAG=`date +%Y%m%s-%H%M%S`
  sudo docker build -t ${HARBOR_IP}/${REPOSITORIES}:${TAG} . &>/dev/null
  
  # Push to the harbor registry
  sudo docker push ${HARBOR_IP}/${REPOSITORIES}:${TAG} &>/dev/null
  ```

- deploy

  ```shell
  # 拉取镜像，发布
  HARBOR_IP='172.28.29.243:80'
  REPOSITORIES='jenkins/maven-docker'
  HARBOR_USER='bailu'
  HARBOR_USER_PASSWD='Dxy000216'
  
  # 登录HABBOR
  docker login -u ${HARBOR_USER} -p ${HARBOR_USER_PASSWD} ${HARBOR_IP}
  
  # Stop container, and delete the container
  CONTAINER_ID=`docker ps | grep "maven-docker" | awk '{print $1}'`
  if [ -n "${CONTAINER_ID}" ]; then
  		docker stop ${CONTAINER_ID}
  		docker rm ${CONTAINER_ID}
  else # 如果容器启动失败，就需要找到该容器
  		docker ps -a | grep "maven-docker" | awk '{print $1}'
  		if [ -n "${CONTAINER_ID}" ]; then
  				docker rm ${CONTAINER_ID}
  		fi
  fi
  
  # Delete image early version
  IMAGE_ID=`sudo docker images | grep ${REPOSITORIES} | awk '{print $3}'`
  if [ -n "${IMAGE_ID}" ]; then
  		docker rmi ${IMAGE_ID}
  fi
  
  # Pull image
  TAG=$(curl -s http://172.28.29.243/api/v2.0/projects/jenkins/repositories/maven-docker/artifacts | grep -o '"name":"[^"]*"' | head -1 | awk -F ':"' '{print $2}' | tr -d '"')
  docker pull ${HARBOR_IP}/${REPOSITORIES}:${TAG} &>/dev/null
  
  # Run
  docker run -d --name maven-docker -p 8080:8080 ${HARBOR_IP}/${REPOSITORIES}:${TAG}
  ```

- Jar dockerfile

  ```dockerfile
  # Version 1.0
  # Base images
  FROM openjdk:11-jre-slim
  
  # Author
  MAINTAINER bailu <1242857354@qq.com>
  
  # ADD war.
  ADD maven-docker.jar /usr/app/maven-docker.jar
  
  # Define working directory
  WORKDIR /usr/app
  
  # Define environment variables
  
  # Define default command.
  CMD ["java", "-jar", "maven-docker.jar"]
  
  # Expose ports
  EXPOSE 8080
  ```

- War dockerfile

  ```dockerfile
  # cd /root/jenkins/docker-file/maven-docker-war
  # vim Dockerfile
  # Version 1.0
  # Base images
  FROM tomcat:8.0.36-alpine
  
  # Author
  MAINTAINER bailu <1242857354@qq.com>
  
  # ADD war.
  ADD maven-docker.war /usr/local/tomcat/webapps/
  
  # Define working directory
  WORKDIR /usr/local/tomcat/bin/
  
  # Define environment variables
  ENV PATH /usr/local/tomcat/bin:$PATH
  
  # Define default command.
  CMD ["catalina.sh", "run"]
  
  # Expose ports
  EXPOSE 8080
  ```

  