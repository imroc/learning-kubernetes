---
title: 进程统计
type: book
date: "2021-04-29"
weight: 50
---

## 线程数排名统计

可用于排查哪个进程有大量线程(线程泄露):

``` bash
$ printf "    NUM  PID\t\tCOMMAND\n" && ps -eLf | awk '{$1=null;$3=null;$4=null;$5=null;$6=null;$7=null;$8=null;$9=null;print}' | sort |uniq -c |sort -rn | head -2
    NUM  PID        COMMAND
    594  14418      java -server -Dspring.profiles.active=production -Xms2048M -Xmx2048M -Xss256k -Dinfo.app.version=33 -XX:+UseG1GC -XX:+UseStringDeduplication -XX:+PrintGCDateStamps -verbosegc -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=10011 -Xloggc:/home/log/gather-server/gather-server-gac.log -Ddefault.client.encoding=UTF-8 -Dfile.encoding=UTF-8 -Dlogging.path=/home/log/gather-server -Dserver.tomcat.accesslog.directory=/home/log/gather-server -jar /home/app/gather-server.jar --server.port=8080 --management.port=10086
    449  7088       java -server -Dspring.profiles.active=production -Dspring.cloud.config.token=nLfe-bQ0CcGnNZ_Q4Pt9KTizgRghZrGUVVqaDZYHU3R-Y_-U6k7jkm8RrBn7LPJD -Xms4256M -Xmx4256M -Xss256k -XX:+PrintFlagsFinal -XX:+UseG1GC -XX:+UseStringDeduplication -XX:MaxGCPauseMillis=200 -XX:MetaspaceSize=128M -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=10011 -verbosegc -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=15 -XX:GCLogFileSize=50M -XX:AutoBoxCacheMax=520 -Xloggc:/home/log/oauth-server/oauth-server-gac.log -Dinfo.app.version=12 -Ddefault.client.encoding=UTF-8 -Dfile.encoding=UTF-8 -Dlogging.config=classpath:log4j2-spring-prod.xml -Dlogging.path=/home/log/oauth-server -Dserver.tomcat.accesslog.directory=/home/log/oauth-server -jar /home/app/oauth-server.jar --server.port=8080 --management.port=10086 --zuul.server.netty.threads.worker=14 --zuul.server.netty.socket.epoll=true
```

* 第一列表示线程数
* 第二列表示进程 PID
* 第三列是进程启动命令
* 自定义末尾的 head 参数来调整要显示的 top 数量
