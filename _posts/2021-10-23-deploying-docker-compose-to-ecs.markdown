---
layout: post
title:  "docker compose 구성 그대로 ECS에 배포하기"
date:   2021-10-21 00:01:57 +0900
categories: AWS
---

사실상 이 글은 튜토리얼이라기 보다는 문제 해결 기록에 가깝습니다. 왜냐하면, 이미 좋은 튜토리얼이 있기 때문이지요. 시도1과 시도2를 보면 충분히 따라하고 응용할 수 있기 때문에 자세한 튜토리얼은 링크를 참고하세요.

저는 제 삽질기, 특히 ARM 아키텍처에서 발생하는 문제를 찾아내고 해결하는 과정을 적어보겠습니다.

## 시도1: 무엇을 하려고 했나요?

- docker-compose 구성을 ECS에 그대로 배포하려고 했음
- [https://www.docker.com/blog/docker-compose-from-local-to-amazon-ecs/](https://www.docker.com/blog/docker-compose-from-local-to-amazon-ecs/) 이 문서대로 따라했음
- 그런데 로컬에서는 잘 실행되는 구성이, ECS에 올리면 리소스를 만들자마자 곧바로 지워지는 상황이 발생함

```
➜  ecsblog-demo git:(master) ✗ docker compose up
WARNING services.build: unsupported attribute
WARNING services.build: unsupported attribute
[+] Running 16/16
 ⠿ ecsblog-demo                   DeleteComplete                  214.0s
 ⠿ BackendTaskExecutionRole       DeleteComplete                  165.0s
 ⠿ FrontendTaskExecutionRole      DeleteComplete                  108.0s
 ⠿ CloudMap                       DeleteComplete                  209.0s
 ⠿ FrontendTCP80TargetGroup       DeleteComplete                  104.0s
 ⠿ DefaultNetwork                 DeleteComplete                  162.0s
 ⠿ Cluster                        DeleteComplete                  162.0s
 ⠿ LogGroup                       DeleteComplete                  164.0s
 ⠿ LoadBalancer                   DeleteComplete                   97.0s
 ⠿ Default80Ingress               DeleteComplete                   98.0s
 ⠿ DefaultNetworkIngress          DeleteComplete                   97.0s
 ⠿ BackendTaskDefinition          DeleteComplete                  132.0s
 ⠿ FrontendTaskDefinition         DeleteComplete                   65.0s
 ⠿ FrontendServiceDiscoveryEntry  DeleteComplete                   57.0s
 ⠿ BackendServiceDiscoveryEntry   DeleteComplete                  115.0s
 ⠿ BackendService                 DeleteComplete
```

## 시도2: 또다른 시도

- 이번엔 [https://aws.amazon.com/ko/blogs/containers/deploy-applications-on-amazon-ecs-using-docker-compose/](https://aws.amazon.com/ko/blogs/containers/deploy-applications-on-amazon-ecs-using-docker-compose/) 이 문서를 가지고 다시 시도함
- 이건 잘됨!!!! 대체 뭐지??

## 분석

- 혹시 `docker compose` 대신 `docker-compose` 란 명령을 써서 그런가? → 그런거 아님
    > Note how I am now using the main docker binary to do this, instead of the docker-compose binary I have used above to deploy locally. The docker binary now comes with an expanded functionality to docker compose up (more on this later). [출처](https://aws.amazon.com/ko/blogs/containers/deploy-applications-on-amazon-ecs-using-docker-compose/)
- `docker compose logs` 라는 명령어가 있어서 분석을 시도함
    - compose up을 하면 AWS 상에 리소스를 만드는데, 리소스 중 log group을 생성하고 나면, 에러로 인해 지워지기 전 잠깐동안 로그를 볼 수 있음

    ```
    ➜  ecsblog-demo git:(master) ✗ docker compose logs
    backend  | standard_init_linux.go:228: exec user process caused: exec format error
    ```

    - 오잉? `exec format error` 는 아예 파일 실행을 못한것임!
        - backend 중 Dockerfile 뒤져봄

        ```
        FROM python:3.7-alpine
        WORKDIR /app
        COPY requirements.txt /app
        RUN pip3 install -r requirements.txt
        COPY main.py /app/main.py
        ENTRYPOINT ["python3"]
        CMD ["/app/main.py"]
        ```

        - 딱히 이상한 거 못느낌
    - 혹시나 싶어 다음 키워드로 구글링 해봄 `standard_init_linux.go:228: exec user process caused: exec format error`
        - 맨 위에 뜨는 해시뱅 이슈는 여기에 해당사항 없는 내용이고,
        - [M1 맥북 사용자의 EKS 배포 오류에 대하여](https://appleg1226.tistory.com/35) → 이거 제목 보는 순간 느낌 팍 왔음! 아 이거다!
            - 해당 글에 분석 과정이 있으니 참고

## 원인 발견과 해결
- 시도1과 시도2의 차이는, **이미지를 내가 빌드했느냐, 원래 빌드된걸 썼느냐의 차이**
    - 나는 M1에서 빌드했으니까 docker hub에 arm64로 올라갔고, 이걸 amd64 기반인 ECS에서 돌리려고 하니 돌아갈리가 없는거다
    - 찾아보니 compose 파일에 아키텍처를 지정할 수 있었음. [링크 참고](https://velog.io/@m0ai/M1-%EB%A7%A5%EC%97%90%EC%84%9C-x8664-%EB%8F%84%EC%BB%A4-%EC%9D%B4%EB%AF%B8%EC%A7%80-%EB%B9%8C%EB%93%9C-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0)
    - 최종 docker-compose.yml 파일

    ```
    version: "3.0"
    services:
      frontend:
        image: gotoweb/starter-front
        build: frontend
        ports:
          - 80:80
        depends_on:
          - backend
        platform: linux/amd64
      backend:
        image: gotoweb/starter-back
        build: backend
        platform: linux/amd64
    ```

- 결국 amd64 플랫폼으로 다시 빌드하고 ECS 배포 성공!