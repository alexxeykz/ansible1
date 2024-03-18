# ansible-training
1. Установил машину с помощью Vagrant
Проверил доступ по ssh 
Проверил конфигурацию

root@testvm:/home/Ansible# vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/Ansible/.vagrant/machines/nginx/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
  PubkeyAcceptedKeyTypes +ssh-rsa
  HostKeyAlgorithms +ssh-rsa

Проверяем хост 
ansible nginx -i staging/hosts -m ping

root@testvm:/home/Ansible# ansible nginx -i staging/hosts -m ping
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ED25519 key fingerprint is SHA256:CaQePAGRhKdAaVfqPEq7bJV8jGOUqRlo2RtFxyJnNF4.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? y
Please type 'yes', 'no' or the fingerprint: yes
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

Создаем ansible.cfg чтобы каждый раз не вписывать пользователя
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False

Смотрим файл

[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key

Проверяем

root@testvm:/home/Ansible# ansible nginx -i staging/hosts -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

Проверяем ядро

root@testvm:/home/Ansible# ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
5.15.0-92-generic

Проверяем сервис статуса firewalld

root@testvm:/home/Ansible# ansible nginx -m systemd -a name=firewalld
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "name": "firewalld",
    "status": {
        "ActiveEnterTimestamp": "n/a",
        "ActiveEnterTimestampMonotonic": "0",
        "ActiveExitTimestamp": "n/a",
        "ActiveExitTimestampMonotonic": "0",
        "ActiveState": "inactive",
        "AllowIsolate": "no",
        "AssertResult": "no",
        "AssertTimestamp": "n/a",
        "AssertTimestampMonotonic": "0",
        "BlockIOAccounting": "no",
        "BlockIOWeight": "[not set]",
        "CPUAccounting": "yes",
        "CPUAffinityFromNUMA": "no",
        "CPUQuotaPerSecUSec": "infinity",
        "CPUQuotaPeriodUSec": "infinity",
        "CPUSchedulingPolicy": "0",
        "CPUSchedulingPriority": "0",
        "CPUSchedulingResetOnFork": "no",
        "CPUShares": "[not set]",
        "CPUUsageNSec": "[not set]",
        "CPUWeight": "[not set]",
        "CacheDirectoryMode": "0755",
        "CanFreeze": "yes",
        "CanIsolate": "no",
        "CanReload": "no",
        "CanStart": "no",
        "CanStop": "yes",

Далее создаем templates/nginx.conf.j2  Шаблон

# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}

Создаем playbook

---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080
  
  tasks:
    - name: update
      apt:
        update_cache=yes
      tags: 
        - update apt
          
    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest
      notify:
      - restart nginx
      tags:
      - nginx-package
    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx      
      tags: nginx-configuration
  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded

Проверяем на ошибки изапускаем
ansible-playbook --check nginx.yml
все норм

Запускаем
root@testvm:/home/Ansible# ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] **************************************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [update] *******************************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX] ****************************************************************************************************************************************************************************************************************
changed: [nginx]

TASK [NGINX | Create NGINX config file from template] ***************************************************************************************************************************************************************************************
changed: [nginx]

RUNNING HANDLER [restart nginx] *************************************************************************************************************************************************************************************************************
changed: [nginx]

RUNNING HANDLER [reload nginx] **************************************************************************************************************************************************************************************************************
changed: [nginx]

PLAY RECAP **********************************************************************************************************************************************************************************************************************************
nginx                      : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Просматриваем проброс на хосте
root@testvm:/home/Ansible/templates# vagrant ssh
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-92-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System information as of Fri Mar 15 11:32:12 AM UTC 2024

  System load:  0.00439453125      Processes:             134
  Usage of /:   12.6% of 30.34GB   Users logged in:       0
  Memory usage: 31%                IPv4 address for eth0: 10.0.2.15
  Swap usage:   0%                 IPv4 address for eth1: 192.168.11.150


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Fri Mar 15 11:22:12 2024 from 10.0.2.2
vagrant@nginx:~$ curl http://192.168.11.150:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>









