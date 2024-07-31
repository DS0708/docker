# 기타 Dockerfile 명령어

## ENV, VOLUME, ARG, USER
- ENV
    - Dockerfile에서 사용될 환경변수를 지정한다. 
    - 설정한 환경변수는 \$\{ENV_NAME\} 또는 \$EVN_NAME의 형태로 사용할 수 있다. 
    - 이 환경변수는 Dockerfile뿐 아니라 이미지에도 저장되므로 빌드된 이미지로 컨테이너를 생성하면 이 환경변수를 사용할 수 있다. 
    - 다음 Dockerfile에서는 test라는 환경변수에 /home이라는 값을 설정했다.
        ```
        # vi Dockerfile
        FROM ubuntu:14.04
        ENV test /home
        WORKDIR $test
        RUN touch $test/mytouchfile
        ```
    - run 명령어에서 -e 옵션을 사용해 같은 이름의 환경변수를 사용하면 기존의 값은 덮어 쓰여진다.
    
- VOLUME
    - 빌드된 이미지로 컨테이너를 생성했을 때 호스트와 공유할 컨테이너 내부의 디렉토리를 설정한다.
    - VOLUME ["/home/dir", "home/dir2"]처럼 JSON 배열의 형식으로 여러 개를 사용하거나 VOLUME /home/dir /home/dir2로도 사용할 수 있다.
    - 다음 예시는 컨테이너 내부의 /home/volume 디렉토리를 호스트와 공유하도록 설정한다.
        ```
        # vi Dockerfile
        FROM ubuntu:14.04
        RUN mkdir /home/volume
        RUN echo test >> /home/volume/testfile
        VOLUME /home/volume
        ```

- ARG
    - build 명령어를 실행할 때 추가로 입력을 받아 Dockerfile 내에서 사용될 변수의 값을 설정
    - 다음 Dockerfile은 build 명령어에서 my_arg와 my_arg_2라는 이름의 변수를 추가로 입력받을 것이라고 ARG를 통해 명시한다.
    - ARG의 기본 값은 기본적으로 build 명령어에서 입력받아야 하지만 다음의 my_arg_2와 같이 기본값을 지정할 수도 있다.
        ```
        # vi Dockerfile
        FROM ubuntu:14.04
        ARG my_arg
        ARG my_arg_2=value2
        RUN touch ${my_arg}/mytouch
        ```
    - 위 내용을 Dockerfile로 저장한 뒤 이미지를 빌드할 때, --build-arg 옵션을 사용해 Dockerfile의 ARG에 값을 입력할 수 있다.
        ```
        # docker build --build-arg my_arg=/home -t myarg:0.0 ./
        ```
    - ARG와 ENV의 값을 사용하는 방법은 ${}로 같으므로 Dockerfile에서 ARG로 설정한 변수를 ENV에서 같은 이름으로 다시 정의하면 --build-arg 옵션에서 설정하는 값은 ENV에 의해 덮어쓰여진다.

- USER
    - USER로 컨테이너 내에서 사용될 사용자 계정의 이름이나 UID를 설정하면 그 아래의 명령어는 해당 사용자 권한으로 실행된다.
    - 일반적으로 RUN으로 사용자의 그룹과 계정을 생성한 뒤 사용한다.
    - 루트 권한이 필요하지 않다면 USER를 사용하는 것을 권장
        ```
        ...
        RUN groupadd -r author && useradd -r -g author covy
        USER covy
        ...
        ```

> 기본적으로 컨테이너 내부에서는 root 사용자를 사용하도록 설정된다. 이는 컨테이너가 호스트의 root 권한을 가질 수 있다는 것을 의미하기 때문에 보안 측면에서 매우 바람직하지 않다. 예를 들어 root가 소유한 호스트의 디렉토리를 컨테이너에 공유했을 때, 컨테이너 내부에서는 공유된 root 소유의 디렉토리를 마음대로 조작할 수 있다. 
<br> 때문에 컨테이너 애플리케이션을 최종적으로 배포할 때는 컨테이너 내부에서 새로운 사용자를 새롭게 생성해 사용하는 것을 권장한다. docker run 명령어 자체에서도 --user 옵션을 지원하지만, 가능하다면 이미지 자체에 root가 아닌 다른 사용자를 설정해 놓는 것이 좋다.


