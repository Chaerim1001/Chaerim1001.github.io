---
title: Docker와 Github Actions로 CICD 파이프라인 구축하기
date: 2022-10-05
categories: [Devops, Github Actions]

tags: [Docker, Github Actions, CI/CD, AWS]
---

> 본 포스트는 아래의 환경을 기준으로 작성되었습니다. <br>
> Node 16.15.1
> mysql 5.7
 {: .prompt-info }


# 지속적 통합: CI(Continuous Integration)

## (1) Dockerfile 작성하기
Docker image 생성을 위해 Dockerfile을 작성해 줍니다.
```yml
FROM node:16.15.1
RUN mkdir -p /app
WORKDIR /app
COPY package*.json /app/
RUN npm install
COPY . /app/
EXPOSE 3000

CMD ["npm", "run", "start:dev"]
```

## (2) docker-compose.yml
데이터베이스는 mysql을 사용하였습니다.

```yml
version: '3'

services:
  db:
    image: mysql:5.7
    container_name: mysql
    env_file: ./.env
    volumes:
      - ./database/data:/var/lib/mysql
    ports:
      - 3307:3306
    environment:
      MYSQL_ROOT_PASSWORD: mysql_password
      MYSQL_DATABASE: database_name
    networks:
      - server

  web:
    image: ci-cd-prac
    container_name: web_server
    env_file: ./.env
    restart: always
    ports:
      - 3333:3000
    depends_on:
      - db
    networks:
      - server

networks:
  server:
```

## (3) docker-ci.yml
master 브랜치에 해당 이벤트가 발생하면 도커허브에 로그인하여 변경사항이 반영된 이미지를 생성해 올리도록 작업을 설정해 줍니다.
여기서 사용하는 변수들은 repository secret 변수에 등록해주어야 사용이 가능합니다. (CD 뒷 부분에 방법 참고!)
```yml
name: Docker Image CI

on:
  push:
  pull_request_target:
    branches: [master]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKER_IMAGE_NAME }}:latest
```


# 지속적 배포: CD(Continuous Deployment)
배포는 AWS를 활용하였습니다. (프리 티어!)

## (1) AWS EC2 인스턴스 생성

### (1) AMI 선택
AMI는 인스턴스를 시작하는 최초의 운영체제를 의미하는데, **ubuntu server 20.04 버전** 으로 설정했습니다.

<img src="https://velog.velcdn.com/images/chaerim1001/post/99a65f6a-66fa-4035-a587-9560da8b8316/image.png"  alt="aws_ec2_ami">

### (2) 키페어 생성
``` .pem ``` 파일 형식으로 키 페어를 생성합니다. 
네트워크나 다른 설정은 기본 값으로 두고 진행하였습니다.

<img src="https://velog.velcdn.com/images/chaerim1001/post/2a17fe06-576f-49ee-9565-cca0093cef5a/image.png" alt="aws_key">

### (3) 보안 그룹 구성
인스턴스를 생성한 뒤에는 인스턴스 상세 화면에서 보안 그룹을 확인할 수 있습니다.
보안 그룹을 클릭하여 인바운드 규칙을 추가해 줍니다.

<img src="https://velog.velcdn.com/images/chaerim1001/post/866b4eb0-59d3-43cf-95e7-9e14f2413f57/image.png" alt="security_group" >

기본으로 ssh 연결을 위해 22 포트가 설정되어 있는데 필요한 포트를 추가적으로 열어줘야 접근이 가능합니다. (지금은 개인 실습이니..ㅎㅎ 원래는 ip는 제한적으로만 열어줘야 합니디.)

<img src="https://velog.velcdn.com/images/chaerim1001/post/8dbf13a4-2777-4d56-8e3d-34e25502bf16/image.png" alt="security_group_2"> 

### (4) 고정 IP 설정
인스턴스를 중지했다가 다시 시작하면 IP가 계속 바뀌기 때문에 고정적인 IP 사용을 위해 탄력적 IP (Elastic) IP를 설정해 줍니다.

<img src="https://velog.velcdn.com/images/chaerim1001/post/d79cf830-ca6f-4bdc-af69-74d1eb3ab479/image.png" alt="elastic_ip">

IP 할당 후 생성된 IP로 들어가 앞서 생성한 인스턴스를 연결해주면 됩니다.

<img src="https://velog.velcdn.com/images/chaerim1001/post/8a6a0ae6-b4e6-416a-9b21-ac20a9644ab7/image.png" alt="elastic_ip_2">

