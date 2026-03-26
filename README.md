# 도커 이미지 스터디

> 우리 FISA 6기 클라우드 엔지니어링
도커 이미지 최적화 전략과 공유 실습
> 

## 1. 팀원 소개

| <img src="https://avatars.githubusercontent.com/u/181299322?v=4" width="120" height="120" /> | <img src="https://avatars.githubusercontent.com/u/113874212?v=4" width="120" height="120" /> |
|:---:|:---:|
| **신성혁**<br>[@ssh221](https://github.com/ssh221) | **사재헌**<br>[@Zaixian5](https://github.com/Zaixian5) |
| springboot 앱 개발 | bash 스크립트 작성 |


## 2. 프로젝트 목표

도커는 애플리케이션 단위로 실행 환경을 격리시켜, 가상화보다 빠른 성능과 보안성을 제공해주는 컨테이너 기술이다. 특히, 다른 환경에서도 애플리케이션의 동일한 동작을 보장하기 위해 컨테이너와 애플리케이션의 모든 설정을 이미지화하여 배포할 수 있다는 것이 도커의 큰 장점이다.

이번 실습에서는 도커의 이미지 기능을 사용하는 방법에 대해 중점적으로 학습하였다. 특히 이미지 배포 및 실행 시 성능 보장을 위한 최적화 전략에 초점을 맞춰, 도커를 활용한 현대 개발 환경의 흐름과 효율적인 DevOps 전략에 대해 이해해보고자 한다.

## 3. 기술 스택

- OS: Ubuntu 24.04
- 컨테이너: Docker
- 애플리케이션: Spring Boot, JSP
- IDE: Spring Toos Suit

## 4. 프로젝트 구성

```
+-----------------------+      +-----------------------+      +-----------------------+
|    1. Development     |      |   2. Build (Gradle)   |      |     3. Dockerize      |    
+-----------------------+      +-----------------------+      +-----------------------+ 
|                       |      |                       |      |                       | 
|   [ STS IDE ]         |      |   [ JAR File ]        |      |   [ Dockerfile ]      |  
|   - Spring Boot App   | ===> |   - ./gradlew build   | ===> |   - docker build      |
|   - JSP (Text View)   |      |   - build/libs/*.jar  |      |   - Image Optimization| 
|                       |      |                       |      |                       |
+-----------------------+      +-----------------------+      +-----------------------+  
        (Local)                        (Local)                     (Local / CI)                
```

1. Development
    - 간단한 텍스트를 보여주는 웹 애플리케이션 개발
        - 프론트엔드: JSP
        - 백엔드: Spring Boot
        - IDE: Spring Tool Suit
2. Build
    - Gradle로 빌드하여 JAR 파일 생성
        - Docker 이미지 빌드에 사용
3. Dockerize
    - Dockerfile과 함께 JAR 파일을 실행하는 컨테이너 이미지 생성
        - 이미지 최적화 전략 조사
        - 최적화 전략을 적용한 Dockerfile과 적용하지 않은 Dockerfile간 이미지 용량 비교
4. Deploy & Run
    - 생성한 이미지를 Docker Hub에 push
    - 다른 컴퓨터에 pull하여 동일한 실행결과를 보이는지 확인

## 4. 이미지 최적화 전략

### 1) 도커 이미지 최적화가 중요한 이유

1. 배포 및 스케일링 속도 향상
    - **빠른 Pull/Push:** 이미지 용량이 작을수록 도커 레지스트리(Docker Hub, AWS ECR 등)에 업로드하거나, 서버로 다운로드하는 시간이 크게 단축된다.
    - **신속한 오토스케일링:** 트래픽이 폭주하여 급하게 컨테이너를 늘려야 할 때, 가벼운 이미지는 네트워크 전송과 압축 해제가 빨라 훨씬 신속하게 실행(Running) 상태로 전환할 수 있다.
2. 인프라 유지 비용 절감
    - **저장 공간 절약:** 누적되는 이미지 버전들이 차지하는 레지스트리 보관 비용과 호스트 서버의 디스크 사용량을 크게 줄일 수 있다.
    - **네트워크 대역폭 절감:** 수십 대의 서버로 이미지를 배포할 때 발생하는 네트워크 트래픽 요금을 최소화할 수 있다.

### 2) 이미지 용량 최적화를 위해 고려해야 할 것

1. Base Image 선택
    - 도커 컨테이너를 만들기 위한 최소한의 운영체제 환경과 실행 도구가 담긴 템플릿
    - Image Varient
        
        | 종류 | 설명 | 특징 |
        | --- | --- | --- |
        | Full (Standart) | 표준 리눅스의 모든 도구가 포함된 기본 이미지 | 호환성이 좋지만, 불필요한 도구가 많아 용량이 매우 큼 |
        | Slim | 표준 리눅스 환경을 유지하되, 실행에 꼭 필요하지 않은 문서나 캐시 파일만 제거한 다이어트 버전 | Full보다 가볍지만, 표준 라이브러리를 그대로 써서 안정적임 |
        | Alpine | 실행만을 위해 극도로 경량화된 Alpine Linux 기반 이미지 | 가장 용량이 작으며, 보안 노출 면적이 좁아 안전함 |
    - 실습 과정에서는 Base 이미지를 용량이 큰 `eclipse-temurin:17-jdk` 이미지 파일과 작은 `eclipse-temurin:17-jre-alpine`으로 각각 구성하여 만들어지는 이미지의 용량을 비교해보았음
2. JAR Layer
    - 스프링 부트는 JAR 내부를 아래 4가지 계층으로 구분함
    
    | **레이어 명칭** | **포함 내용** |
    | --- | --- |
    | **dependencies** | 일반 라이브러리 (Spring, Hibernate 등) |
    | **spring-boot-loader** | JAR를 실행하기 위한 로더 관련 코드 |
    | **snapshot-dependencies** | 개발 중인 SNAPSHOT 라이브러리 |
    | **application** | **내 소스코드, 설정, JSP 등** |
    - 이미지 빌드시 전체 JAR를 한 번에 빌드하지 않고
        
        `java -Djarmode=layertools -jar app.jar extract` 명령으로 나눠 빌드
        
        - Docker는 각각의 레이별로 빌드된 캐시를 저장
        - 추후 소스코드가 수정되어 재 빌드시 Docker는 각각의 캐시를 재사용할 수 있어 성능 향상에 도움이 됨
        - 나눠진 레이어를 빌드할 때는
            
            `java -jar app.jar` 이 아니라, JarLancher를 사용하여
            
            `java org.springframework.boot.loader.launch.JarLauncher` 로 빌드
            
        
        ```
        예시:
        
        소스 코드의 수정 ->
        build.gradle은 수정되지 않았음(dependencies 레이어 변경 없음) ->
        docker build시 dependencies 레이어 캐시 재사용 ->
        빌드 성능 향상!
        ```
        
3. 멀티 스테이지 빌드(Multi-stage Build)
    - 하나의 Dockerfile안에 여러 `FROM` 구문을 사용하여 빌드 환경과 실행 환경을 분리하는 방식
    - 빌드 과정에서만 필요한 도구(컴파일러, SDK 등)는 첫 번째 단계에서 사용하고, 최종 결과물(컴파일된 바이너리, 정적 파일 등)만 가벼운 실행 용 이미지로 복사하여 이미지 크기를 획기적으로 줄임
    
    ```docker
    예시:
    
    # 빌드 단계
    FROM openjdk:17 AS builder
    COPY . /usr/src/myapp
    WORKDIR /usr/src/myapp
    RUN javac Main.java
    
    # 런타임 단계
    FROM openjdk:17-jre
    WORKDIR /usr/src/myapp
    COPY --from=builder /usr/src/myapp /usr/src/myapp
    CMD ["java", "Main"]
    ```

## 5. 이미지 최적화 실습

### 1) 최적화를 적용하지 않은 Dockerfile(Dockerfile.bad)

- 최적화를 고려하지 않은 Dockerfile 생성 script

```jsx
# 1. 무겁고 불필요한 도구가 모두 포함된 전체 JDK 사용
FROM eclipse-temurin:17-jdk

# 2. 디렉토리 선정
WORKDIR /app

# 3. JAR 파일을 한 번에 복사
COPY docker-opt-web.jar app.jar

# 스프링 부트 앱을 실행하는 포트
EXPOSE 80

# 4. JAR 파일 통째로 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- Dockerfile 구성 요소 설명

| 단계 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 명령어 | 설명 | 문제점 |
| --- | --- | --- | --- |
| 1. 베이스 이미지 | `FROM eclipse-temurin:17-jdk` | 자바 17 실행 환경, 컴파일러, 디버깅 도구 등이 모두 포함된 전체 JDK 이미지를 기반으로 시작 | jdk에는 jar를 실행하기 위한 기능 뿐만 아니라, 불필요한 개발 도구도 포함되어 있어 용량을 많이 차지함 또한 스테이지를 구분하지 않고 `FROM`절 하나만으로 빌드하여 성능 문제 발생 가능 |
| 2. 작업 디렉토리 | `WORKDIR /app` | 명령어들이 실행될 컨테이너 내부의 기본 폴더를 `/app`으로 지정  |  |
| 3. 파일 복사 | `COPY [파일명].jar app.jar` | 빌드된 JAR 파일을 `/app/app.jar`로 복사 | jar 실행을 레이어 별로 구분하기 않고 통째로 복사하게 되어 성능이 저하될 수 있음 |
| 4. 포트 설정 | `EXPOSE 80` | 컨테이너가 80번 포트를 사용 |  |
| 5. 실행 명령 | `ENTRYPOINT ["java", "-jar", "app.jar"]` | 컨테이너가 시작될 때 `java -jar` 명령어를 통해 JAR 파일을 직접 실행 | jar 실행을 레이어 별로 구분하고 실행하지 않고 통째로 실행하므로 성능이 저하될 수 있음 |

### 2) 최적화 전략 별 비교

> Dockerfile.bad와 아래 각각의 최적화 전략을 개별적으로 적용한 Dockerfile을 사용하여 전략별로 최적화 효율을 비교해 보고자 한다.
> 
1. Base Image 선정
    - 용량이 큰 `eclipse-temurin:17-jdk` 이미지와 작은 `eclipse-temurin:17-jre-alpine`으로 각각 구성하여 만들어지는 이미지의 용량을 비교해보았음
    
    ```docker
    FROM eclipse-temurin:17-jre-alpine
    
    WORKDIR /app
    
    COPY docker-opt-web.jar app.jar
    
    EXPOSE 80
    
    ENTRYPOINT ["java", "-jar", "app.jar"]
    ```
    
    | 명령어 | 설명 |
    | --- | --- |
    | `FROM eclipse-temurin:17-jre-alpine` | 자바 17 Jre 이미지를 가져옴. 가장 용량이 적은 alphine 버전을 베이스 이미지로 사용하여 일반 jdk만 사용한 이미지와 비교 |
    - 측정 결과
    
2. 멀티 스테이징 + JAR Layer 분해
    - 빌드 스테이지에서 JAR를 4개의 Layer로 분해하고, 실행 스테이지로 필요한 Layer만 복사해옴으로써 빌드 속도와 이미지의 최적화를 시도함
    
    ```jsx
    # --- 1단계: 빌드 스테이지 (레이어 추출) ---
    FROM eclipse-temurin:17-jdk AS builder
    WORKDIR /builder
    
    # 실제 파일명 적용
    COPY docker-opt-web.jar app.jar
    RUN java -Djarmode=layertools -jar app.jar extract
    
    # --- 2단계: 실행 스테이지 (레이어별 복사) ---
    FROM eclipse-temurin:17-jdk
    WORKDIR /app
    
    # 분해된 레이어를 순서대로 복사
    COPY --from=builder /builder/dependencies/ ./
    COPY --from=builder /builder/spring-boot-loader/ ./
    COPY --from=builder /builder/snapshot-dependencies/ ./
    COPY --from=builder /builder/application/ ./
    
    EXPOSE 8080
    
    # 일반적인 실행 방식 (분해된 상태라 하더라도 내부 JAR 로더 사용)
    ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
    ```
    
    - Dockerfile 구성 요소 설명
    
    | 스테이지 | 명령어 | 설명 |
    | --- | --- | --- |
    | 빌드 스테이지 (레이어 추출) | `FROM eclipse-temurin:17-jdk AS builder` | 자바 17 JDK 이미지를 가져오며, 이 단계를 `builder`라는 이름으로 부르겠다고 정의 |
    |  | `RUN java -Djarmode=layertools -jar app.jar extract` | 스프링 부트의 `layertools`를 사용하여 `app.jar` 내부를 성격에 따라 4개(의존성, 로더, 스냅샷, 애플리케이션)의 레이어 폴더로 추출 |
    | 실행 스테이지 (레이어별 복사) | `FROM eclipse-temurin:17-jdk` | 빌드 스테이지의 무거운 JDK 찌꺼기들을 버리고 깨끗한 상태로 다시 시작 |
    |  | `COPY --from=builder /builder/레이어이름/ ./` |   • 빌드 스테이지에서 쪼개놓은 4개의 폴더를 하나씩 가져옴<br>• 거의 변하지 않는 `dependencies`(의존성)를 먼저 복사하고, 가장 자주 변하는 `application`(내 코드)을 마지막에 복사하여 도커 캐시의 효율을 극대화 |
    |  | `EXPOSE 8080`  | 컨테이너가 8080번 포트 사용 |
    |  | **`ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]`** | 압축된 JAR를 통째로 실행하는 대신, 이미 풀려있는 클래스들을 `JarLauncher`를 통해 즉시 실행 |
   

### 3) 모든 최적화 전략을 적용한 Dockerfile(Dockerfile.good)

- 모든 최적화 방식을 적용한 Dockerfile

```jsx
# 1. 멀티 스테이지 적용
# --- [Stage 1: Builder] JAR 파일에서 레이어를 추출하는 단계 ---
# 2. 경량 이미지 사용
FROM eclipse-temurin:17-jre-alpine AS builder
WORKDIR /build

