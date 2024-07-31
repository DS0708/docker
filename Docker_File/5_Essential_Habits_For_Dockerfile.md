# Dockerfile을 사용하는데 좋은 습관

1. \(역슬래시)로 나눠서 가독성을 높일 수 있도록 작성
    ```
    # vi Dockerfile
    ...
    RUN apt-get install package-1 \
    package-2 \
    package-3
    ```

2. .dockerignore파일을 작성해 불필요한 파일을 빌드 컨텍스트에 포함하지 않기

3. &&을 사용하여 RUN 명령을 하나로 묶기 -> Dockerfile은 명령 당 레이어가 하나씩 생성되기 때문에 RUN 명령을 여러 개로 나눠서 작성할 경우, 불필요한 레이어가 많이 생성될 수 있기 때문
    ```
    # 안 좋은 예

    FROM Ubuntu
    RUN mkdir /test
    RUN fallocate -1 100m /test/dumy
    RUN /test/dummy
    ```
    ```
    # 좋은 예

    FROM Ubuntu
    RUN mkdir /test && \
    fallocate -1 100m /test/dumy && \
    rm /test/dummy
    ```

    > 안 좋은 예의 경우 100MB의 /test/dummy라는 파일이 존재하는 이미지 레이어가 추가로 더 생성되지만 좋은 예의 경우 RUN이 하나의 명령어로 묶여 하나의 레이어로 생성되기 때문에 /test/dummy라는 레이어가 생성되지 않아, 메모리 효율적이다.

4. 다른 사람이 빌드한 이미지에 불필요한 이미지 레이어가 들어있다면 해당 이미지로 컨테이너를 생성하고 docker export, import 명령어를 사용해 컨테이너를 이미지로 만듦으로써 이미지의 크기를 줄일 수 있다.
    - export된 파일을 import해서 다시 도커에 저장하면 레이어가 한 개로 줄어든다.
    - 그러나 이전 이미지에 저장돼 있던 각종 이미지 설정은 잃어버리게 되므로 주의해야 한다.
    ```
    docker run -it -d --name temp falloc_100mb:0.0

    docker export temp | docker import - falloc_100mb:0.1

    docker run -it -d --name temp2 falloc_100mb:0.1
    ```

    > 위 예에서는 베이스 이미지인 ubuntu:14.04 이미지의 커맨드 명령어가 손실되어 설정되지 않았고 entrypoint 또한 설정되지 않아 에러를 출력하며 컨테이너가 생성되지 않는다.

















