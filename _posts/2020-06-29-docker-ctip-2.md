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

 이번 시간에는 빨간색 테두리로 표시한 부분을 구성합니다.우리는 **도커 컴포즈**를 통해 젠킨스 컨테이너 + 쥬피터 컨테이너를 한 쌍으로 묶어 관리할 것입니다.

또, Makefile을 작성하여 편리하게 여러 쌍의 인스턴스를 생성/삭제/마이그레이션 하는 스크립트를 마련합니다.

이렇게 생성된 독립된 (젠킨스+배포서버) 쌍들은 프로젝트에 당 하나씩 할당하여 사용하면 됩니다.

### 사용할 포트
각 컨테이너들이 쓰게될 포트는 다음과 같습니다.
- 젠킨스: 7000-7009
- 쥬피터: 8000-8009

## 준비물

- 10GB 이상의 리눅스 서버 한 대
- 도커를 설치한다
- 도커 컴포즈를 설치한다

## 기본적인 도커 컴포즈 파일
`(젠킨스 + 쥬피터) `

make를 통한 컨테이너 관리에 관심이 없으십니까?

그렇다면 평범한 도커 컴포즈 파일만으로 젠킨스 + 쥬피터 쌍을 구동해 봅시다.

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
- 원하는 폴더에 위 내용을 `docker-compose.yaml`로 저장합니다.
- 같은 폴더에 `jenkins-data` 폴더를 만듭니다. 
	- 젠킨스 컨테이너가 자료를 저장하는 볼륨입니다
	- 쥬피터 컨테이가 이 폴더의 빌드 결과물을 노트북에다 전시하게 됩니다

- 이제 컨테이너 쌍을 실행시키려면 `docker-compose up` 합니다.

    ```bash
    docker-compose up
    ```

- 컨테이너 쌍을 정지하려면 `docker-compose stop`
- 컨테이너 쌍을 재시작하려면 `docker-compose restart`
- 컨테이너 쌍을 삭제하려면 `docker-compose -f rm`

### xentai/jenkins는 무엇인가요?
- 평범한 jenkins 도커 이미지와 똑같습니다
- 네임스페이스만 제 것으로 바꾼 것입니다.

### xentai/juptyer는 무엇인가요?
- 환경변수 `pass=비밀번호` 를 설정할 수 있습니다
- 쥬피터 접속 비밀번호를 설정할 수 있는 이미지입니다.

## 여러 쌍의 인스턴스 관리하기
도커 컴포즈 파일을 사용했더니

한 쌍의 젠킨스+배포서버 컨테이너 생명주기를 동시에 관리할 수 있게 되었습니다.

하지만 제가 관심있는 것은 고작 한 쌍의 인스턴스가 아닙니다. 

이제는 Makefile 스크립트를 작성하여 간단하게 여러 쌍의 인스턴스를 다뤄봅시다.

### (젠킨스 + 쥬피터) 도커 컴포즈 템플릿 작성

Makefile로 컨테이너를 다루기 위해

기존 도커 컴포즈 파일의 치환가능한 부분을 플레이스 홀더로 만들어 버립시다.

즉, 도커 컴포즈 파일 템플릿을 작성하자는 것이죠. 

플레이스 홀더는 `sed` 명령어를 사용하여 실제 값으로 치환될 것입니다.



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
방금 말한 플레이스 홀더입니다. 

젠킨스+쥬피터 쌍에 id라는 개념을 붙입시다. id 값은 0~9까지가 된다고 하죠.

그렇다면 이 Makefile은 최대 10쌍의 젠킨스+쥬피터를 관리할 수 있게 됩니다.

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

#### `: instance$(id)` 같은 건 무엇인가요?
Makefile은 한 쌍의 인스턴스를 생성하기 위하여 사용자가 직접 `instance{id}` 에 해당하는 폴더를 생성할 것을 요구하고 있습니다.

단순히 `make instance id=3` 라는 명령을 내린다고 해서 3번 인스턴스 쌍이 구동되지 않으니, 같은 폴더에 `instance3` 폴더를 하나 만들어 두세요.

이렇게 메뉴얼한 절차를 남겨둔 목적은 볼륨 데이터를 날리는 실수를 방지하기 위함입니다.

#### 인스턴스 마이그레이션은 무엇인가요?
인스턴스 10 쌍을 이용하고 싶으신가요? 그렇다면 젠킨스를 10번 설정하셔야 겠군요?
1. 젠킨스 회원가입
2. 깃허브 인증, 깃허브 연결 설정, 웹 훅 설정..
3. 플러그인 설정 ..
4. 슬랙 설정 ..

이런 설정 작업을 10번 반복하는 것은 옳지 않습니다. 

