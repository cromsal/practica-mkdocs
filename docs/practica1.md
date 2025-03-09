# Configuración de Infraestructura con Ansible

Este repositorio contiene varios playbooks de Ansible para configurar un entorno de WordPress con balanceo de carga, NFS y HTTPS.

## Playbooks Incluidos

### 1. Configuración de la Base de Datos
```--- 
- name: Playbook para configurar HTTPS
  hosts: backend_ip
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Quitamos la base de datos
      mysql_db:
        name: "{{ wordpress.db.name }}"
        state: absent
        login_user: root
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Creamos la base de datos
      mysql_db:
        name: "{{ wordpress.db.name }}"
          state: present
        login_user: root
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Quitamos el usuario si esiste
      mysql_user:
        name: "{{ wordpress.db.user }}"
        host: "{{ FRONTEND_NETWORK }}"
        state: absent
        login_user: root
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Creamos el usuario
      mysql_user:
        name: "{{ wordpress.db.user }}"
        password: "{{ wordpress.db.password }}"
        host: "{{ FRONTEND_NETWORK }}"
        priv: "{{ wordpress.db.name }}.*:ALL"
        state: present
        login_user: root
        login_unix_socket: /var/run/mysqld/mysqld.sock
```   
### 2. Configuración de WordPress
```--- 
- name: Configuracion de wordpress
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Quitamos el wp-cli
      file:
        path: "/tmp/wp-cli.phar"
        state: absent

    - name: Descargamos el wp-cli
      get_url:
        url: "https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar"
        dest: "/tmp/wp-cli.phar"
        mode: '0755'

    - name: movemos el wp-cli a la carpeta wp
      command: mv /tmp/wp-cli.phar /usr/local/bin/wp
      args:
        creates: /usr/local/bin/wp
    - name: Creamos el directorio de wordpress
      file:
        path: "{{ wordpress.directory }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Descargamos wordpress
      command: wp core download --locale=es_ES --path={{ wordpress.directory }} --allow-root

    - name: Creamos el archivo de configuracion
      command: wp config create --dbname={{ wordpress.db.name }} --dbuser={{ wordpress.db.user }} --dbpass={{ wordpress.db.password }} --dbhost={{ mysql.backend_ip1 }} --path={{ wordpress.directory }} --allow-root

    - name: Install WordPress
      command: wp core install --url={{ certbot.domain }} --title="{{ wordpress.titulo }}" --admin_user={{ wordpress.db.user }} --admin_password={{ wordpress.db.password }} --admin_email={{ certbot.email }} --path={{ wordpress.directory }} --allow-root

    - name: Cambiamos el dueño a www-data
      file:
        path: "{{ wordpress.directory }}"
        state: directory
        recurse: yes
        owner: www-data
        group: www-data
    - name: Instalamos los temas
      command: wp theme install mindscape --activate --path={{ wordpress.directory }} --allow-root

    - name: Configuramos los linkspermanentes
      command: wp rewrite structure '/%postname%/' --path={{ wordpress.directory }} --allow-root

    - name: Instalamos los plugins de seguridad
      command: wp plugin install wps-hide-login --activate --path={{ wordpress.directory }} --allow-root

    - name: Ponemos la url privada
      command: wp option update whl_page "{{ wordpress.hideurl }}" --path={{ wordpress.directory }} --allow-root

    - name: Copiamos el htaccess
      copy:
        src: "../htaccess/.htaccess"
        dest: "{{ wordpress.directory }}/.htaccess"
        owner: www-data
        group: www-data
        mode: '0644'

    - name: Habilitamos https en el directorio
      lineinfile:
        path: "{{ wordpress.directory }}/wp-config.php"
        insertafter: "COLLATE"
        line: "$_SERVER['HTTPS'] = 'on';"
        state: present
        - name: Cambiar el dueño a www-data de nuevo
      file:
        path: "{{ wordpress.directory }}"
        state: directory
        recurse: yes
        owner: www-data
        group: www-data
```  
### 3. Configuración del Servidor de Base de Datos
```--- 
- name: Playbook para configurar HTTPS
  hosts: backend_ip
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Actualizamos los repositorios
      apt:
        update_cache: yes

    - name: Instalamos mysql-server
      apt:
        name: mysql-server
        state: present

    - name: Instalamos el modulo de python para mysql-server
      apt:
        name: python3-pymysql
        state: present

    - name: Configuramos el /etc/mysql
      replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: 127.0.0.1
        replace: 0.0.0.0

    - name: Reseteamos mysql-server
      service:
        name: mysql
        state: restarted
```  
### 4. Configuración del Frontend
```--- 
- name: Configuracion de frontend
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Actualizamos los repositorios
      apt:
        update_cache: yes

    - name: Actualizamos el ubuntu
      apt:
        upgrade: yes

    - name: Instalamos apache2 
      apt:
        name: apache2
        state: present

    - name:
      apache2_module:
        name: rewrite
        state: present

    - name: Copiamos el 000-default dentro de apache2
      copy:
        src:  ../templates/000-default.conf
        dest: /etc/apache2/sites-available

    - name: Instalamos los modulos de php
      apt:
        name:
          - php 
          - libapache2-mod-php
          - php-mysql
        state: present
        
    - name:
      service:
        name: apache2
        state: restarted

    - name:
      file:
        path: /var/www/html
        mode: 755
        owner: www-data
        group: www-data
        recurse: yes
```

