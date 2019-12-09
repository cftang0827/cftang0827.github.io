---
layout: post
title:  "在CircleCI上面使用多個Docker conatainer的方案"
date:   2019-12-09 14:00:00 +09
categories: Docker CircleCI
---

這是一篇筆記文，我的文筆很爛，此外我也剛開始接觸Docker，這不是教學而是有點像是踩雷心得，請加減看看，有問題歡迎指教XD

### 前情提要
我們公司使用CircleCI 作為CI的工具，基本上，CircleCI的使用相當簡單

綁定Github account 到CircleCI 
→ Enable Repository in CircleCI
→ 撰寫CircleCI config YAML 檔 
→ 推到Github 上面

理論上他就會針對config裡面的步驟進行一步一步的測試和部署，關於config的撰寫，circleci提供一些[範例](https://circleci.com/docs/2.0/sample-config/)，這邊就不贅述了。

CircleCI在他們的伺服器上使用Docker container來進行測試，一旦把想要測試和部署的專案推上Github以後，他會觸發一個webhook讓CircleCI開啟一個container進行乾淨的測試和部署。

這次遇到的問題是，如果我今天想要測試一個API app，整合測試的部分一定有可能會使用到其他的服務，例如MySQL、Redis等等，這個就會有一點小小的麻煩了。


一開始我找到了CircleCI的文件，關於不同服務的整合測試，CircleCI有提供官方的[解決方案](https://circleci.com/docs/2.0/postgres-config/)

但是如果我就是想要用Docker Compose進行測試怎麼辦？(很任性XD)
因為這樣就能直接把在Local端的測試環境直接搬上CI環境，也不用搞一堆其他有的沒的...?

總之，本篇文章主要會以在CircleCI docker環境內再創建另外一個Docker-compose的方法為主，也就是一個container裡面再有container的概念 

### Talk is Cheap, show me the code ?!

這邊就不囉唆，直接上circleci 的yaml config
```yaml
version: 2
jobs:
  build:
    working_directory: /test_app
    parallelism: 1
    docker:
      - image: docker/compose:1.25.0-rc2-debian
    steps:
      - run:
          name: Configure environment
          command: |
            apt-get update
            apt-get install git -y
      - setup_remote_docker
      - checkout

      - run:
          name: Docker test 
          command: |
            # sleep 15
            # docker exec api make test
            docker-compose run api make test 
      - run:
          name: Clean up docker-compose table
          command: |
            docker-compose down
```

以及我們要執行測試的docker compose file
其實就是一個很單純的 api 測試，執行測試的時候需要用到Celery worker，而Celery需要連到Redis & MySQL

- 附註: Celery worker是一個async python worker framework, 用法是和api同一個container但是使用celery command，`celery -A api worker -l info`，這樣就可以把一個async worker叫起來

docker compose yaml file如下

```yaml
version: "3"

services:
    db:
        image: mysql:5.7
        restart: always
        container_name: db
        ports:
            - "3306:3306"
        environment:
            - MYSQL_DATABASE=test
            - MYSQL_ROOT_PASSWORD=xxxxx
            - MYSQL_PORT=3306
    redis:
        image: redis
        container_name: redis
        restart: always
        ports:
            - "6379:6379"
    celery:
        build: .
        image: celery
        command:
              celery -A api worker -l info
        container_name: celery
        links:
            - db
            - redis
        environment:
            - DATABASE_MASTER_URL=######Link to db##########
            - CELERY_REDIS_URL=redis://redis:6379
volumes:
    db_data: {}

```

好像很簡單，對吧？

但是人生沒那麼容易的，如果你就這樣單純的執行下去
你很有可能會遇到一個問題...

```
django.db.utils.InterfaceError: (2003, "2003: Can't connect to MySQL server on 'db:3306' (111 Connection refused)", None)
M
```

什麼？！！

竟然連不到DB? 在本機端測試明明都沒問題啊....


### 問題分析
根據CircleCI提供的錯誤log，分析可能是因為MySQL server壓根沒有被執行起來
基本上docker-compose 如果加上`depend_on` 基本上可以指定container執行的順序
例如:

```yaml
version: "3"

services:
    db:
        image: mysql:5.7
        restart: always
        container_name: db
        ports:
            - "3306:3306"
        environment:
            - MYSQL_DATABASE=test
            - MYSQL_ROOT_PASSWORD=xxxxx
            - MYSQL_PORT=3306
    redis:
        image: redis
        container_name: redis
        restart: always
        ports:
            - "6379:6379"
    celery:
        build: .
        image: celery
        command:
              celery -A api worker -l info && echo "Check db and redis OK"
        container_name: celery
        depends_on:
            - db
            - redis
        links:
            - db
            - redis
        environment:
            - DATABASE_MASTER_URL=######Link to db##########
            - CELERY_REDIS_URL=redis://redis:6379
volumes:
    db_data: {}
```

結果？

很不幸的還是一樣的錯誤...



在上述的yaml 可以看到如果在api加上了depens_on，確實可以指定api會在redis 以及 MySQL後**被執行**

不過請注意喔，是被執行，並不代表是可以被連線或是服務已經ready，可以參考這篇[文章](https://blog.tldnr.org/2018/06/06/control-docker-compose-startup-order/)


### 解決方案
根據文章的的建議，我們應該可以設定一個小小的檢查器，如果該服務還沒有被確定啟動，就不會執行，讓他一直等，等到dependencies 都確定可以被連線以後，才會執行下一步

這邊我們用的小工具叫做 [wait-for-it](https://github.com/vishnubob/wait-for-it)

基本上就是一個pure bash script，可以用來block住你的程式碼，等到需要等待的資料庫或是其他服務真的**ready** 後才會執行

使用範例如下
```bash
./wait-for-it.sh db:3306 -- echo "db is up"
```

也由於他是純bash寫成的，你不用安裝任何的dependencies，直接把那隻bash script丟進repo或是在circleci build的時候直接下載也可

所以改寫後的docker compose會像這樣
```yaml
version: "3"

services:
    db:
        image: mysql:5.7
        restart: always
        container_name: db
        ports:
            - "3306:3306"
        environment:
            - MYSQL_DATABASE=test
            - MYSQL_ROOT_PASSWORD=xxxxx
            - MYSQL_PORT=3306
    redis:
        image: redis
        container_name: vs_api_redis
        restart: always
        ports:
            - "6379:6379"
    celery:
        build: .
        image: celery
        command:
            bash -c "./wait-for-it.sh db:3306 -- ./wait-for-it.sh redis:6379 -- celery -A vs_api worker -l info && echo "Check db and redis OK""
        container_name: vs_api_celery
        depends_on:
            - db
            - redis
        links:
            - db
            - redis
        environment:
            - DATABASE_MASTER_URL=######Link to db##########
            - CELERY_REDIS_URL=redis://redis:6379
volumes:
    db_data: {}
```

可以注意到在celery command的部分，我們加上了wait-for-it的指令，你就會在circleci的log上面發現，他真的有在等待

```
^@^@wait-for-it.sh: waiting 15 seconds for db:3307
```

然後你的circleci就成功了!!!!!!!!!! :) 

### 結語
當然，我還是認為在circleci的docker container裡面再起一個docker-compose不是一個很好的idea，因為目前我們還是遇到了circleci build time非常久的問題，但是如果你發現circleci提供的[多個conatinaer作法](https://circleci.com/docs/2.0/postgres-config/)不合你的胃口，或許你也可以嘗試看看本篇的做法唷。有問題以及指正歡迎在下方留言，謝謝。



