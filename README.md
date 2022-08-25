# 무중단 배포 #
- Docker를 이용한 배포과정을 설명한다.
- bash 프로그램을 이용하여 블루 그린 배포를 설정한다. 

## 무중단 배포 방식 ##
- 롤링(Rolling Update) 방식
- **블루 그린(Blue-Green Deployment) 방식**
- 카나리(Canary Release) 방식

## Docker-compose 이용한 설정 ##
### Scenario ###
1. docker-compose 통한 Nginx 설치
2. Nginx 브라우저 접속 확인
3. default.conf 파일 수정 - service-url.inc 파일 생성
4. 네트워크생성
5. docker-compose 통한 java-spring image 생성
6. 접속 확인
7. deploy.sh 작성

### 주의사항 ###
1. 웹훅
    - http://locahost:8080를 입력하시면 정상적으로 동작하지 않습니다.
    - http://public-ip:8080 같이 공개 IP를 사용하는 경우에도 정상적으로 동작하지 않습니다.
    - ngrok 어플리케이션을 통해 외부에서 접근할 수 있는 도메인을 사용합니다.
    - Content type - application/json 타입을 사용합니다.

### 설명 ###
1. **네트워크 생성**
    - docker network create was-network
    - container간의 통신을 위해 network를 생성한다.

2. **네트워크 조회**
    - docker network ls

3. **nginx이미지를 생성할 nginx compose 생성**
    - docker-compose up ( 기본명령어 )
    - docker-compose -p proxy-server up
    - docker-compose -p dtd -f ./docker-compose.blue.yml up -d
    - p: 프로젝트 이름 
    - f: docker-compose.yml를 설정 파일로 사용합니다. 다른 이름이나 경로의 파일을 Docker Compose 설정 파일로 사용하고 싶다면 -f 옵션으로 명시를 해줍니다.
    - d: 백그라운드로 실행
    - 같은 이름의 프로젝트이면 container가 같은 그룹으로 생성된다. 
    - 프로젝트 이름을 설정하지않고 기본명령어만 사용할 경우 compose 그룹으로 생성된다.
    - docker-compose.yml
    ````yml
    version: '3.1'

    services:
      nginx-proxy:
        image: nginx:latest
        container_name: nginx-proxy
        ports:
          - "30000:80"
        volumes:
          - ../volume/conf:/etc/nginx/conf.d/

    networks:
      default:
        external:
          name: was-network
    ````

## Dockerfile 이용한 설정 ##
1. **Dockerfile생성 ( dockerfile을 이용해 docker 이미지 생성한다. )**
    - 아무위치에 생성해도 상관없다.
    - 현재 host는 windows이므로 C:\orange\dockers\dockerfiles\java 여기에 생성했다.

        #### DOCKERFILE-SAMPLE-1 ####
        ````cmd
        # docker hub에서 사용할 image
        FROM adoptopenjdk/openjdk11:alpine-jre

        # 앞에는 host의 jar명 뒤에는 container명
        ADD *.jar app1.jar

        # HOST의 포트
        EXPOSE 8888

        # container가 올라가면 실행될 명령어
        CMD ["java", "-jar", "app1.jar"]
        ````

        #### DOCKERFILE-SAMPLE-2 ####
        ````
        FROM adoptopenjdk/openjdk11:alpine-jre

        ENV PROFILE prod1

        ADD jar/*.jar webservice/app.jar
        ADD properties/application-${PROFILE}.properties /webservice/config/application-${PROFILE}.properties

        EXPOSE 60001

        ENTRYPOINT ["java", "-Dspring.profiles.active=${PROFILE}", "-Dspring.config.location=/webservice/config/application-${PROFILE}.properties", "-jar", "webservice/app.jar"]
        ````

        #### DOCKERFILE-SAMPLE-3 ####
        ````
        FROM adoptopenjdk/openjdk11:alpine-jre

        ADD jar/*.jar webservice/app.jar
        ADD properties/*.properties /webservice/config/

        ENTRYPOINT ["java", "-jar", "webservice/app.jar"]
        ````

2. **JDK image 생성**
    - SAMPLE3을 이용해 이미지를 만든다.
    - docker build --tag jre11:alpine-jre .
    - **끝에 마침표 필수**
    - 이미지이름:태그(버전)

3. **container 실행**
    - $ docker run -e "SPRING_PROFILES_ACTIVE=prod1" -e "SPRING_CONFIG_LOCATION=/webservice/config/" --name was1 -p 60001:50001 jre11:alpine-jre
    - p는 포트를 의미
    - d는 백그라운드에서 실행
    - 60001:50001 앞에 포트는 host 뒤에 포트는 container 포트
        
4. **nginx 이미지 설치**
    - docker pull nginx:latest
    - 최신버전 설치

5. **nginx 실행**
    - docker run 명령에서 container 간 연결 옵션은 --link <컨테이너 이름>:<별칭> 형식이다.
    - $ docker run --name proxy -p 9090:80 --link was1:was1 nginx

6. **nginx proxy 설정하기**
    - /etc/nginx/conf.d/default.conf
    ````cmd
      include /etc/nginx/conf.d/service-url.inc;
      location / {
        resolver 127.0.0.11;
        proxy_pass $service_url;
      }
    ````
    - resolver 확인 cat /etc/resolv.conf
    - **was1은 컨테이너명, 50001은 호스트 port가 아닌 "내부" local port 이다! 확실하게 해둘것!**
    - nginx -s reload
    
7. **service-url.inc 설정하기**
    - include /etc/nginx/conf.d/service-url.inc;
    - set $service_url http://was1:50001;
      
#### DOCKER 설명 ####
- https://www.youtube.com/watch?v=Bhzz9E3xuXY&t=350s
- https://pyrasis.com/Docker/Docker-HOWTO
- https://hub.docker.com/_/ubuntu?tab=tags ( 우분투를 예시로 든 image 검색 )

#### docker 실행 ####
- docker run -i -t containerName /bin/bash ( 예제에서는 우분투를 실행했음 )
- run 과 start 의 차이는 run은 도커 진입 start는 실행만 하고 docker에 진입은하지 않음
- docker attach containerName ( start로 실행 후 진입하는 명령어 )
- image는 하나고 여러개의 container의 종속을 만들 수 

![2022-08-08 14 03 14](https://user-images.githubusercontent.com/24876345/183343019-30da31a2-073d-4e69-a57b-69ed579d1134.png)

#### 명령어 ####

- 파일복사 ( container to host )
````
$ docker cp prod1:/etc/nginx/conf.d/ c:/devops/share
````

- 파일복사 ( host to container )
````
$ docker cp c:/devops/share/conf.d/ prod1:/etc/nginx/
````

- 컨테이너 실행
````
$ docker run --name webserver(컨테이너 이름) -p 8080:80 -d  nginx(이미지:버전)
````
컨테이너와 컨테이너를 연결하기
--link <연결할 컨테이너명>:<컨테이너 연결에 사용할 이름>