### 5. Configuración del letsencrypt
```--- 
- name: Configuramos a donde ira la configuracion.
  hosts: loadbalancer_ip
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Desinstalar instalaciones previas de Certbot
      apt:
        name: certbot
        state: absent

    - name: Instalar Certbot con snap
      snap:
        name: certbot
        classic: yes
        state: present

    - name: Solicitamos el certificado
      command:
        certbot --nginx \
        -m {{ certbot.email }} \
        --agree-tos \
        --no-eff-email \
        --non-interactive \
        -d {{ certbot.domain }}
```

### 6. Configuración del Balanceador de Carga  
```
--- 
- name: Configuracion de loadbalancer
  hosts: loadbalancer_ip
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Actualizamos los repositorios
      apt:
        update_cache: yes

    - name: Actualizamos el ubuntu
      apt:
        upgrade: yes
    
    - name: Instalamos el nginx
      apt:
        name: nginx
        state: present

    - name: Quitamos el sitio por /etc/nginx/sites-enabled/default
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent

    - name: Copiamos el loadbalancer en la pagina
      copy:
        src: ../templates/load-balancer.conf
        dest: /etc/nginx/sites-available/load-balancer.conf
        mode: '755'

    - name: Remplazamos la ip del frontend por la del mysql
      replace:
        path: /etc/nginx/sites-available/load-balancer.conf
        regexp: "IP_FRONTEND_1"
        replace: "{{ mysql.frontend_ip }}"
    
    - name: Remplazamos la ip del frontend por la del mysql
      replace:
        path: /etc/nginx/sites-available/load-balancer.conf
        regexp: "IP_FRONTEND_2"
        replace: "{{ mysql.frontend_ip2 }}"

    - name: Remplazamos la ip del frontend por la del mysql
      replace:
        path: /etc/nginx/sites-available/load-balancer.conf
        regexp: "LE_DOMAIN"
        replace: "{{ certbot.domain }}"

    - name:
      file:
        src: /etc/nginx/sites-available/load-balancer.conf
        dest: /etc/nginx/sites-enabled/load-balancer.conf
        state: link

    - name: Recargamos Nginx 
      systemd:
        name: nginx
        state: restarted
```        

### 7. Configuración de NFS client
```--- 
- name: Configuracion de nfs server
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Install NFS client package
      apt:
        name: nfs-common
        state: present
        update_cache: yes

    - name: Add NFS mount to /etc/fstab
      lineinfile:
        path: /etc/fstab
        line: "{{ nfs.NFS_SERVER_IP }}:/var/www/html /var/www/html nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0"
        insertafter: "LABEL=UEFI"

    - name: reiniciamos systemd daemon
      systemd:
        daemon_reload: yes

    - name: montar el NFS
      mount:
        path: /var/www/html
        src: "{{ nfs.NFS_SERVER_IP }}:/var/www/html"
        fstype: nfs
        state: mounted
```  

### 8. Configuración de NFS server
```--- 
- name: Configuracion de nfs server
  hosts: nfs_ip
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Instalamos nfs server
      apt:
        name: nfs-kernel-server
        state: present

    - name: Creamos el directorio html
      file:
        path: /var/www/html
        state: directory
        owner: nobody
        group: nogroup
        mode: 0777

    - name: Copiamos el archivo nfs
      copy:
        src: ../nfs/exports
        dest: /etc/exports
        owner: root
        group: root
        mode: 0644

    - name: actualizamos el archivo exports
      replace:
        path: /etc/exports
        regexp: 'FRONTEND_NETWORK'
        replace: "{{ nfs.FRONTEND_NETWORK }}"

    - name: Reiniciamos el NFS
      service:
        name: nfs-kernel-server
        state: restarted
```

## Variables
Las variables se encuentran en `/vars/variables.yml` y contienen la configuración personalizadas para la instalación.  