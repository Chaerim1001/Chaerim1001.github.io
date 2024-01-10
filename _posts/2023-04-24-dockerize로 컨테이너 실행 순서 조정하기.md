---
title: dockerize로 컨테이너 실행 순서 조정하기
date: 2023-04-24
categories: [Docker]

tags: [Docker, Docker compose]
---

## docker compose를 사용할 때 발생할 수 있는 문제
**docker compose**란, 여러 개의 컨테이너로부터 이루어진 서비스를 구축, 실행하는 순서를 자동으로 하여 관리를 간단히 할 수 있도록 제공하는 기능입니다. 즉, docker compose를 사용하면 여러 컨테이너를 함께 연결하여 구동시킬 수 있습니다!

그러나 docker compose에서는 컨테이너가 실행되고 완료되는 순서를 완벽히 보장하지는 않기 때문에 컨테이너끼리의 연결이 필요할 경우 문제가 발생할 수 있습니다.

## 컨테이너 실행 순서 조정하기
### (1) depends on 속성 사용하기
`depends_on`은 여러 개의 컨테이너를 docker compose로 실행할 때, 실행 시점을 제어할 수 있는 속성입니다.
`depends_on` 을 사용하여 컨테이너 실행 순서를 제어할 수 있지만, 컨테이너의 실행 프로세스 완료 상태까지 제어하는 것은 아니기 때문에 완벽하게 우리가 원하는 대로 동작하리라는 보장은 없습니다.

```yml
version: "3"

services:
  db:
    image: mysql:5.7
    container_name: mysql
    env_file: ./.env
    volumes:
      - ./database/data:/var/lib/mysql
    ports:
      - 3306:3306
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: database_name
    networks:
      - server

  web:
    image: web_image_name
    container_name: web
    env_file: ./.env
    restart: always
    ports:
      - 8000:8000
    depends_on:
      - db
    networks:
      - server
networks:
  server:

```

### (2) dockerize 모듈
저는 docker-compose로 node.js 서버와 mysql을 연결할 때 문제가 발생하였습니다. node.js 서버보다 mysql이 늦게 구동하여 발생한 문제로 [dockerize](https://github.com/jwilder/dockerize) 모듈을 사용하여 해결하였습니다.

## dockerize 사용 예시
### (1) dockerize 모듈 설치하기
우선 `Dockerfile`에 `dockerize` 설치 명령어를 추가해 줍니다.

```dockerfile
ENV DOCKERIZE_VERSION v0.2.0
RUN wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz
```

### (2) dockerize로 실행할 스크립트 작성하기 (docker-entrypoint.sh)
설치한 `dockerize`를 사용할 shell script를 작성해 줍니다. 저는 db로 사용하는 컨테이너가 모두 올라간 뒤 서버를 실행시키도록 스크립트를 작성했습니다.

```sh
echo "wait db server"
dockerize -wait tcp://db:3306 -timeout 20s

echo "start node server"
npm start
```

### (3) 스크립트 파일 실행 명령어 추가하기

앞서 작성한 스크립트를 실행하도록 `Dockerfile`에 명령어를 추가해 줍니다.

```dockerfile
RUN chmod +x docker-entrypoint.sh
ENTRYPOINT ./docker-entrypoint.sh
```

### (4) 전체 Dockerfile
아래는 Dockerfile 코드입니다.
이렇게 dockerize를 통해 컨테이너 실행 순서를 조정할 수 있습니다.

```dockerfile
FROM node:16.14.2

ENV DOCKERIZE_VERSION v0.2.0
RUN wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz

RUN mkdir -p /app
WORKDIR /app
COPY package*.json /app/
RUN npm install
COPY . /app/

RUN chmod +x docker-entrypoint.sh
ENTRYPOINT ./docker-entrypoint.sh

EXPOSE 8000
```

> 참고 <br>
> https://unpasoadelante.tistory.com/197 <br>
> https://github.com/jwilder/dockerize <br>
> https://jupiny.com/2016/11/13/conrtrol-container-startup-order-in-compose/
