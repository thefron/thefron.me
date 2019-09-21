---
title: "Heroku Pg Psql Using Docker"
url: /heroku-pg-psql-using-docker
date: 2016-08-31T16:45:45+09:00
tags:
  - heroku
  - docker
  - postgres
  - psql
comments: true
draft: false
---

최근 로컬 개발환경을 리셋하면서 redis, postgres 등 각종 개발 디펜던시를 설치하지 않고, 오직 docker만 이용해서 로컬 환경을 최대한 깔끔하게 유지하기로 마음먹었다(똥고집이다).

그런데 난관이 등장했으니,

프로덕션 배포 중인 heroku 앱의 postgres 디비 콘솔에 접근하기 위해 heroku toolbelt에서 제공하는 `heroku pg:psql` 커맨드를 자주 사용하는데, 이 때 로컬 경로의 `psql`을 이용한다는 것이다.

```bash
$ heroku pg:psql
---> Connecting to DATABASE_URL
sh: psql: command not found
```

굳이 toolbelt를 쓰지 않고 `docker run -it --rm postgres psql` 커맨드에 적당히 credential을 넣어주면 되지만, 매번 credential을 알아오기도 귀찮고 heroku toolbelt를 썩히기도 아까워 해결책이 있을지 고민해봤다.
처음에는 간단히 alias를 걸면 될 줄 알았는데 heroku command에서 쉘 환경을 리셋시켜버려서 command not found가 떴다.

결국 아래와 같이 psql의 인자를 받아 docker 커맨드로 포워드하는 가짜 psql 프로그램을 `/usr/local/bin/psql`에 만들어 넣어서 처리했다.
좀 더 예쁘게 할 수도 있겠지만 bash 지식이 부족하여 일단 해결했다는데서 위안 삼기로 했다.

```bash
#!/bin/bash
docker run -it --rm \
-e PGAPPNAME="$PGAPPNAME" \
-e PGUSER="$PGUSER" \
-e PGPASSWORD="$PGPASSWORD" \
-e PGDATABASE="$PGDATABASE" \
-e PGPORT="$PGPORT" \
-e PGHOST="$PGHOST" \
postgres psql "$@"
```
