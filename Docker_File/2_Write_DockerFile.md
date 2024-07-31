# Dockerfile 작성

## Dockerfile 명령어
- Dockerfile는 한 줄이 하나의 명령어가 된다.
- 명령어를 명시한 뒤에 옵션을 추가하는 방식이다.
- 명령어를 소문자로 표기해도 상관없지만 일반적으로 대문자로 표기한다.
- Docker's Instruction
    - FROM 
        - 생성할 이미지의 베이스가 될 이미지. 
        - 반드시 한 번 이상 입력해야 하며, 이미지 이름 포맷은 docker run 명령어에서 이미지 이름 입력 포맷과 동일함
    - MAINTAINER
        - 이미지를 생성한 개발자의 정보를 나타냄
        - 일반적으로 Dockerfile을 작성한 사람과 연락할 수 있는 이메일 등을 입력
        - 단 MAINTAINER는 도커 1.13.0 버전 이후로 사용하지 않고, 대신 아래와 같이 사용
        - ex) LABEL maintainer "covy <covy@gmail.com>"
    - LABEL
        - 이미지에 메타데이터를 추가
        - 메타데이터는 'key:value' 형태로 저장되며, 여러 개의 메타데이터가 저장될 수 있다.
        - 추가된 메타데이터는 docker inspect 명령어로 이미지의 정보를 구해서 확인할 수 있다.
    - RUN 
        - 이미지를 만들기 위해 컨테이너 내부에서 실행해야하는 명령어를 정의
        - 이미지를 빌드할 때 별도의 입력을 받아야 하는 RUN이 있다면 build 명령어는 이를 오류로 간주(예: apt-get install -y apache2)
    - ADD
        - 파일을 이미지에 추가
        - 추가하는 파일은 Dockerfile이 위치한 디렉토리인 Context에서 가져온다.
        - JSON 배열의 형태로 ["추가할 파일 이름" "컨테이너에 추가될 위치"]와 같이 사용
    - WORKDIR 
        - 명령어를 실행할 디렉토리를 나타냄
        - Bash 쉘에서 cd 명령어를 입력하는 것과 같은 기능을 한다.
    - EXPOSE
        - Dockerfile의 빌드로 생성된 이미지에서 노출할 포트를 설정
        - 그러나 EXPOSE를 설정한 이미지로 컨테이너를 생성했다고 해서 반드시 이 포트가 호스트의 포트와 바인딩 되는 것은 아니며, 단지 컨테이너의 80번 포트를 사용할 것임을 나타낸다.
        - EXPOSE는 컨테이너를 생성하는 run 명령어에서 모든 노출된 컨테이너의 포트를 호스트에 Publish하는 -P 플래그(flag)와 함께 사용된다.
    - CMD
        - CMD는 컨테이너가 시작될 때마다 실행할 명령어를 설정
        - Dockerfile에서 한 번만 사용할 수 있다.
        - CMD의 입력은 JSON 배열 형태인 ["실행 가능 파일", "명령줄 인자1", "명령줄 인자2" ..] 형태로도 사용 가능하다.

## Dockerfile을 사용하기 위한 간단한 시나리오 : 웹 서버 이미지를 생성하는 예

```bash
mkdir dockerfile && cd dockerfile
echo test >> test.html

vi Dockerfile
```

```docker
FROM ubuntu:14.04
LABEL maintainer "covy <covy@gmail.com>"
LABEL "purpose"="practice"
RUN apt-get update
RUN apt-get install apache2 -y
ADD test.html /var/www/html
WORKDIR /var/www/html
RUN ["/bin/bash", "-c", "echo hello >> test2.html"]
EXPOSE 80
CMD apachectl -DFOREGROUND
```
> 도커 엔진은 Dockerfile을 읽어 들일 때 기본적으로 현재 디렉토리에 있는 Dockerfile 이라는 이름을 가진 파일을 선택한다.

> RUN ["/bin/bash", "-c", "echo hello >> test2.html"]은 /bin/bash 쉘을 이용해 'echo hello >> test2.html'을 실행한다는 뜻이다. 이처럼 Dockerfile의 일부 명령어는 RUN ["실행가능파일", "명령어1", "명령어2" ..]와 같은 방법으로 사용할 수 있다.

> WORKDIR 명령어를 여러 번 사용하면 cd 명령어를 여러 번 사용한 것과 같다. 예를 들어 WORKDIR /var/www/html와 WORKDIR /var, WORKDIR www/html이 같음