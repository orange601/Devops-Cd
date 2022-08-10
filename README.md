# 무중단 배포

## 무중단 배포 방식 ##
- 롤링(Rolling Update) 방식
- **블루 그린(Blue-Green Deployment) 방식**
- 카나리(Canary Release) 방식

## Docker ##
### 1. Dockerfile생성 ( dockerfile을 이용해 docker 이미지 생성한다. ) ###
    - 아무위치에 생성해도 상관없다.
    - 현재 host는 windows이므로 C:\orange\dockers\dockerfiles\java 여기에 생성했다.

설명을 위한 SAMPLE
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

내가 작업한 dockerfile
````
FROM adoptopenjdk/openjdk11:alpine-jre

ADD jar/*.jar webservice/app.jar
ADD properties/*.properties /webservice/config/application-prod1.properties

EXPOSE 60001

ENTRYPOINT ["java", "-Dspring.profiles.active=prod1", "-jar", "webservice/app.jar"]
````

### 2. image 생성 ###
        - docker build --tag myjava:0.1 .
        - 이미지이름:태그(버전)

### 3. container 실행 ###
        - $ docker run --name webserver(컨테이너 이름) -p 50001:8080 -d  jre11:alpine-jre(이미지:버전)
        - p는 포트를 의미
        - d는 백그라운드에서 실행
        - 8080:80 앞에 포트는 host 뒤에 포트는 container 포트


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