## Onbuild, Stopsignal, Healthcheck, Shell
- ONBUILD
    - 빌드된 이미지를 기반으로 하는 다른 이미지가 Dockerfile로 생성될 때 실행할 명령어를 추가한다.
    - 간단한 예로, 먼저 아래와 같은 내용의 Dockerfile을 작성해보자.
        ```
        # vi Dockerfile
        FROM ubuntu:14.04
        RUN echo "this is onbuild test!"
        ONBUILD RUN echo "onbuild!" >> /onbuild_file
        ```
    - "onbuild!"라는 명령어가 최상위 디렉토리의 onbuild_file에 저장되도록 하였다. 이 명령어는 이 Dockerfile을 빌드할 때 실행되지 않으며, 별도의 정보로 이미지에 저장될 뿐이다. 이미지를 빌드해보자
        ```
        # docker build ./ -t onbuild_test:0.0
        ```
    - 예상했다시피 이 이미지로 컨테이너를 생성해도 디렉토리에는 /onbuild_file이 존재하지 않는다. 
    - 다음 명령어를 통해 onbuild_test:0.0 이미지로 컨테이너를 생성하고, 컨테이너의 커맨드를 ls /로 설정함으로써 컨테이너의 디렉토리를 출력해보자.
        ```
        # docker run -it --rm onbuild_test:0.0 ls /
        bin  boot  dev	etc  home  lib	media  mnt  opt  proc  root  run  sbin	srv  sys  tmp  usr  var
        ```
    - 이번에는 위에서 생성한 이미지를 기반으로 하는 새로운 이미지를 Dockerfile로 생성해보자. 다음과 같이 작성해보자
        ```
        # vi Dockerfile2

        FROM onbuild_test:0.0
        RUN echo "this is child image!"

        # docker build -f ./Dockerfile2 ./ -t onbuild_test:0.1

        [+] Building 0.3s (7/7) FINISHED                                                                                                 docker:desktop-linux
        => [internal] load build definition from Dockerfile2                                                                                            0.0s
        => => transferring dockerfile: 92B                                                                                                              0.0s
        => [internal] load metadata for docker.io/library/onbuild_test:0.0                                                                              0.0s
        => [internal] load .dockerignore                                                                                                                0.0s
        => => transferring context: 2B                                                                                                                  0.0s
        => [1/2] FROM docker.io/library/onbuild_test:0.0                                                                                                0.0s
        => [2/2] RUN echo "onbuild!" >> /onbuild_file                                                                                                   0.1s
        => [3/2] RUN echo "this is child image!"                                                                                                        0.1s
        => exporting to image                                                                                                                           0.0s
        => => exporting layers                                                                                                                          0.0s
        => => writing image sha256:584d0f7c497a67cab51a7dbccb9db4087f4e54f1143cd29652d183ce2a40a3f9                                                     0.0s
        => => naming to docker.io/library/onbuild_test:0.1
        ```
    - 이제야 RUN echo "onbuild!" ...가 하나의 Dockerfile 명령어로 동작한 것을 확인할 수 있다.
        ```
        # docker run -it --rm onbuild_test:0.1 ls /
        bin  boot  dev	etc  home  lib	media  mnt  onbuild_file  opt  proc  root  run	sbin  srv  sys	tmp  usr  var
        ```
    - 이처럼 ONBUILD는 ONBUILD, FROM, MAINTANER를 제외한 RUN, ADD 등, 이미지가 빌드될 떄 수행돼야 하는 각종 Dockerfile의 명령어를 나중에 빌드될 이미지를 위해 미리 저장해 놓을 수 있다.
    - 단 이미지의 속성을 설정하는 다른 Dockerfile 명령어와는 달리 ONBUILD는 부모 이미지의 자식 이미지에만 적용되며, 자식 이미지는 ONBUILD 속성을 상속받지 않는다.
    - ONBUILD를 활용하는 좋은 방법 중에 하나는 이미지가 빌드하거나 활용할 소스코드를 ONBUILD ADD로 추가해 좀 더 깔끔하게 Dockerfile를 사용하는 것이다.
    - 다음은 그 예로, 도커 이미지 중 Maven은 다음과 같은 Dockerfile을 가지고 있으며, 이 이미지를 사용하는 개발자는 프로젝트 폴더에 Dockerfile을 위치시키고, 아래의 Dockerfile로부터 빌드된 이미지를 FROM 항목에 입력함으로써 메이븐을 쉽게 사용할 수 있다.
        ```
        FROM maven:3-jdk-8-alpine
        RUN mkdir -p /usr/src/app
        WORKDIR /usr/src/app
        ONBUILD ADD . /usr/src/app
        ONBUILD RUN mvn install
        ```

