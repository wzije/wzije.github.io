+++
tags = ["docker", "connection"]
title= "Koneksi container dengan host (localhost)"
date= 2022-03-01T12:38:51+07:00
weight= 15
+++

## Case 
Disini kita akan mengkoneksikan docker container dengan mysql yang terinstall di host mesin (luar container). 

## Prequist  
- Mesin   : ubuntu 20.04 LTS
- Mysql   : mysql  Ver 14.14 Distrib 5.7.37,
- Docker  : Docker version 20.10.12, build e91ed57
- Docker Compose 

## Attention
1. Pastikan mysql open akses / specific ip address, agar bisa diakses secara remote dari ip address tertentu
2. Pastikan jalankan docker dengan extra network
3. pastikan port 3306 allow, ufw allow 3306

## Steps
### 1. Konfigurasi MySql 
 - pastikan bind-address terbuka ubah ke `0.0.0.0`, file `/etc/mysql/mysql.conf.d/mysqld.cnf`
    ```vim
    [mysqld]
    pid-file        = /var/run/mysqld/mysqld.pid
    socket          = /var/run/mysqld/mysqld.sock
    datadir         = /var/lib/mysql
    log-error       = /var/log/mysql/error.log
    
    # bagian ini
    bind-address    = 0.0.0.0 

    symbolic-links=0
    ```

    _note: jika ingin ubah port, tambahkan variabel port di file /etc/mysql/my.cnf_
    ```vim
    [mysqld]
    port=3305
    ```

- Set spesifik ip address untuk user mysql tertentu
    - mengecek user 
      ```sql 
        mysql> select user,host from user;

        +---------------+-----------+
        | user          | host      |
        +---------------+-----------+
        | simade        | %         |
        | mysql.session | localhost |
        | mysql.sys     | localhost |
        | root          | localhost |
        | simade        | localhost |
        +---------------+-----------+
        5 rows in set (0.00 sec)
      ```
    - Ubah identity user atau rename kemudian sesuaikan ip addressnya. Tanda `%` dipake agar bisa diakses dari semua host. Sedang disini kita akan gunakan spesifik ip address nya docker container.

        Untuk mendapatkan Ip adress container jalankan perintah berikut :
        ```vim
        docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' [container_id/name]

        172.31.0.2
        ```    

    - Setelah mendapatkan ip address-nya, tempel ip tersebut ke user mysql dengan perintah berikut dst :

        ```sql
        #Cara 1
        mysql> ALTER USER 'tarno'@'%' IDENTIFIED WITH mysql_native_password BY 'password';

        #Cara 2
        mysql> RENAME USER 'tarno'@'127.0.0.1' TO 'tarno'@'localhost';

        #Grant all
        mysql> GRANT ALL ON *.* TO 'tarno'@'localhost'; 

        #Flush
        mysql> FlUSH PRIVILEGE;
        ```

- Forward port mysql
    - cek status
        ```bash 
        $ ufw status 
        Status: active

        To                         Action      From
        --                         ------      ----
        22/tcp                     ALLOW       Anywhere
        80                         ALLOW       Anywhere
        443                        ALLOW       Anywhere
        63791                      ALLOW       Anywhere
        22/tcp (v6)                ALLOW       Anywhere (v6)
        80 (v6)                    ALLOW       Anywhere (v6)
        443 (v6)                   ALLOW       Anywhere (v6)
        63791 (v6)                 ALLOW       Anywhere (v6)

        $ ufw allow 3306 
        $ ufw status [number]
        $ ufw del [number]
        ```
- Sesuaikan policy requirement (lihat bagian Troubleshoot)


### 2. konfigurasi Docker Compose
- pastikan docker-compose.yaml file sbb :
    ```yaml
    version: "3.9"
    services:
    simade:
        image: ${REPOSITORY}
        container_name: my_name
        ports:
        - "80:80"
        restart: on-failure
        volumes:
        - /home/serverdpmd/project/upload/:/var/www/app/application/upload/
        - /home/serverdpmd/project/.env:/var/www/app/.env
        extra_hosts:
        # tambahkan bagian ini
        # atau gunakan perintah '--add-host=host.docker.internal:host-gateway' jika dijalankan manual
        - "host.docker.internal:host-gateway" 
    # konfigurasi network & global network (paling bawah), untuk mendapatkan ip statik
    networks:
      network:
        ipv4_address: 172.x.x.x

    networks:
    default:
        name: my_network
    #will be prefixed with 'parent_dir_name_' so for me, it will be 'mysql_network'
    network: 
        ipam:
            config:
                - subnet: 172.x.x.x/16
    ```

## Securing MySql
1. Ubah mysql port 
2. Jalankan `mysql_secure_installation` dan pastikan user root tidak dapat diakses sembarangan

 ## Troubleshoot :
1. - Error : `Your password does not satisfy the current policy requirements` 
   - Solusi : sesuaikan kebutuhan validate passwordnya seperti ini.

        ```sql
        mysql> SET GLOBAL validate_password_check_user_name=ON;
        Query OK, 0 rows affected (0.00 sec)

        mysql> SET GLOBAL validate_password_length = 4;
        Query OK, 0 rows affected (0.00 sec)

        mysql> SHOW VARIABLES LIKE 'validate_password%';
        +--------------------------------------+-------+
        | Variable_name                        | Value |
        +--------------------------------------+-------+
        | validate_password_check_user_name    | ON    |
        | validate_password_dictionary_file    |       |
        | validate_password_length             | 4     |
        | validate_password_mixed_case_count   | 1     |
        | validate_password_number_count       | 0     |
        | validate_password_policy             | LOW   |
        | validate_password_special_char_count | 1     |
        +--------------------------------------+-------+
        ```