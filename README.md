# **Введение**

В данном домашнем задании нам необходимо получить практический опыт работы с Selinux.

---

# **Подготовка и запуск окружения** 

1. Создаём файл Vagrantfile и вносим в него содержимое из методички:
```
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
    :selinux => {
        :box_name => "centos/7",
        :box_version => "2004.01",
        #:provision => "test.sh",
    },
}

Vagrant.configure("2") do |config|
    MACHINES.each do |boxname, boxconfig|
        config.vm.define boxname do |box|
            box.vm.box = boxconfig[:box_name]
            box.vm.box_version = boxconfig[:box_version]
            box.vm.host_name = "selinux"
            box.vm.network "forwarded_port", guest: 4881, host: 4881
            box.vm.provider :virtualbox do |vb|
                vb.customize ["modifyvm", :id, "--memory", "1024"]
                needsController = false
        end
        box.vm.provision "shell", inline: <<-SHELL
            #install epel-release
            yum install -y epel-release
            #install nginx
            yum install -y nginx
            #change nginx port
            sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
            sed -i 's/listen 80;/listen 4881;/' /etc/nginx/nginx.conf
            #disable SELinux
            #setenforce 0
            #start nginx
            systemctl start nginx
            systemctl status nginx
            #check nginx port
            ss -tlpn | grep 4881
        SHELL
        end
    end
end
```

2. После этого запускаем наш Vagrantfile.
```
dima@Test-Ubuntu-1:~/otus/my-hw11$ vagrant up
```

3. При разворачивании нашей vm видим следующие ошибки:
![alt text](/screenshots/hw11-1.PNG?raw=true "Screenshot1")

4. Далее заходим в vm и пытаемся исправить ошибку и запустить nginx.
```
dima@Test-Ubuntu-1:~/otus/my-hw11$ vagrant ssh
[vagrant@selinux ~]$ sudo -i
```
---

# **Запуск nginx на нестандартном порту 3-мя разными способами** 