- STOPSIGNAL
    - 컨테이너가 정지될 때 사용될 시스템 콜의 종류를 지정
    - 아무것도 설정하지 않으면 기본적으로 SIGTERM으로 설정되지만 Dockerfile에 STOPSIGNAL을 정의해 컨테이너가 종료되는 데 사용될 신호를 선택할 수 있다.
    - 다음은 Dockerfile에서 STOPSIGNAL을 SIGTERM이 아닌 SIGKILL로 지정한 예이다.
        ```
        FROM ubuntu:14.04
        STOPSIGNAL SIGKILL
        ```

> Dockerfile의 STOPSIGNAL은 docker run 명령어에서 --stop-signal 옵션으로 컨테이너에 개별적으로 설정할 수 있다. 이는 docker stop뿐 아니라 docker kill에도 적용된다.

- HEALTHCHECK
    - HEALTHCHECK는 이미지로부터 생성된 컨테이너에서 동작하는 애플리케이션의 상태를 체크하도록 설정
    - 컨테이너 내부에서 동작 중인 애플리케이션의 프로세스가 종료되지는 않았으나 애플리케이션이 동작하고 있지 않은 상태를 방지하기 위해 사용
    - 다음 예시는 1분마다 curl -f ~ 를 실행해 nginx 애플리케이션의 상태를 체크하며, 3초 이상이 소요되면 이를 한 번의 실패로 간주하여, 3번 이상 타임아웃이 발생하면 해당 컨테이너는 unhealthy 상태가 된다. 단, HEALTHCHECK에서 사용되는 명령어가 curl이므로 컨테이너에 curl을 먼저 설치해야 한다.
        ```
        FROM nginx
        RUN apt-get update -y && apt-get install curl -y
        HEALTHCHECK --interval=1m --timeout=3s --retries=3 CMD curl -f http://localhost || exit 1
        ```

- SHELL 
    - 사용하려는 쉘을 따로 지정하고 싶을 때 사용
    - node를 기본 쉘로 사용하도록 설정한 예시
    ```
    FROM node
    RUN echo hello, node!
    SHELL ["/usr/local/bin/node"]
    RUN -v
    ```

## ADD, COPY
- COPY는 로컬 디렉토리에서 읽어 들인 컨텍스트로부터 이미지에 파일을 복사하는 역할을 하며 COPY를 사용하는 형식은 ADD와 같다.
    ```
    COPY test.html /home/
    COPY ["test.html", "/home/"]
    ```
- COPY와 ADD의 차이점은 ADD는 외부 URL 및 tar 파일에서도 파일을 추가할 수 있지만, COPY는 로컬의 파일만 이미지에 추가할 수 있다는 점이다.
- 즉, COPY의 기능이 ADD에 포함된다.
- ADD 사용 예시
    ```
    ADD https://raw.githubusercontent.com/alicek106/mydockerrepo/master/test.html /home
    ```
    ```
    ADD test.tar /home
    ```
- 이처럼 ADD는 추가할 파일을 깃과 같은 외부 URL로 지정할 수 있으며 tar 파일을 추가할 수도 있다.
- 그러나 tar 파일을 그대로 추가하는 것이 아니라 tar파일을 자동으로 해제해서 추가한다.

> 하지만, ADD를 사용하는 것은 권장하지 않는다. 이유는 ADD로 URL이나 tar 파일을 추가할 경우 이미지에 정확히
> 어떤 파일이 추가될지 알 수 없기 때문이다. 그에 비해 COPY는 로컬 컨텍스트로부터 파일을 직접 추가하기 때문에 빌드 시점에서도 어떤 
> 파일이 추가될지 명확하게 알 수 있다.

## ENTRYPOINT, CMD
- CMD는 컨테이너가 시작될 때 실행할 명령어를 설정하며, 이는 docker run 명령어에서 맨 뒤에 입력했던 커맨드와 같은 역할을 한다.
- 그러나 컨테이너의 실행 옵션에는 CMD와 유사한 ENTRYPOINT라는 명령어도 존재하며, ENTRYPOINT와 CMD는 역할 자체는 비슷하지만 서로 다른 역할을 담당한다.

