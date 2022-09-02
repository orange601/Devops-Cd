# CD #
- 배포 자동화
- 지속적인 서비스 제공(Continuous Delivery) 또는 지속적인 배포(Continuous Deployment)를 의미
- Build 후 배포 과정만을 설명한다.

## 목표 ##
- **블루 그린(Blue-Green Deployment) 배포**
- Jenkins의 webhook과 build 과정은 CI에서 관리한다.
- [Jenkins를 통한 DEVOPS CI](https://github.com/orange601/devops-ci)

## Scenario ##
1. bash를 명령어를 통한 deploy.sh 파일 실행
2. service-url.inc의 URL과 port가 변경된다.
3. docker-compose.blue.yml 과 docker-compose.green.yml이 번갈아가면서 실행된다.
4. Nginx로 접속시 blue 혹은 green으로 접속한다.

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

### 1. docker-compose.yml 을 이용한 Nginx 설치 ### 
- docker-compose 명령을 이용해서 nginx 이미지와 container 설치
- docker-compose.yml 위치에서 아래 명령어를 실행한다.
````docker
$ docker-compose -p project_name up -d
````
- p: 프로젝트명 ( docker에서 관리할때 사용 )
- d: background 실행
- f: 파일이름이 docker-compose.yml이 아닐 경우나 파일경로가 현재 경로에 없을 경우 사용한다.
- 예) docker-compose -p dtd -f //docker-compose.blue.yml up -d


### 2. Nginx Proxy 설정 ( reverse proxy ) ### 
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

## JAVA ##
### 1. docker-compose.yml 을 이용한 설치 ### 
- 아래는 docker-compose.blue.yml 이다.
````yml
version: '3.1'

services:
  was-blue-prod1:
    image: adoptopenjdk/openjdk11:alpine-jre
    container_name: was-blue-prod1 # service-url.inc의 container 이름과 같아야한다.
    command: "java -jar /sharing/java/jar/app.jar"
    ports:
      - "9090:58001"
````
- container_name의 이름과 nginx에서 service-url.inc 파일의 URL이 같은지 확인 ( set $service_url http://was-blue-prod1:50001; )
- ports 설정에서 9090:58001 부분중 뒷 port(local port)가 service-url.inc 파일의 포트 부분과 일치하는지 확인



