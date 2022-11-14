# CD #
- 배포 자동화
- 지속적인 서비스 제공(Continuous Delivery) 또는 지속적인 배포(Continuous Deployment)를 의미
- Build 후 배포 과정만을 설명한다.

## 블루 그린(Blue-Green Deployment) 배포 ##
- Docker에 jenkins와 java 설치 후 연동하는것이 목표
- Jenkins의 webhook과 build 과정은 CI에서 관리한다.
- [Jenkins를 통한 DEVOPS CI](https://github.com/orange601/devops-ci)

## Scenario ##
1. github push 이벤트 발생
2. webhook을 통해 jenkins에 이벤트 push 전달
3. jenkis에서 gradle 통해 github 소스 build 후 jar 생성
4. Execute shell 이벤트를 통해 bash를 명령어로 deploy.sh 파일 실행
	````cmd
	$ bash /sharing/bash/deploy.sh
	````

## 설치시 주의 사항 및 Troubleshooting시 가장먼저 확인해야 되는 부분 ##
- Nginx 확인 
1. default.conf 파일에서 service-url.inc의 파일위치가 맞는지 확인
2. service-url.inc 파일과 default.conf의 변수명이 일치하는지 확인 ( $service_url ) 오타확인
3. service-url.inc 파일에서 http뒤에 URL이 container 명이 맞는지 확인 ( http://was-container:9898; )
	+ nginx에서 WAS(spring)으로 proxy를 전달해야 하므로 WAS container명과 일치하는지 확인
4. service-url.inc 파일에서 포트(9898)가 내부port가 맞는지 확인 ( http://was-container:9898; )
	+ WAS container의 내부 포트가 맞는지 확인
	+ java의 compose 내용중 뒤에 오는 포트
	+ 예)
	````yml
	services:
	  was-prod1:	    
	    ports:
	      - "9090:9898"
	````

## Nginx ##
### 1. nginx 이미지와 container 설치하기 위해 docker-compose.yml를 작성한다. ###
- docker-compose.yml 작성 후 적당한 위치에 두고 같은 위치에 Dockerfile도 작성한다.
- 사실 Dockerfile 없이 docker-compose.yml 하나로 설치가 가능하나 가독성을 위해(내생각) 작성한다.

	````yml
	version: '3.1'

	services:
	  nginx-proxy-server:
	    build: . # 현재위치에 있는 dockerfile을 빌드한다.
	    image: nginx:latest
	    container_name: nginx-proxy-server
	    restart: on-failure # 실패시 재실행
	    ports:
	      - "12345:80" # hostport : containerport 
	    volumes:
	      - orange_volume:/share/ # nginx에 share이라는 dir가 생성된다.

	volumes:
	    orange_volume:

	networks:
	  default:
	    external:
	      name: your-orange-network
	````
	````yml
	# Dockerfile
	FROM nginx:latest
	RUN apt-get update && \
	apt-get -y install vim
	````
	




### 2. Nginx 설치 ### 
- docker-compose.yml 위치에서 아래 명령어를 실행한다.

	````docker
	$ docker-compose -p project_name up -d
	````
- p: 프로젝트명 ( docker에서 관리할때 사용 )
- d: background 실행
- f: 파일이름이 docker-compose.yml이 아닐 경우나 파일경로가 현재 경로에 없을 경우 사용한다.
- 예) docker-compose -p dtd -f //docker-compose.blue.yml up -d

### 3. Nginx 실행확인 ###
- compose에서 설정한 port로 접속한다.
> localhost:30000 

### Welcome to nginx! ###
If you see this page, the nginx web server is successfully installed and working. Further configuration is required.   

For online documentation and support please refer to nginx.org.   
Commercial support is available at nginx.com.   

Thank you for using nginx.

### Volume 생성 ###
- compose를 실행하면 자동으로 생성된다. 
- 자동생성될때 volume 이름앞에 프로젝트이름이 접두어로 사용되기때문에 미리 만들어두면 헷갈린다.

	````cmd
	$ docker volume create orange_volume
	````

### 네트워크 생성 ###
- 통신을 위해 Network를 생성한다.

	````yml
	$ docker network create your-orange-network
	````


### 4. Nginx Proxy 설정 ( reverse proxy ) ### 
- /etc/nginx/conf.d/default.conf 파일 수정
````
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;
	include /etc/nginx/conf.d/service-url.inc;

    location / {
        #root   /usr/share/nginx/html;
        #index  index.html index.htm;
		resolver 127.0.0.11;
		proxy_pass	$service_url;
    }
````
- include /etc/nginx/conf.d/service-url.inc;
- proxy_pass	$service_url;
- resolver 127.0.0.11; 
- resolver 확인은 아래 명령어로 확인한다.
````
$  cat /etc/resolv.conf
````

#### 같은위치에 service-url.inc 파일생성 ###
````
set $service_url http://was-blue-prod1:58001;
````
1. http 뒤에 container 이름을 사용한다.
2. 여기서는 WAS(springboot) container 이름을 사용했다.
3. **58001은 호스트 port가 아닌 "WAS의 내부" local port 이다!**
4. (WAS Container에서 9090:58001으로 설정했다면 뒤에 내부 port이다.)

### 5. Nginx Custom Iamge 만들기 ###
- 컨테이너에서 작업한 내용들은 컨테이너가 종료되면 함께 사라진다.
- 컨테이너에서 작업한 내용을 이미지로 커밋(Commit)하여 작업했던 내용을 다시 사용 할 수 있다.
- commit 명령어를 통해 컨테이너 상태 그대로의 이미지 생성
	> docker commit [CONTAINER ID] [IMAGE NAME]
	````cmd
	$ docker commit 1a0a902554ba jenkinsmake
	````
- CONTAINER ID는 docker ps -a로 확인한다.
````cmd
$ docker ps -a
````
````
CONTAINER ID   IMAGE               COMMAND       CREATED        STATUS                    PORTS     NAMES
1a0a902554ba   ubuntu:16.04        "/bin/bash"   9 days ago     Exited (0) 6 days ago               containernam
````

### Nginx 명령어 ###
- nginx container 실행
	````cmd
	$ docker start container_name
	````
- Restart
	````cmd
	sudo service nginx restart
	````

- Reload
	````cmd
	nginx -s reload
	````

## JAVA ##
### 1. docker-compose.yml 을 이용한 설치 ### 
- 아래는 docker-compose.blue.yml 이다.
````yml
version: '3.1'

services:
  orange-was-blue:
    image: adoptopenjdk/openjdk11:alpine-jre
    container_name: orange-was-blue # service-url.inc의 container 이름과 같아야한다.
    restart: on-failure
    command: "java -jar /sharing/java/jar/app.jar --server.port=50001" # properties가 아닌 여기서 port 설정한다. 이유는 아래에
    ports:
      - "60001:50001"
    environment:
      - "SPRING_PROFILES_ACTIVE=dev"
      - "SPRING_CONFIG_LOCATION=/sharing/java/prop/"
    volumes:
      - orange_volume:/sharing/

volumes:
    orange_volume:

networks:
  default:
      name: orange-network
      external: true
````
- properties 파일에서 port를 설정해도 되지만 propeties 파일을 blue와 green이 똑같은 propeties를 사용하기 위해 실행시 port설정한다.
- container_name의 이름과 nginx에서 service-url.inc 파일의 URL이 같은지 확인 ( set $service_url http://orange-was-blue:50001; )
- ports 설정에서 60001:50001 부분중 뒷 port(local port)가 service-url.inc 파일의 포트 부분과 일치하는지 확인

## bash.sh ##
- 윈도우에서 bash 프로그램을 수정하기 위해선 editor를 UNIX 형식으로 변환해야 한다.
- Windows와 UNIX의 끝줄 처리 방식이 다르기 때문이다. ( 줄바꿈 문자를 Windows에서는 \n\r을 사용하고, UNIX에서는 \n을 사용 )
- Notepad++ 를 이용해서 bash 프로그램을 수정할 계획이기때문에 Notepad의 옵션을 변경해야 한다.
- 편집 > 줄의 끝문자(EOL) 변환 > Unix 형식으로 변환


````bash
# 1. blue container가 실행여부 확인한다.
# 2. container 정상구동 확인 - 핑 10번
# 3. 정상확인시 service-url.inc에 포트변경 후 Nginx를 reload 시켜 80 port에 새로운 container를 바인딩
# 4. 전에 구동되고 있던 container는 삭제

PROJECT_NAME=dtd_v2
COMPOSE_DIR=/sharing/java/compose
NGINX_CONTAINER_NAME=nginx-proxy-server

# 1. blue docker가 실행되고 있는지 확인
EXIST_BLUE=$(docker-compose -p ${PROJECT_NAME} -f ${COMPOSE_DIR}/docker-compose.blue.yml ps | grep Up)

if [ -z "${EXIST_BLUE}" ]; then # -z는 문자열 길이가 0이면 true. Blue가 실행 중이지 않다는 의미.
	echo "Blue Will RUN"
	START_CONTAINER=blue
	EXTENAL_START_PORT=60001
	DOWN_CONTAINER=green
	EXTERNAL_DOWN_PORT=60002
	# 내부port
	INSIDE_START_PORT=50001
	INSIDE_DOWN_PORT=50002
	START_PROFILE=prod1
	DOWN_PROFILE=prod2
else
	echo "GREEN Will RUN"
	START_CONTAINER=green
	EXTENAL_START_PORT=60002
	DOWN_CONTAINER=blue
	EXTERNAL_DOWN_PORT=60001
	# 내부port
	INSIDE_START_PORT=50002
	INSIDE_DOWN_PORT=50001
	START_PROFILE=prod2
	DOWN_PROFILE=prod1	
fi

docker-compose -p ${PROJECT_NAME} -f ${COMPOSE_DIR}/docker-compose.${START_CONTAINER}.yml up -d

# 2
for cnt in {1..10} # 생성한 컨테이너 안에 Application이 정상적으로 구동되기까지 10번의 핑을 보내 확인한다.
do
	echo "check server start.."
	
	# host.docker.internal
	UP=$(curl -s host.docker.internal:${EXTENAL_START_PORT}/smile/act/health | grep 'UP')
	
	if [ -z "${UP}" ]; then # -z는 문자열 길이가 0이면 true
		sleep 10
		continue       
    else
		break
    fi
done

if [ $cnt -eq 10 ] # 10번동안 실행되지 않았다면, 배포실패, 종료
then
    echo "Deployment Failed."
	docker-compose -p ${PROJECT_NAME} -f ${COMPOSE_DIR}/docker-compose.${START_CONTAINER}.yml down
    exit 1
fi

# 3 
# sed 명령어를 이용해서  service-url.inc의 url값 변경
# sed -i "s/기존문자열/변경할문자열" 파일경로 입니다. 여기서는 container의 내부(local)port를 사용하고 있음을 주의해야 합니다.
DOWN_WAS=was-${DOWN_CONTAINER}-${DOWN_PROFILE}:${INSIDE_DOWN_PORT}
START_WAS=was-${START_CONTAINER}-${START_PROFILE}:${INSIDE_START_PORT}
sed -i "s/${DOWN_WAS}/${START_WAS}/" /sharing/nginx/conf/service-url.inc
echo "Deploy Completed!!"

# 4
echo "${DOWN_CONTAINER}-Container:${EXTERNAL_DOWN_PORT} DOWN"
# 오류가 났을때 down하지 않는 이유는 에러 로그를 확인하려고 down시키지 않는다.
# docker-compose -p ${PROJECT_NAME} -f ${COMPOSE_DIR}/docker-compose.${DOWN_CONTAINER}.yml down

docker exec -i ${NGINX_CONTAINER_NAME} nginx -s reload
````


