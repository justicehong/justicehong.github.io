---
title: docker-compose guideline
date: 2020-03-12
---
# docker-compose guide

## docker-compose file
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
        command: bash -c "source /puzzleai_django/bin/activate && pip3 install -r requirements.txt && uwsgi --ini puzzleai_uwsgi.ini && python3 /puzzleai/manage.py runserver 0.0.0.0:8000 && python3 /puzzleai/manage.py makemigrations puzzleai"
        #command: bash -c "source /puzzleai_django/bin/activate && pip3 install -r requirements.txt && while true; do python3 /puzzleai/manage.py runserver 0.0.0.0:8000 && uwsgi --ini puzzleai_uwsgi.ini; sleep 2; done"
        volumes:
            - ./uploadvoice:/puzzleai
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
        command: bash -c "pip3 install -r requirements.txt && python3 -m celery worker --beat -l DEBUG --logfile=logfile/celery1%I.log --pidfile=logfile/celery1.pid --hostname=celery1@026d1d0ece14"
        volumes:
            - .:/sample-project/sample-project
            - sample-project-media-volume:/sample-project/sample-project-media:Z
