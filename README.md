# 무중단 배포

## 무중단 배포 방식 ##
- 롤링(Rolling Update) 방식
- 블루 그린(Blue-Green Deployment) 방식
- 카나리(Canary Release) 방식


## Docker ##
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
컨테이너 실행
````
$ docker run --name webserver(컨테이너 이름) -p 8080:80 -d  nginx(이미지:버전)
````