### ENTRYPOINT와 CMD의 차이점
- entrypoint는 커맨드와 동일하게 컨테이너가 시작될 때 수행할 명령을 지정한다는 점에서 같다.
- 그러나 entrypoint는 커맨드를 인자로 받아 사용할 수 있는 스크립트의 역할을 할 수 있다는 점에서 다르다.
- entrypoint가 없을 때 : 컨테이너 내부에서 /bin/bash 실행
    ```
    # docker run -it --name no_entrypoint ubuntu:14.04 /bin/bash

    Unable to find image 'ubuntu:14.04' locally
    14.04: Pulling from library/ubuntu
    d1a5a1e51f25: Already exists
    75f8eea31a63: Already exists
    a72d031efbfb: Already exists
    Digest: sha256:64483f3496c1373bfd55348e88694d1c4d0c9b660dee6bfef5e12f43b9933b30
    Status: Downloaded newer image for ubuntu:14.04
    root@0e495f604aea:/# 
    ```
- entrypoint가 있을 때 : 컨테이너 내부에서 echo /bin/bash 실행
    ```
    # docker run -it --entrypoint="echo" --name yes_entrypoint ubuntu:14.04 /bin/bash

    /bin/bash
    ```

> 이처럼 entrypoint가 설정되었다면 cmd는 단지 entrypoint에 대한 인자 기능만 한다. 만약 커맨드와 entrypoint가 둘 다 설정되지 않으면 컨테이너는 생성되지 않고 에러를 출력하므로 반드시 둘 중 하나를 설정해야 한다.



### entrypoint를 이용한 스크립트 실행
- 일반적으로 스크립트 파일을 entrypoint의 인자로 사용해 컨테이너가 시작될 때마다 해당 스크립트 파일을 실행하도록 설정한다.
    ```
    docker run -it --name entrypoint_sh --entrypoint="/test.sh" ubuntu:14.04 /bin/bash
    ```
- 단 실행할 스크립트 파일은 컨테이너 내부에 존재해야하며, 이는 이미지 내에 스크립트 파일이 존재해야 한다는 것을 의미한다. (COPY나 ADD를 사용하여 이미지에 파일을 추가)
- 이미지를 빌드할 때 이미지가 작동하기 위해서는 다음과 같은 단계를 거친다.
    1. 어떤 설정 및 실행이 필요한지에 대해 스크립트로 정리
    2. ADD 또는 COPY로 스크립트를 이미지에 복사
    3. ENTRYPOINT를 이 스크립트로 설정
    4. 이미지를 빌드해 사용
    5. 스크립트에서 필요한 인자는 docker run 명령에서 cmd로 entrypoint의 스크립트에 전달

> Dockerfile에서는 ENTRYPOINT, CMD를 조합해서 사용할 수 있다. 그러나 Dockerfile에 명시된 CMD와 ENTRYPOINT는 docker run 명령어에서 --entrypoint 옵션과 마지막 옵션으로 재정의해 사용하면 재정의한 명령어로 덮어 쓰인다.

- 컨테이너가 시작될 떄, echo hello world를 실행
    ```
    ENTRYPOINT ["echo"]
    CMD ["hello", "world"]
    ```

### JSON 배열 형태와 일반 형식의 차이점
- CMD 또는 ENTRYPOINT에 설정하려는 명령어를 /bin/sh로 사용할 수 없다면 JSON 배열의 형태로 명령어를 설정해야 한다.
- JSON 배열 형태가 아닌 CMD와 ENTRYPOINT를 사용하면 실제로 이미지를 생성할 떄 cmd와 entrypoint에 /bin/sh -c가 앞에 추가되기 때문이다.

```
CMD echo test
# -> /bin/sh -c echo test

ENTRYPOINT /entrypoint.sh
# -> /bin/sh -c /entrypoint.sh

# 실제 컨테이너에서 실행되는 명령어는 /bin/sh -c entrypoint.sh /bin/sh -c echo test
```

- 예를 들어, 다음 CMD는 실제로 이미지로 생성될 떄 echo test가 아닌 /bin/sh -c echo test로 실행된다.
- ENTRYPOINT도 마찬가지로 /entrypoint.sh가 아닌 /bin/sh -c /entrypoint.sh로 설정된다.
> JSON 배열 형태로 입력하지 않으면 CMD와 ENTRYPOINT에 명시된 명령어의 앞에 /bin/sh -c가 추가 된다고 이해하면 쉽다.

- CMD와 ENTRYPOINT를 JSON 형태로 명령어를 입력하면 입력된 명령어가 그대로 이미지에 사용된다.
```
CMD ["echo", "test"]
# -> echo test

ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
# -> /bin/bash /entrypoint.sh
```