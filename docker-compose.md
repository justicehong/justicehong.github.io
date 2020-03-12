---
title: docker-compose guideline
date: 2020-03-12
---
# docker-compose guide

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


## requirements.txt file
    django
    celery[redis]
    redis
    django-redis
    
   