0번 인스턴스에서 모든 것을 설정한 뒤 1~9번 인스턴스로 마이그레이션 하는 것이 해법입니다.

#### 인스턴스 마이그레이션의 원리는 무엇인가요?

만약 0번 인스턴스를 3번인스턴스로 마이그레이션 한다고 해보죠. 그 절차는 단순합니다.

1. `intance0/jenkins-data` 폴더를 `instance3/jenkins-data` 에다 복사합니다.
2. 끝!

## 실습

### 준비 단계

<figure>
	<a href="/images/docker-ctip-2-prac-1.png"><img src="/images/docker-ctip-2-prac-1.png" /></a>
	<figcaption>디렉토리 구성</figcaption>
</figure>

1. `jenkins-project`라는 폴더를 만든 뒤 이동하였습니다.
3. `instance0`, `instance1`, `instance2 `, `instance3` 폴더를 만들었습니다.
4. `docker-compose.yaml` 템플릿과 `Makefile`을 작성했습니다

### 인스턴스 생성해보기

3번 인스턴스를 만들어 봅시다. 

```bash
make instance id=3
```

실행 결과입니다.

<figure>
	<a href="/images/docker-ctip-2-prac-2.png"><img src="/images/docker-ctip-2-prac-2.png" /></a>
	<figcaption>인스턴스 생성 결과</figcaption>
</figure>

- 3번 인스턴스로서 두 개의 컨테이너 `jjb-jenkins3`, `jjb-notebook3`가 구동되었군요
- 젠킨스 컨테이너의 포트는 8003, 50003 입니다
- 쥬피터 컨테이너의 포트는 7003 입니다.
- 쥬피터 접속 비밀번호는 `make` 명령시 `pwd` 값을 지정하지 않아서 기본값인 0000이 배정되었습니다. 

어떻게 인스턴스가 생성되는지 감이 잡히셨을 겁니다. 0, 1, 2 번 인스턴스도 같은 명령어로 구동해보세요.



### 인스턴스 멈추기

```bash
make stop id=3
```

<figure>
	<a href="/images/docker-ctip-2-prac-3.png"><img src="/images/docker-ctip-2-prac-3.png" /></a>
	<figcaption>인스턴스 정지 결과</figcaption>
</figure>

- 정지를 원하지 않을 경우 `<C-c>` 하여 취소합니다.
- 정지를 원할 경우 `<ENTER>` 하여 수락합니다



### 인스턴스 제거

`make rm id=3`

<figure>
	<a href="/images/docker-ctip-2-prac-4.png"><img src="/images/docker-ctip-2-prac-4.png" /></a>
	<figcaption>인스턴스 삭제 결과</figcaption>
</figure>

- 삭제를 원하지 않을 경우 `<C-c>` 하여 취소합니다.
- 삭제를 원할 경우 `<ENTER>` 하여 수락한 다음, y를 입력하여 한번 더 수락합니다.
-  두 개의 컨테이너 `jjb-jenkins3`, `jjb-notebook3`가 잘 삭제 됐습니다.



### 인스턴스 마이그레이션

- 인스턴스 0과 3이 살아있다고 가정
- 인스턴스 0에서 젠킨스 설정을 다 해둡니다

```bash
make migrate from=0 to=3
docker restart jjb-jenkins3
```

<figure>
	<a href="/images/docker-ctip-2-prac-5.png"><img src="/images/docker-ctip-2-prac-5.png" /></a>
	<figcaption>인스턴스 마이그레이션 결과</figcaption>
</figure>

- cp 명령어가 수행되어 볼륨이 복제된 것을 확인 가능합니다.

### 인스턴스 접속해보기

- 0번 인스턴스의 모든 내용을 물려 받은 3번 인스턴스에 접속해 볼까요?
  - 젠킨스: `http://호스트ip:7003`
  - 쥬피터: `http://호스트ip:8003`

- 젠킨스

  - 0번 인스턴스에서 설정한 모든 것이 존재.

  <figure>
  	<a href="/images/docker-ctip-2-prac-6.png"><img src="/images/docker-ctip-2-prac-6.png" /></a>
  	<figcaption>3번 젠킨스 인스턴스 접속</figcaption>
  </figure>

- 쥬피터 

  - 따로 비밀번호를 지정하지 않았음. 
  - 0000을 입력하여 접속하면 0번에 있는 모든 내용이 고스란히 존재.
  - 빌드 결과물을 앞으로 쥬피터에서 확인 가능

  <figure>
  	<a href="/images/docker-ctip-2-prac-7.png"><img src="/images/docker-ctip-2-prac-7.png" /></a>
  	<figcaption>3번 쥬피터 인스턴스 접속</figcaption>
  </figure>

## 다음 이 시간에

젠킨스 + 깃허브 + 슬랙 연동을 위한 기본설정을 다룹니다