---
title: '서버 하나에 CTIP을 구성해보자 (2)'
layout: post
tags:
  - ctip
  - ' docker-compose'
  - ' make'
category: CTIP
---
# 젠킨스와 배포서버 쌍 관리

## 개요

<figure>
	<a href="/images/docker-ctip-2-intro.png"><img src="/images/docker-ctip-2-intro.png" /></a>
	<figcaption>이번 글에서 구성할 젠킨스 + 쥬피터 컨테이너</figcaption>
</figure>

 이번 시간에는 빨간색 테두리로 표시한 부분을 구성합니다.우리는 도커 컴포즈를 통해 젠킨스 컨테이너 + 쥬피터 컨테이너를 한 쌍으로 묶어 관리할 것입니다.

또, Makefile을 작성하여 편리하게 여러 쌍의 인스턴스를 생성/삭제/수정하는 스크립트를 마련해 보려 합니다.

이렇게 생성된 독립된 (젠킨스+배포서버) 쌍은 개별 프로젝트마다 할당해서 사용하면 됩니다.

### 사용할 포트
각 컨테이너들이 네트워크와 통신할 수 있도록
- 젠킨스: 7000-7003
- 쥬피터: 8000-8003 포트를 할당할 것입니다.

## 준비물

- 10GB 이상의 리눅스 서버 한 대를 준비한다
- 도커를 설치한다
- 도커 컴포즈를 설치한다

## 기본적인 (젠킨스 + 쥬피터) 도커 컴포즈 파일
make를 통한 컨테이너 관리에 관심이 없다면, 아래의 도커 컴포즈 파일만으로 젠킨스 + 쥬피터 쌍을 구동할 수 있습니다.
```yaml
version: "2"
services:
 jenkins:
  user: "0" #루트권한
  image: xentai/jenkins
  container_name: jjb-jenkins0
  volumes:
   - "./jenkins-data:/var/jenkins_home"
  ports:
   - "7000:8080" #사용할 포트는 7000입니다
   - "50000:50000"
  networks:
   - jjb-ci-network

 jupyter:
  image: xentai/jupyter
  container_name: jjb-notebook0
  environment:
   - pass=0000 # 쥬피터 접속 비번
  volumes:
   - "./jenkins-data/workspace:/distribution"
  ports:
   - "8000:8888"

networks:
 jjb-ci-network:
```
- 원하는 폴더에 위 내용과 같은 파일을 `docker-compose.yaml`로 저장합니다.
- 같은 폴더에 `jenkins-data` 폴더를 만듭니다. 
	- 젠킨스 컨테이너가 자료를 저장하는 볼륨입니다
	- 쥬피터 컨테이가 이 폴더의 빌드 결과물을 노트북에다 전시하게 됩니다

이제 컨테이너 쌍을 실행시키려면 `docker-compose up` 합니다.
```bash
docker-compose up
```

컨테이너 쌍을 정지하려면 `docker-compose stop`
컨테이너 쌍을 재시작하려면 `docker-compose restart`
컨테이너 쌍을 삭제하려면 `docker-compose -f rm`

### xentai/jenkins는 무엇인가요?
- 평범한 jenkins 도커 이미지와 다르지 않으며 네임스페이스만 제 것으로 바꾼 것입니다.

### xentai/juptyer는 무엇인가요?
- 환경변수 `pass=비밀번호` 를 설정함으로써 쥬피터 접속 시 사용할 비밀번호 설정이 가능한 이미지 입니다.

## 여러 쌍의 인스턴스 관리하기
도커 컴포즈 파일에 한 쌍의 젠킨스+배포서버 컨테이너를 명시하는 것으로 이들의 생명주기를 동시에 관리할 수 있게 되었습니다.

하지만 제가 관심있는 것은 고작 한 쌍의 인스턴스가 아닙니다. 이제는 Makefile 스크립트를 작성하여 여러 쌍의 인스턴스를 관리토록 할 것입니다.

### (젠킨스 + 쥬피터) 도커 컴포즈 템플릿 작성

Makefile로 컨테이너를 관리하기 위해서 기존 도커 컴포즈 파일에서 치환가능한 부분에 플레이스 홀더를 집어넣겠습니다. 즉 도커 컴포즈 파일의 템플릿을 하나 만들어 두는 것이죠. 플레이스 홀더는 `sed` 명령어를 사용하여 실제 값으로 치환됩니다.

