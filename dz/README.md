#### Стенд для занития PAM PolKit

Цель: Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников
* дать конкретному пользователю права работать с докером и возможность рестартить докер сервис

Поднимим `vagrant up`
1. Создадим 3-х пользователей, а четвертого пользователя `vagrant` добавим в группу `admin`
- user1 - группа `user1` пароль `user`
- user2 - группа `user2` пароль `user`
- admin - группа `admin` пароль `admin`

```
[root@pam ~]# useradd user1 && useradd user2 && useradd admin
[root@pam ~]# echo "user"|sudo passwd --stdin user1 && echo "user"|sudo passwd --stdin user2 && echo "admin"|sudo passwd --stdin admin
```
Пользователь `vagrant` добавим в группу `admin`
```
[root@pam ~]# usermod vagrant -G admin
```

2. Разрешим вход через ssh по паролю:
```
[root@pam ~]# sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config && systemctl restart sshd.service
```
3. Настроить доступ пользователям, запретим вход по ssh всем пользователя, кроме группы `admin` 

Изменим файл `/etc/pam.d/sshd` и добавим модуль pam_exec, будет осуществлять необходимые проверки.
```
account    required	pam_exec.so /usr/local/bin/gist_login.sh
```
Скрипт gist_login.sh
```
#!/bin/bash

ugroup=$(id $PAM_USER | grep -ow admin)
uday=$(date +%u)

if [[ -n $ugroup || $uday -lt 6 ]]; then
	exit 0
else
	exit 1
fi
```
```
[root@pam ~]# chmod +x /usr/local/bin/gist_login.sh
```
4. Настроить доступ пользователям, запретим вход локально всем пользователя, кроме группы `admin` 

Изменим файл `/etc/pam.d/login` и добавим модуль pam_exec, будет осуществлять необходимые проверки.
```
account    required	pam_exec.so /usr/local/bin/gist_login.sh
```
5. Установим следующие пакеты
```
[root@pam ~]# yum install yum-utils -y
[root@pam ~]# yum install wget -y
```
Подключим репозиторий `docker-ce.repo`
```
[root@pam ~]# wget https://download.docker.com/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```
Установим `docker`
```
[root@pam ~]# yum install docker-ce y
```

6. Пользователю `user1` дадим права для работы с `docker` и рестарт `systemctl restart docker`

Добавим пользователя `user1` в группу `docker`
```
[root@pam ~]# usermod -G docker user1
```
Настроим PolKit для логирования и перезагрузки сервиса `docker` для пользователя `user1`. Создадим два файла `00-log-access.rules`, `01-docker-service.rules`.
```
[root@pam ~]# vi /etc/polkit-1/rules.d/00-log-access.rules
```
```
polkit.addRule(function(action, subject) {
    polkit.log("action=" + action);
    polkit.log("subject=" + subject);
});
```
```
[root@pam ~]# vi /etc/polkit-1/rules.d/01-docker-service.rules
```
```
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units" &&
        action.lookup("unit") == "docker.service" &&
        action.lookup("verb") == "restart" &&
        subject.user == "user1") {
        return polkit.Result.YES;
    }
});
```
Для функционирования правил `action.lookup("unit")` и `action.lookup("verb")` в `systemclt`, нужно обновить.
Подключим репозиторий для обновления `systemclt`
```
[root@pam ~]# wget https://copr.fedorainfracloud.org/coprs/jsynacek/systemd-backports-for-centos-7/repo/epel-7/jsynacek-systemd-backports-for-centos-7-epel-7.repo -O /etc/yum.repos.d/systemd-centos-7.repo
```
Обновим `systemctl`
```
[root@pam ~]# yum update systemd -y
```

### Выполним `vagrant up` подключимся по ssh на 192.168.11.150
1. Пользователь `admin` `passwd=admin` и `vagrant` разрешено подключение на сервер в любое время
2. Пользователь `user1`, `user2` `passwd=user` запрещено подключение на сервер в субботу и воскресение.
3. Пользователь `user1` разрешено выполнять перезагрузку сервиса `docker`.