_인스턴스를 더이상 사용하지 않아 중지할 때에는 IP도 함께 릴리즈해야 합니디!!_

## (2) 인스턴스 접속하기
EC2로 인스턴스에 ssh를 사용하여 접속해 봅시다!

<img src="https://velog.velcdn.com/images/chaerim1001/post/26babcad-7ec8-4758-84f0-e008b58da87e/image.png" alt="connect">

생성한 인스턴스의 Connect 탭으로 들어가 SSH Client 부분에서 자신의 환경에 맞는 명령어를 확인할 수 있으니 그대로 진행하면 됩니다.

## (3) Docker 설치
접속한 서버에 docker를 설치해 줍니다.

```console
$ sudo apt-get update

$ sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release

$ sudo mkdir -p /etc/apt/keyrings

$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

$ echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu / $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$ sudo apt-get update

$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

이후 docker 명령을 입력하면 docker 명령어들이 나오는데 그렇게 나오면 docker가 잘 설치된 것입니다.

## (4) Github actions 설정

### (1) build-and-deploy.yml 작성
```yml
name: build and deploy

on:
  push:
    branches: [master]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKER_IMAGE_NAME }}:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: ssh connect & production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          script: |
            cd project_dir
            docker system prune -a --volumes -f
            docker compose pull -q
            docker compose up --force-recreate --build -d --quiet-pull 2>log.out
            cat log.out
```

master 브랜치에 push 이벤트가 발생했을 때 해당 작업을 수행하도록 워크플로우를 작성해 줍니다.

### (2) ssh password 생성

ssh 배포에 [appleboy](https://github.com/appleboy/ssh-action) 를 활용하는데 private key나 password를 사용해 ssh 접속이 가능합니다. 저는 password를 사용해 접속하는 방법을 선택했기 때문에 비밀번호를 생성해준 뒤 repository의 secret 변수로 등록해 줍니다.

ssh로 접속해서 아래의 명령을 통해 password 설정이 가능합니다.

```console
$ sudo passwd <username>
```

password를 사용한 접속을 허용하기 위해 sshd_config 파일에 **PasswordAuthentication** 을 **yes**로 변경해 줍니다.

```console
$ sudo vi /etc/ssh/sshd_config
```

<img src="https://velog.velcdn.com/images/chaerim1001/post/8673e6bf-ab9d-4c12-984f-d46fb9b56b53/image.png" alt="ssh">

이후 restart해주면 적용이 됩니다.
```console
$ service ssh restart
```

### (3) secret 변수 설정
yml에서 사용한 secret 변수들을 설정해 줍니다.
repository의 **settings/security/secrets/actions** 탭에 들어가서 변수를 추가할 수 있습니다.

<img src="https://velog.velcdn.com/images/chaerim1001/post/2ab9657a-9d6d-417d-9371-7281517fed66/image.png" alt="github_actions">

* host: ip 주소
* username: ssh의 username
ec2 인스턴스는 생성할 때 선택한 AMI에 따라 [username](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs.html)이 다르니 공식 문서를 참조해 사용하면 됩니다.
* password: 위에서 생성해준 ssh의 password
* port: ssh 접속 포트번호 (기본으로 ssh 접속을 위해 열려있는 포트 번호는 22다)

## (5. Docker 권한 문제)
윗 단계까지 진행하고 push를 하면

```
err: Got permission denied while trying to connect to the Docker daemon socket 
```

과 같은 에러가 발생할 수 있습니다.
이는 docker group에 해당 유저가 없어서 생기는 에러인데 docker group에 추가만 해주면 바로 해결됩니다.

### (1) ssh로 접속해서 docker group이 없을 경우에는 생성해준다.
```console
$ sudo groupadd docker
```

### (2) docker group 해당 유저 추가
```console
$ sudo usermod -aG docker $USER
```

### (3) 변경사항 적용
```console
$ newgrp docker
```

>참고 <br>
>https://docs.docker.com/engine/install/ubuntu/ <br>
>https://github.com/appleboy/ssh-action <br>
>https://velog.io/@combi_jihoon/Docker-AWS-EC2-%EB%9D%84%EC%9A%B0%EA%B8%B0 <br>
>https://goodgid.github.io/Github-Action-CI-CD-AWS-EC2/ <br>
>https://seulcode.tistory.com/557 <br>
