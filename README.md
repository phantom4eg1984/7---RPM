# Домашнее задание по теме: Управление пакетами. Дистрибьюция софта.

## Сборка собственного пакета Nginx с поддержкой SSL
Устанавливаем необходимые пакеты:
```
[root@nginxssl ~]# yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils gcc
```

Возьмем пакет Nginx и соберем его с поддержкой OpenSSL. Загрузим SRPM пакет, затем установим его.
```[root@nginxssl ~]# wget https://nginx.org/packages/centos/8/SRPMS/nginx-1.20.2-1.el8.ngx.src.rpm
[root@nginxssl ~]# rpm -i nginx-1.20.2-1.el8.ngx.src.rpm
```
Загрузим SSL пакет с поддержкой которого будем собирать наш Nginx rpm пакет и разархивируем его.
```
[root@nginxssl ~]# wget https://www.openssl.org/source/openssl-1.1.1a.tar.gz
[root@nginxssl ~]# tar -xvf openssl-1.1.1a.tar.gz
```

После установки пакета nginx-1.20.2-1.el8.ngx.src.rpm в нашей директории создался файл nginx.spec. Исправим этот файл и добавим в него строчку **with-openssl=/root/openssl-1.1.1a**.
Устанавливем все зависимости для сборки rpm пакета.
```
[root@nginxssl ~]# yum-builddep ./rpmbuild/SPECS/nginx.spec 
```

Собираем RPM пакет:
```
[root@nginxssl ~]# rpmbuild -bb rpmbuild/SPECS/nginx.spec
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.3Wh5vd
+ umask 022
+ cd /root/rpmbuild/BUILD
+ cd nginx-1.23.3
+ /usr/bin/rm -rf /root/rpmbuild/BUILDROOT/nginx-1.23.3-1.el8.ngx.x86_64
+ exit 0
```

Проверяем что созданные нами пакеты были созданы:
```
[root@nginxssl SPECS]# ls -l  /root/rpmbuild/RPMS/x86_64/
total 5152
-rw-r--r--. 1 root root 2061260 Jan 22 08:15 nginx-1.23.3-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 2527912 Jan 22 08:15 nginx-debuginfo-1.23.3-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root  679080 Jan 22 08:15 nginx-debugsource-1.23.3-1.el8.ngx.x86_64.rpm
```

Устанавливаем созданный нами пакет в систему и проверяем что nginx запустился:
```
[root@nginxssl SPECS]# yum localinstall -y /root/rpmbuild/RPMS/x86_64/nginx-1.23.3-1.el8.ngx.x86_64.rpm
[root@nginxssl SPECS]# systemctl start nginx
[root@nginxssl SPECS]# systemctl status nginx

● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2023-01-22 08:38:52 UTC; 53s ago
```
Successful! :+1:

## Создание своего собственного локального репозитария

Для этого создаем для него своей каталог и помещаем в него установочные файлы на которых будем проводить тестирование.
```
[root@nginxssl SPECS]# mkdir /usr/share/nginx/html/repo
[root@nginxssl SPECS]# cp /root/rpmbuild/RPMS/x86_64/nginx-1.23.3-1.el8.ngx.x86_64.rpm /usr/share/nginx/html/repo
[root@nginxssl repo]# wget https://downloads.percona.com/downloads/percona-distribution-mysql-ps/percona-distribution-mysql-ps-8.0.28/binary/redhat/8/x86_64/percona-orchestrator-3.2.6-2.el8.x86_64.rpm -O /usr/share/nginx/html/repo/percona-
orchestrator-3.2.6-2.el8.x86_64.rpm
```

Создаем репозитарий:
```
[root@nginxssl SPECS]# createrepo /usr/share/nginx/html/repo
Directory walk started
Directory walk done - 1 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```

Через nginx который мы установили ранее настроим доступ к файлам. В location добавим директиву autoindex on.
```
location / {
root /usr/share/nginx/html;
index index.html index.htm;
autoindex on;
}
```

Проверим синтаксис файлы который мы только что создали и перезапустим nginx.
```
[root@nginxssl repo]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

[root@nginxssl repo]# nginx -s reload
```

Проверяем что наш репозитарий доступен через web:
```
[root@nginxssl repo]# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          					 22-Jan-2023 12:02                   -
<a href="mc-4.8.29-1.mga9.x86_64.rpm">mc-4.8.29-1.mga9.x86_64.rpm</a>                        15-Jan-2023 00:50             1901680
<a href="nginx-1.23.3-1.el8.ngx.x86_64.rpm">nginx-1.23.3-1.el8.ngx.x86_64.rpm</a>            22-Jan-2023 12:01             2061260
</pre><hr></body>
</html>
```

Добавляем в систему наш новый репозитарий:
```
[root@nginxssl repo]# cat /etc/yum.repos.d/local.repo
[local]
name=local-repo
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
```

Файл который мы разместили в нашем локальном репозитарии нам доступен и мы можем его установить. :+1:
```
[root@nginxssl repo]# yum list | grep local
Failed to set locale, defaulting to C.UTF-8
percona-orchestrator.x86_64                            2:3.2.6-2.el8                                              @local  
```
