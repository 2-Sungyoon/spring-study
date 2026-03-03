# 📅 [week 6] 주제: 

## 🎯 학습 목표
- Build 기반과 Image 기반 배포 구조의 차이를 이해한다.
- CI 중심 배포에서 이미지 태그 전략과 재배포 방식의 의미를 설명할 수 있다.
- Dockerfile 경량화 및 backend만 교체하는 배포 개선 이유를 이해한다.

---

## 📝 주요 개념 정리
### 1. 기존 / 현재 CICD workflow 파일 비교
**캡스톤 1 당시 CICD workflow 파일** 

```yaml
name: Phomate CI/CD

on:
  push:
    branches: ["develop"]   # develop에 push될 때마다 배포

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. 코드 가져오기
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. JDK 17 세팅
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      # 3. gradlew 실행 권한 주기
      - name: Grant execute permission for Gradle wrapper
        run: chmod +x ./gradlew

      # 4. Spring Boot 빌드 (테스트 생략)
      - name: Build JAR without tests
        run: ./gradlew clean build -x test --no-daemon

      # 5. Docker Hub 로그인
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      # 6. Docker 이미지 빌드 & 푸시
      - name: Build & Push Docker Image
        run: |
          IMAGE_NAME=${{ secrets.DOCKERHUB_USERNAME }}/phomate-backend
          docker build -t $IMAGE_NAME:dev .
          docker tag $IMAGE_NAME:dev $IMAGE_NAME:latest
          docker push $IMAGE_NAME:dev
          docker push $IMAGE_NAME:latest

      # 7. EC2에 접속 → compose로 재배포
      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }} # 탄력적 IP
          username: ubuntu
          key: ${{ secrets.EC2_KEY }}  # PEM 전체 복사
          script: |
            cd /home/ubuntu
            IMAGE_NAME=${{ secrets.DOCKERHUB_USERNAME }}/phomate-backend:dev
            sudo docker pull $IMAGE_NAME
            sudo docker-compose down || true
            sudo docker-compose up -d
            sudo docker image prune -f원재 CICD 배포 파일
```

**캡스톤 2 CICD workflow 파일**

```yaml
name: Phomate CI/CD

on:
  push:
    branches: ["develop"]   # develop에 push될 때마다 배포

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. 코드 가져오기
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. JDK 17 세팅
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          # gradle 캐시 추가
          cache: gradle

      # 3. gradlew 실행 권한 주기
      - name: Grant execute permission for Gradle wrapper
        run: chmod +x ./gradlew

      # 4. Spring Boot 빌드 (테스트 생략)
      - name: Build JAR without tests
        run: ./gradlew clean build -x test --no-daemon

      # 5. Docker Hub 로그인
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      # 6. Docker 이미지 빌드 & 푸시
      - name: Build & Push Docker Image
        run: |
          IMAGE_NAME=${{ secrets.DOCKERHUB_USERNAME }}/phomate-backend
          SHA_TAG=${GITHUB_SHA::12}
          docker build -t $IMAGE_NAME:dev -t $IMAGE_NAME:$SHA_TAG .
          docker push $IMAGE_NAME:dev
          docker push $IMAGE_NAME:$SHA_TAG
      # CI 로그에 태그 찍는 용도
      - name: Show image info
        run: |
          docker images | grep phomate-backend || true
          echo "SHA deployed: ${GITHUB_SHA::12}"
      # 7. EC2에 접속 → compose로 재배포
      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }} # 탄력적 IP
          username: ubuntu
          key: ${{ secrets.EC2_KEY }}  # PEM 전체 복사
          script: |
            cd /home/ubuntu/PhoMate_BE
            ~~IMAGE_NAME=${{ secrets.DOCKERHUB_USERNAME }}/phomate-backend:dev~~
            sudo docker compose pull backend
            # docker compose / docker-compose 호환
            sudo docker compose up -d --no-deps --force-recreate backend
            sudo docker image prune -f
```

### 변경 사항

1. **gradle 캐시 추가**

```yaml
- name: Set up JDK 17
  uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: gradle
```

- gradle 캐시 추가
- 매번 의존성 재다운로드 방지 → 빌드 속도 향상

1. **이미지 태그 방식 변경**

```bash
docker build -t $IMAGE_NAME:dev .
docker tag $IMAGE_NAME:dev $IMAGE_NAME:latest
docker push $IMAGE_NAME:dev
docker push $IMAGE_NAME:latest
```

```bash
IMAGE_NAME=${{ secrets.DOCKERHUB_USERNAME }}/phomate-backend
SHA_TAG=${GITHUB_SHA::12}

docker build -t $IMAGE_NAME:dev -t $IMAGE_NAME:$SHA_TAG .
docker push $IMAGE_NAME:dev
docker push $IMAGE_NAME:$SHA_TAG
```

- latest 제거
- SHA 태그 추가
- 어떤 커밋이 배포됐는지 추적 가능

1. **ec2 배포 방식 변경**

```bash
cd /home/ubuntu

sudo docker pull $IMAGE_NAME
sudo docker compose down || true
sudo docker compose up -d
```