원하는 폴더에 `docker-compose.yaml`을 작성합니다.
```yaml
version: "2"
services:
 jenkins:
  user: "0"
  image: xentai/jenkins
  container_name: jjb-jenkins{id}
  volumes:
   - "./jenkins-data:/var/jenkins_home"
  ports:
   - "700{id}:8080"
   - "5000{id}:50000"
  networks:
   - jjb-ci-network

 jupyter:
  image: xentai/jupyter
  container_name: jjb-notebook{id}
  environment:
   - pass={pwd}
  volumes:
   - "./jenkins-data/workspace:/distribution"
  ports:
   - "800{id}:8888"

networks:
 jjb-ci-network:
```
#### {id}가 무엇인가요?
방금 말한 플레이스 홀더입니다. 젠킨스+쥬피터 쌍에 id라는 개념을 붙일 것입니다. 그리고 값은 0~9까지 가질 수 있습니다. 그렇다면 이 Makefile은 최대 10쌍의 젠킨스+쥬피터를 관리할 수 있게 됩니다.

#### {pwd}가 무엇인가요?
쥬피터 접속용 비밀번호가 자리 잡을 곳입니다.

### Makefile 작성
Makefile의 기능으로 4가지가 고려되었습니다.
1. **인스턴스 생성과 실행** `make instance id=[0-9] pwd=.*`
2. **인스턴스 정지** `make stop id=[0-9]`
3. **인스턴스 제거** `make rm id=[0-9]`
4. **인스턴스 마이그레이션** `make migrate from=[0-9] to=[0-9]`

동일한 폴더에 `Makefile`을 작성합니다.
```Makefile
id := 0
pwd := 0000
from :=
to :=

main:

        @echo Creates an instance running one jenkins service and one jupyter-notebook as a distribution service.
        @echo You can specify the id of your instance and the password of jupyter-notebook.
        @echo Usage: make instance [id=[0-9]] [pwd=*]
        @echo Usage: make stop id=[0-9]
        @echo Usage: make rm id=[0-9]
        @echo Usage: make migrate from=[0-9] to=[0-9]

instance: instance$(id)

        cd instance$(id); \
        cp ../docker-compose.yml docker-compose.yml; \
        sed -i 's/{id}/$(id)/g' docker-compose.yml; \
        sed -i 's/{pwd}/$(pwd)/g' docker-compose.yml; \
        docker-compose up -d

        @echo the port of jenkins may be 700$(id), 5000$(id).
        @echo the port of jupyter-notebook may be 800$(id).
        @echo the password for jupyter-notebook is $(pwd). Never forget!

stop: instance$(id)

        @read -p 'Are you trying to shutdown instance$(id)? If not, press CTRL + C, else press ENTER.' x
        @cd instance$(id); \
        docker-compose stop

rm: instance$(id)

        @read -p 'Are you trying to remove instance$(id)? If not, press CTRL + C, else press ENTER. Check things to backup.' x
        @cd instance$(id); \
        docker-compose stop; \
        docker-compose rm

migrate: instance$(from) instance$(to)

        cp -rT ./instance$(from)/jenkins-data ./instance$(to)/jenkins-data
        @cd ./instance$(to); \
        docker-compose up -d

jupyter:

        @deprecated
        @read -p 'instance id(0~9):' id; \
        read -p 'jupyter password:' jup_pass; \
        docker run -d --name jupyter$${id} --env pass=$${jup_pass} -p 800$${id}:8888 -v /home/g079111w/jenkins-project/root-jenkins-data$${id}/jobs:/distribution xentai/jupyter;
```
#### 무슨 원리인가요?
우리는 일전에 도커 컴포즈 파일을 템플릿화 해 놓았지요. Makefile에서는 `sed` 명령어로 사용자가 지정한 id 값과 password를 도커 커포즈 파일에다 치환하여 넣어주고 있습니다.

#### `: instance$(id)` 는 무엇인가요?
Makefile은 한 쌍의 인스턴스를 생성하기 위하여 사용자가 직접 `instance{id}` 에 해당하는 폴더를 생성할 것을 요구하고 있습니다.

단순히 `make instance id=3` 라는 명령을 내린다고 해서 3번 인스턴스 쌍이 구동되지 않으니, 같은 폴더에 `instance3` 폴더를 하나 만들어 두세요.

이렇게 메뉴얼한 절차를 남겨두는 것은, 볼륨 데이터를 날리는 실수를 방지하기 위함입니다.

#### 인스턴스 마이그레이션은 무엇인가요?
인스턴스 10 쌍을 이용하고 싶으신가요? 그렇다면 젠킨스를 10번 설정하셔야 겠군요?
1. 깃허브 연결
2. 플러그인 설치 등.. 

이 작업을 10번 반복하는 것은 옳지 않습니다. 0번 인스턴스에서 모든 것을 설정한 뒤 1~9번 인스턴스로 마이그레이션 하는 것이 해법입니다.