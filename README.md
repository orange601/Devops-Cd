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