# JAR 파일 복사
COPY docker-opt-web.jar app.jar

# 3. jar 파일을 레이어 별로 분해
# Spring Boot의 layertools를 사용해 JAR 내부를 각 계층별로 분해(extract)
RUN java -Djarmode=layertools -jar app.jar extract

# --- [Stage 2: Runtime] 실제 실행될 경량 이미지 단계 ---
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# 3. jar 파일을 레이어 별로 분해
# 변동 가능성이 가장 '낮은' 것부터 '높은' 순서로 복사 (캐싱 극대화)
COPY --from=builder /build/dependencies/ ./
COPY --from=builder /build/spring-boot-loader/ ./
COPY --from=builder /build/snapshot-dependencies/ ./
COPY --from=builder /build/application/ ./

EXPOSE 80

# JarLauncher 사용
# 분리된 레이어를 조합하여 애플리케이션 실행
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

## 6. 측정 결과 비교

| 최적화 적용 방식 | 이미지 용량 | 빌드 실행 시간 |
| --- | --- | --- |
| 최적화 적용 x | 216 MB | 2.2s |
| Alpine 베이스 이미지  | 85.7 MB | 2.0 s |
| 멀티 스테이징 + Layer 분해 | 216 MB | 1.5 s |
| Alpine 베이스 이미지 + 멀티 스테이징 + Layer 분해 | 85.7 MB | 1.4 s |

## 7. 최종 정리

> **simpleweb:bad → 최적화 되지 않은 이미지
simpleweb:good → 최적화된 이미지**
> 

<img width="1115" height="183" alt="전체비교" src="https://github.com/user-attachments/assets/a0754d6f-a4b0-454f-84ce-b01efd3740f4" />


멀티스테이징, Layer 분해, Base 이미지 선정 등의 최적화 방식을 사용하여 실험을 진행하였다. 실험 결과, 빌드 실행 시간에 큰 영향을 준 것은 멀티스테이징과 Layer 분해 방식이었고, 이미지 용량 최적화에 영향을 준 최적화 방식은 Base 이미지를 Alpine으로 설정한 것이었다.
