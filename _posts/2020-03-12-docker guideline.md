---
title: "docker 가이드라인 dockerfile docker-compose를 이용한 django mysql redis celery 구축하기"
categories: [Blog]
last_modified_at: 2020-03-12T17:20:00:00+09:00
toc: true
---
# docker-compose guide

도커 컴포즈를 활용하는 이유
1. 여러 개발환경에서 테스트 할 시 프로그램 및 패키지 간의 설치 에러 문제등 발생할 수 있다
2. 여러 환경 변수들 간의 꼬임을 방지
3. 서버를 다른 곳에 배포를 해주기 위한 수작업할 필요성이 낮아지며 개발환경 및 서버환경이 같지 않아지는 문제를 줄여줌
4. 개발 환경 구성에 대한 지속적 업데이트는 매우 힘들기 떄문에 컴포즈를 통해 관리


도커-컴포즈를 시작하기 위해서는 우선 대표적인 파일 2가지가 필요하다.
docker-compose.yml , Dockerfile 


docker-compose 의 경우

$ docker-compose up
$ docker-compose down
$ docker-compose down -v [ 볼륨 드라이버까지 삭제 ]
$ docker-compose build [ 빌드 ]

위에 명령어들을 대표적으로 사용한다.

docker-compose.yml 파일의 내용을 보면

container_name : 컨테이너 이름 지정
image : 컨테이너로 생성할 이미지
environment : MysqlDB 생성시 기본 설정 관련
ports : 외부에서 접속시 포트 : 내부에 접속될 포트
volumes : 바인드 마운팅 시 도커 안에서의 경로 : 컨테이너 안의 경로
command : 컨테이너 실행 후 실행될 
tty : 컴포즈의 경우 커맨드를 실행한 뒤 종류 되는 데. 종류되는 것을 방지하기 위한 옵션
build : image 항목 대신 도커파일을 가져와서 빌드하겠다는 내용
    - context : Docker 빌드 명령어를 실행할 위치
    - dockerfile : 도커 파일 위치


## docker-compose.yml file
    version: "4.7"
    volumes:  # Container 들에서 사용되는 Volume을 정의한다.
    sample-project-db-volume: {}
    sample-project-cache-volume: {}
    sample-project-media-volume: {}

    services:
        sample-project-db:
            container_name: sample.mysql
            image: mysql:5.7
            environment:
                MYSQL_ROOT_PASSWORD: ""
                MYSQL_DATABASE: ""
                MYSQL_USER: ""
                MYSQL_PASSWORD: ""
            ports:
                - "3306:3306"
            volumes:
                - sample-project-db-volume:/var/lib/mysql
            command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci

        sample-project-cache:
            container_name: sample.redis
            image: redis:2.8
            command: redis-server --requirepass samplepassword
            ports:
                - "0.0.0.0:7772:6379"
            volumes:
                - sample-project-cache-volume:/data
            healthcheck:
                test: "redis-cli -h 127.0.0.1 ping"
                interval: 3s
                timeout: 1s
                retries: 5

        sample-project:
            container_name: sample.django
            image: sample.django:v1
            #build:
            #    context: .
            #    dockerfile: ./Dockerfile
            ports:
                - "0.0.0.0:7770:8000"
            depends_on:
                - sample-project-db
                - sample-project-cache
            links:
                - sample-project-db:sample-project-db
                - sample-project-cache:sample-project-cache
            tty: true
            command: bash -c "source /venv/bin/activate && pip3 install -r requirements.txt && uwsgi --ini uwsgi.ini && python3 manage.py runserver 0.0.0.0:8000 && python3 manage.py makemigrations {db}"
            volumes:
                - .:/sample-project/sample-project
                - sample-project-media-volume:/sample-project/sample-project-media:Z

        sample-project-task:
            container_name: sample.celery
            #image: uploadvoice_sample-project-task:v1
            build:
                context: .
                dockerfile: ./Dockerfile
            depends_on:
                - sample-project-db
                - sample-project-cache

            links:
                - sample-project-db:sample-project-db
                - sample-project-cache:sample-project-cache
            #command: bash -c "pip3 install -r requirements.txt"
            command: bash -c "pip3 install -r requirements.txt && python3 -m celery worker --beat -l DEBUG --logfile=logfile/celery1%I.log --pidfile=logfile/celery1.pid --hostname=celery1@컨테이너이름"
            volumes:
                - .:/sample-project/sample-project
                - sample-project-media-volume:/sample-project/sample-project-media:Z


Dockerfile 의 내용이다

FROM : 어떤 이미지를 사용해서 빌드를 할 것인지 선언
ENV : 환경 변수 지정
RUN : 해당 명령어들을 순차적으로 실행
ADD : 이미지 빌드 시 바인드 마운트를 한다 ( compose 에 volumes는 컴포즈를 실행시킬떄 일시적으로 이어지지만 ADD는 빌드시에 가져오는 것이라 영구적이라 보면 됨 )
WORKDIR : 저기 디렉토리에서부터 명령어를 실행하겠다.


## Dockerfile file
    FROM ubuntu:16.04

    ENV PYTHONUNBUFFERED 1
    ENV PPYTHONENCODING utf-8

    RUN apt-get update -y
    RUN apt-get install -y software-properties-common build-essential python3 python3-dev python3-pip libmysqlclient-dev language-pack-ko

    RUN DEBIAN_FRONTEND=noninteractive apt-get install -y tzdata
    ENV LANG ko_KR.UTF-8
    ENV LANGUAGE ko_KR.UTF-8
    ENV LC_ALL ko_KR.UTF-8
    RUN locale-gen ko_KR.UTF-8
    RUN ln -fs /usr/share/zoneinfo/Asia/Seoul /etc/localtime
    RUN dpkg-reconfigure --frontend noninteractive tzdata

    RUN python3 -m pip install pip --upgrade
    RUN python3 -m pip install wheel
    RUN pip install mysqlclient

    RUN pip install django_select2
    RUN pip install django_celery_beat
    RUN pip install django_celery_results
    RUN pip install numpy

    ADD . /sample-project/sample-project
    WORKDIR /sample-project/sample-project


requirements.txt 파일은 docker-compose up을 시켰을 떄. command를 통해 다운로드 받는다.
( 순차적으로 도커 파일에 테스트하면서 다 올릴 예정 )

## requirements.txt file
    django
    celery[redis]
    redis
    django-redis
    
   

# 에러 났던 항목


1번째 (mysql root *@ 이런식으로 나옴)

docker-compose down -v
docker-compose up
실행 시 mysql "root" 라는 말이 뜨면서 안됬을 때.
mysql을 새로 빌드하는 데 그 빌드하는 이미지 안에 mysql 계정이 root라는 이름으로 이미 있어서 그렇다. 빌드시에 이미 그전에 덤프한 mysql 이미지라면 user랑 password를 빼버리거나 아무도 쓰지않는 계정으로 설정 후 실행하면 정상적으로 실행가능



2번째 (puzzleai.mysql 이 안켜져있다고 나옴)

다른서버들이 전부 켜졌지만 django 서버가 안켜졌을 경우 django 서버가 켜질때 mysql서버가 죽어있어서 manage.py runserver 8000을 진행할때 mysql DB가 연동이 안되면서 죽은거기 때문에 mysql db를 먼저 빌드해서 실행해주고 시작하면 좋다.
docker-compose up {db-name} 이런식으로 1개 먼저 구동도 가능