문제점 : 

- cd /home/ubuntu :
    - compose 파일을 탐색하는 위치가 애매함
- pull $IMAGE_NAME :
    - 이미지명으로 pull 받음
    - 태그 전략이 수정되면 `compose.yml`, `스크립트`에서 모두 변경해줘야함
- docker compose down || true
    - 전체 서비스가 재시작함 → 시간 낭비

→ down time 발생 가능

```bash
cd /home/ubuntu/PhoMate_BE

sudo docker compose pull backend
sudo docker compose up -d --no-deps --force-recreate backend
sudo docker image prune -f
```

해결

- cd /home/ubuntu/PhoMate_BE
    - 정확히 원하는 프로젝트 안에서 탐색하도록 명시
- pull backend :
    - 서비스명으로 pull 받음
    - `compose.yml`에 정의된 backend 서비스명 기준으로 pull
- —no-deps :
    - backend만 재시작
    - DB와 같이 다른 서비스에 영향 X
- —force-recreate :
    - 새 컨테이너로 강제 교체
    - 이걸 안하니까, 새로운 이미지를 생성해도 컨테이너가 그걸로 안 돌아가짐

→ 전체 down 없이 backend만 교체

→ 낭비 줄이고, 실패했을 때 영향이 적음

1. **Dockerfile 변경**

Dockerfile의 역할 : **`코드 → EC2로 올라감 → EC2에서 Dockerfile로 빌드 → 실행`**

```docker
FROM eclipse-temurin:17-jdk-jammy
```

- `JDK` 기반

```docker
FROM eclipse-temurin:17-jre-jammy
```

- `JRE` 기반으로 변경 → `경량화`
- 컨테이너 내부에서 build X
- CI에서 jar 생성 후 복사만

<aside>

**⭐️ 추가로.. 멀티스테이지 빌드란?**

- **`멀티스테이지 빌드 = 빌드용 이미지 + 실행용 이미지 분리`**
    - Dockerfile 안에 FROM이 2번 이상 있는 것
    - 빌드 단계에서 만든 결과물(jar)만 실행 단계 이미지로 복사하는 방식
    - 최종 실행 이미지를 작게, 안전하게, 단순하게 만들기 위함
    

**그럼 왜 Phomate는 멀티스테이지 빌드 방식을 쓰지 않았나?**

- Phomate 구조 : **`CI 빌드 + Dockerfile 실행`**
    - CI에서 진행시 빌드 환경이 더 안정적임
    - 빌드 실패를 빨리 알 수 있음
    - Dockerfile이 단순해짐 → 배포 속도 빠름
</aside>

### 배포 전략 비교 - Image vs Build

1. Image 기반(Phomate)
- `registry 빌드 기반`
    - 이미지 업데이트가 핵심
- 코드가 서버에 없어도 됨 → 어차피 이미지 기반이니까
- 비유 :
    - 서버 = 매장
    - 공장에서 완제품(이미지) 만들어서 진열만 함
- 확장성, 안정성 good
- 배포 빠름
1. Build 기반
- `server 빌드 기반`
    - 서버에서 직접 코드를 빌드하는 방식
- 코드가 서버에 있어야 함
- 비유 :
    - 서버 = 공장
    - 설계도(코드) 받아서 직접 제품(이미지) 생산
- 확장성, 안정성 상대적으로 bad
- 배포 느림(build도 필요)

---

## 💡 회고 및 질의응답
이전에 진행했던 다소니 프로젝트에서는 docker-compose.yml 에서 build 기반으로 진행했고, 캡스톤 Phomate에서는 image 기반으로 배포하였다.

이전 다소니에서는 서버에 코드를 반영할때 

- `git pull`
- `docker compose up -d —build spring-backend`

현재 Phomate에서는 

- `git pull`
- `docker compose pull backend`
- `docker compose up -d —force-recreate backend` 과정을 거쳐야 했다.

다소니가 절차가와 명령어가 더 단순하기도 하고, 새로운 코드를 반영할때마다 이미지를 만드는게 낭비가 아닌가 라는 생각이 들어서 build 기반의 배포가 더 효율적이라고 생각했다. 하지만 오히려 image 기반의 배포가 서버에 의존하지 않기 때문에 더 안정적이고, 사실 jar 파일 하나를 생성하는 것이라 부담이 되는 정도가 아니라는 것을 알게됐다. 

또, CICD 배포를 할 때 하나의 workflow 안에도 세부적인 job들이 있고 앞의 job들이 성공적으로 수행돼도 하나의 job이 후에 실패하면 전체 workflow의 실패로 띄우기 때문에 실패한 workflow에 가서 세부적인 job들의 흐름을 보는 것이 중요하다는 것을 알게됐다. 실제로 캡스톤 1때 CICD 배포가 전체적으로 문제가 있어서 workflow가 실패했다고 생각했는데, jar build & Docker Hub 로그인 & 이미지 build, push까지는 성공하고 Deploy to EC2(EC2에 배포하는 job) 단계, 즉, 실제 배포 단계에서만 문제가 있었음을 알았다.
