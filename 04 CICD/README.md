# Github Actions
> 도커 이미지를 생성 후 EC2서버로 배포하는 시나리오


####  .githubs/workflows/deploy.yml
> 깃허브 원격 레포지토리에 반영, main 브랜치로 merge 되는 순간 트리거됨

```yaml
name: Deploy to Amazon EC2
on:
  push:
    branches: [ "main" ] # main 브랜치 merge 트리거 되었을 경우

env:
  REGISTRY: ghcr.io # 깃허브 컨테이너 저장소를 사용한다.
  IMAGE_NAME: yonggyo1125/auth-service
  VERSION: latest

jobs:
  Deploy:
    name: Build Docker Image and Deploy EC2 #도커 이미지를 만들고 EC2에 배포
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write
    environment: deploy
    steps:
      - name: Checkout source code # 소스코드 체크 아웃
        uses: actions/checkout@v4
      - name: Set up JDK # JDK 지정
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: gradle
      - name: Build container image # 도커 이미지 생성
        run: |
          chmod +x gradlew
          ./gradlew bootJar
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.VERSION }} .
      - name: Log into container registry # 깃허브 이미지 컨테이너 로그인 및 push
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Publish container image
        run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.VERSION }}

      - name: Executing remote ssh commands # AWS EC2 서버 SSH로 접속하고 deploy.sh 실행
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.REMOTE_IP }} # 인스턴스 IP
          username: ${{ secrets.REMOTE_USER }} # 우분투 아이디
          key: ${{ secrets.REMOTE_PRIVATE_KEY }} # ec2 instance pem key
          port: ${{ secrets.REMOTE_SSH_PORT }} # 접속포트
          script: | # 실행할 스크립트
            sh deploy.sh
```


### deploy.sh
> AWS EC2 서버의 /home/ubuntu 경로에 deploy.sh 생성<br>/home/ubuntu/deploy.sh

```shell
#!/bin/sh
sudo docker stop auth-service 2> /dev/null
sudo docker rm auth-service 2> /dev/null
sudo docker rmi ghcr.io/yonggyo1125/auth-service 2> /dev/null
sudo docker pull ghcr.io/yonggyo1125/auth-service 2> /dev/null
sudo docker run -d --name auth-service -p 3003:3003 ghcr.io/yonggyo1125/auth-service
```

- EC2 서버에는 도커가 설치되어 있어야 한다. [Docker 설치 참고](https://github.com/yonggyo1125/lecture_cicd)


# Jenkins
