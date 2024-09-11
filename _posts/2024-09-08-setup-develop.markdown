---
title: Django 개발 환경 구축 (uwsgi + nginx + postgresql)
date: YYYY-MM-DD HH:MM:SS +09:00
categories: [개발 관련, Django]
tags: Django
---

# Django 개발 환경 구축 (uwsgi + nginx + postgresql)

## 개요

최근 Django에 대한 흥미를 가지게 되어 개발 연습삼아 환경을 구축해 보았습니다.

개발 및 시험 운영에 필요하다고 생각되는 nginx + uwsgi + Django + Postgresql 조합으로 환경을 구성했으며, 과정마다 겪은 문제와 해결 방안도 함께 정리합니다.

❖ 본 문서는 python 가상 환경이 아닌 시스템 전역에 단일 환경으로 구성하는 과정을 작성합니다.

​

개발 환경은 아래와 같습니다.

OS : Redhat 8

따라서 이 문서의 적용 범위는 같은 Redhat 계열인 Alma와 Rocky linux 8에 국한됩니다.

​

글은 아래 순서대로 정리합니다.

패키지 설치 및 설정

* Django

* uwsgi

* nginx

* postgresql (다음 포스트)

* Django 앱 작성 (다음 포스트)

​

각 단계는 설치, 설정, 트러블 슈팅 순서로 구성합니다.

​

## Django 설치

Python Packaging Authority 도구를 이용해 Django를 설치합니다.

여기서는 Django 코드를 작성하고 개발된 앱을 시스템에 적용하는 일반 사용자 계정(foo)를 추가하는 내용이 포함됩니다.

또한 웹서버 계정(nginx)와 같은 파일에 엑세스 하기 위한 그룹(www-data)를 생성합니다.
<br><br>
[계정 추가]

root 계정으로 일반 사용자 계정(foo)와 일반 그룹 (www-data)를 추가합니다.

```{shell}
adduser foo
groupadd www-data
useradd -g www-data -s /usr/sbin/nologin www-data
```
<br>

**Django 설치**

```{shell}
pip install Django
mkdir -p /var/www/example
chown -R foo:www-data /var/www/example
cd /var/www/example
mkdir logs repo run ssl
```

**Django 프로젝트 생성**

```{shell}
cd /var/www/example/repo
django-admin.py startproject conf .
./manage.py makemigrations
./manage.py migrate
./manage.py createsuperuser
./manage.py collectstatic
```

**Django 설정**

작성 중..

**트러블 슈팅**

```{shell}
sudo pip list | grep Django
Django                       3.2.10    #Django 설치 확인
sudo /var/www/example/repo/manager.py runserver 0.0.0.0:8000 #장고 테스트 서버가 동작하는지 확인
```

<br>

## uwsgi 설치
---
Python Packaging Authority 도구를 이용해 uwsgi를 설치합니다.
```{shell}
pip install uwsgi
```

<br>

**uwsgi 설정**
```{shell}
#/etc/uwsgi/sites/example.ini
[uwsgi]
uid = foo
gid = www-data
base = /var/www/example
chdir = %(base)/repo
module = conf.wsgi:application
env = DJANGO_SETTINGS_MODULE=conf.settings

master = true
processes = 5

socket = %(base)/run/uwsgi.sock
logto = %(base)/logs/uwsgi.log
chown-socket = %(uid):www-data
chmod-socket = 666
vacuum = true
```

<br>

## nginx 설치

---

패키지 관리 도구(dnf)를 이용해 nginx를 설치합니다.

```{shell}
sudo dnf install nginx
sudo systemctl start nginx
```

**nginx 설정**

nginx는 기본적으로 /etc/nginx/nginx.conf를 참조합니다.

사용자 설정은 /etc/nginx/conf.d 경로에 확장자 .conf 파일을 생성합니다.

```{shell}
#/etc/nginx/conf.d/example.conf 내용
server {
    listen 80;
    server_name {IP주소};
    root /var/www/example;
    location /static/ {
        alias /var/www/example/repo/static/;
    }
    location / {
        include uwsgi_params;
        uwsgi_pass unix:/var/www/example/run/uwsgi.sock;
    }
}
```

<br>

**트러블 슈팅**

웹 브라우저로 접속을 시도했을때 접속할 수 없는 경우 (CONNECTION_REFUSE)

먼저 서비스 포트가 열려있는지 확인한다.

HTTP 서비스 포트로 LISTEN 중인지 확인한다.

아래 두 명령어를 통해 port 80이 서비스 되고 있는지와 `nginx` 프로세스가 80 포트를 열고 있는지 확인할 수 있다.

```{shell}
sudo netstat -antl
sudo lsof -c nginx -a -i
```

그리고 selinux 정책에서 막혀있는지 확인한다.

```{shell}
setsebool httpd_can_network_connect on -P  # 이 옵션이 off일 경우 HTTP 모듈이 네트워크 또는 원격 포트에 대한 연결을 시작하지 못하게 한다.
getsebool -a  | grep httpd #확인
```

방화벽 데몬에서 포트를 차단하는지도 확인한다.

```{shell}
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
firewall-cmd --reload
```

```{shell}
tail -f /var/log/nginx/error.log
2024/09/07 21:47:54 [crit] 31478#0: *1 connect() to unix:/var/www/com.example/run/uwsgi.sock failed (13: Permission denied) while connecting to upstream, client: 172.16.123.1, server: 172.16.123.128, request: "GET / HTTP/1.1", upstream: "uwsgi://unix:/var/www/com.example/run/uwsgi.sock:", host: "172.16.123.128"
2024/09/07 21:47:54 [crit] 31478#0: *1 connect() to unix:/var/www/com.example/run/uwsgi.sock failed (13: Permission denied) while connecting to upstream, client: 172.16.123.1, server: 172.16.123.128, request: "GET /favicon.ico HTTP/1.1", upstream: "uwsgi://unix:/var/www/com.example/run/uwsgi.sock:", host: "172.16.123.128", referrer: "http://172.16.123.128/"
```

permissive 모드에서 보안 문제는 /var/log/audit/audit.log에 기록된다[1]
​<br>
<br>
**트러블 슈팅**

`nginx`가 unix 소켓에 접근 가능하도록 정책 허용

{% highlight ruby %}
grep nginx /var/log/audit/audit.log | audit2allow
grep nginx /var/log/audit/audit.log | audit2allow -m nginx
grep nginx /var/log/audit/audit.log | audit2allow -M nginx
{% endhighlight %}

```{console}
# show the new rules to be generated
grep nginx /var/log/audit/audit.log | audit2allow

# show the full rules to be applied
grep nginx /var/log/audit/audit.log | audit2allow -m nginx

# generate the rules to be applied
grep nginx /var/log/audit/audit.log | audit2allow -M nginx

# apply the rules
semodule -i nginx.pp
```

<br>

## 참조 자료
---

[1] [selinux-policy-for-nginx-add-gitlab-unix-socket-in-fedora]("https://axilleas.me/en/blog/2013/selinux-policy-for-nginx-and-gitlab-unix-socket-in-fedora-19/")
