---
title: docker command 정리
author: mingo
date: 2023-05-05 23:21:09 +0900
categories: [docker]
tags: [docker]
---

*사용할 때마다 추가 예정 낄낄*

## docker build
- `docker build .` : Dockerfile이 존재하는 경로에서 사용하면 파일 스크립트를 읽고 난 뒤 이미지 생성
- `docker build -t [이미지 명:태그 명] .` : 생성되는 이미지명과 태그명을 설정

## docker run
> 컨테이너 실행
 - `docker run --name spring-server -itd -p 9000:9000 -p 9022:22 boot-17-image:latest`
 - `docker run -v [호스트 경로]:[컨테이너 경로]` : 호스트 PC의 경로와 컨테이너 경로를 마운트 시킴
 - `docker run -u [사용자 or UID]:[사용자 그룹 or GID]` : 컨테이너 속에서 실해되는 사용자 및 사용자 그룹을 지정 /UID(사용자 식별자), GID(그룹 식별자)
 - `docker run -p [호스트 포트]:[컨테이너 포트]` : 호스트와 도커 컨테이너 사이에서 특정 포트를 연결하고, 외부에서 컨테이너의 서비스에 접근할 수 있도록 포트포워딩
 - `docker run -i` : interactive, 컨테이너와 상호작용을 할 수 있음
 - `docker run -t` : tty, 터미널을 할당하고 표준입력을 제공함
 - `docker run -d` : detach, 컨테이너를 백그라운드에서 실행함
 - `docker run --name [컨테이너 명 설정]` : 실행시킬 컨테이너에 이름을 부여

## docker stop
> 컨테이너 실행중지
 - `docker stop [컨테이너ID or 컨테이너 명]` : 실행중인 컨테이너 중지

## docker images
> 이미지 목록 확인

## docker container

## docker compose

## docker pull
> 이미지 당겨오기
 - `docker pull []` : 사용할 이미지 당기기 (https://hub.docker.com/search?q=&type=image)

## docker search

## docker ps
> 컨테이너 목록 조회
 - `docker ps` : 현재 실행중인 컨테이너 목록
 - `docker ps -a` : 모든 컨테이너 목록

## docker images
> 이미지 목록 조회
 - `docker image` : 모든 이미지 목록

## docker start
> 컨테이너 실행
 - `docker start [컨테이너ID or 컨테이너 명]` : 컨테이너 실행

## docker attach

## docker stop
> 컨테이너 실행중지
- `docker stop [컨테이너ID or 컨테이너 명]` : 컨테이너 실행중지

## docker rm
> 컨테이너 제거
 - `docker rm [컨테이너ID or 컨테이너 명]` : 해당하는 컨테이너 제거

## docker rmi
> 이미지 제거
 - `docker rmi [이미지ID or 이미지 명]` : 해당하는 이미지 제거

## docker logs
 - `docker logs [컨테이너ID or 컨테이너 명]` : 해당하는 컨테이너 로그

## docker exec
 - `docker exec -it [컨테이너ID or 컨테이너 명] /bin/bash` : 컨테이너 내부로 들어감
 - `CTRL + P`, `CTRL + Q` : 컨테이너 내부에서 나옴

## docker inspect
 - `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' [컨테이너ID or 컨테이너 명]` : 컨테이너 IP 출력
