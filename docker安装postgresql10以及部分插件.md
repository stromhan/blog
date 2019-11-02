
# docker方式部署postgresql

## 1 安装docker
### 1.1 通过二进制包安装docker ###
到官网下载自己需要安装包

    wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.4.tgz

### 1.2 将安装包解压到/home/docker目录下

    chmod 777 /home/docker/docker-19.03.4/*
    cp /home/docker/docker-19.03.4/* /usr/local/bin/
### 1.3 将docker加入到系统服务
生成docker.service的文件并设定到/usr/lib/systemd/system目录下

    cat > /usr/lib/systemd/system/docker.service <<"EOF"
    [Unit]
    Description=Docker Application Container Engine
    Documentation=http://docs.docker.io
    [Service]
    Environment="PATH=/usr/local/bin:/bin:/sbin:/usr/bin:/usr/sbin"
    EnvironmentFile=-/run/flannel/docker
    ExecStart=/usr/local/bin/dockerd --log-level=error $DOCKER_NETWORK_OPTIONS
    ExecReload=/bin/kill -s HUP $MAINPID
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=infinity
    LimitNPROC=infinity
    LimitCORE=infinity
    Delegate=yes
    KillMode=process
    
    [Install]
    WantedBy=multi-user.target
    EOF
### 1.4 重启docker
    systemctl restart docker


##2 docker 上部署postgresql##
### 2.1制作postgresql的镜像
创建Dockerfile等制作镜像脚本的目录

    mkdir /home/postgresql10_docker
    cd /home/postgresql10_docker

创建Dockerfile

    vim Dockerfile    
Dockerfile  内容如下

    FROM centos:7.6.1810
    
    ENV PGDATA=/var/lib/pgsql/10/data
    ENV PGHOME=/usr/pgsql-10/ 
    ENV LANG en_US.UTF-8
    
    RUN  set -eux; \
        echo "export LANG=en_US.utf8" >> /root/.bash_profile \
        && yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm -y  \
        && yum install postgresql10-contrib postgresql10-server postgresql10-devel.x86_64 -y \
        && echo "export PATH=/usr/pgsql-10/bin:$PATH" >> /etc/profile.d/pg_env.sh \
        && echo "export LD_LIBRARY_PATH=$PGHOME/lib" >> /etc/profile.d/pg_env.sh \
        && echo "export PGDATA=$PGDATA"  >> /etc/profile.d/pg_env.sh \
        && echo "export PGHOME=$PGHOME"  >> /etc/profile.d/pg_env.sh \
        && echo "export LANG=en_US.utf8" >> /etc/profile.d/pg_env.sh \
        && chmod 777 /etc/profile.d/pg_env.sh \
        && echo "source /etc/profile.d/pg_env.sh" >> /var/lib/pgsql/.bash_profile \
        && /etc/profile.d/pg_env.sh \
        && su - postgres -c "initdb -E utf8" \
        #&& systemctl enable postgresql-10.service \
        && yum  install wget net-tools epel-release -y \
        && yum install postgis25_10 postgis25_10-client -y \
        && yum install ogr_fdw10 -y \
        && yum install pgrouting_10 -y \
        && yum install osm2pgrouting_10 -y \
        && yum install gcc -y \
        && yum install make -y \
        && yum install git -y \
        && git clone git://git.postgresql.org/git/pldebugger.git \
        && mv pldebugger /usr/pgsql-10/share/contrib \
        && yum install openssl-devel -y \
        && yum install postgresql10-devel.x86_64 -y \
        && cd /usr/pgsql-10/share/contrib/pldebugger \
        && source /etc/profile.d/pg_env.sh \
        && USE_PGXS=1 make clean \
        && USE_PGXS=1 make  \
        && USE_PGXS=1 make install \
        && sed -i s/"#shared_preload_libraries = ''"/"shared_preload_libraries ='\/usr\/pgsql-10\/lib\/plugin_debugger.so'"/g $PGDATA/postgresql.conf \
        && yum install pgagent_10.x86_64 -y \
        && su - postgres -c "pg_ctl start" && psql -U postgres -d postgres -f /usr/share/pgagent_10-4.0.0/pgagent.sql \
        && yum install lbzip2.x86_64 -y \
        && cd /home \
        && wget -P /home http://www.xunsearch.com/scws/down/scws-1.2.3.tar.bz2 \
        && git clone https://github.com/amutu/zhparser.git \
        && tar -xf scws-1.2.3.tar.bz2 \
        && cd scws-1.2.3 && ./configure && make install \
        && export SCWS_HOME=/usr/local \
        && cd /home/zhparser &&  PG_CONFIG=/usr/pgsql-10/bin/pg_config make && make install \
        #&& yum -y install perl perl-devel \
        #&& yum install -y php php-devel \
        #&& yum install -y httpd \
        #&& yum -y install httpd-devel \
        #&& yum -y install perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker \
        #&& yum -y install perl-CPAN \
        #&& cpan Text::CSV \
        #&& cpan JSON::XS \
        #&& wget -P /home/ https://github.com/darold/pgbadger/archive/v10.3.tar.gz \
        #&& cd /home/ && tar -xvf v10.3.tar.gz \
        #&& cd pgbadger-10.3 \
        #&& perl Makefile.PL \
        #&& make && make install \
        && yum install -y cmake3 && ln -s /usr/bin/cmake3 /usr/bin/cmake    \
        && wget -P /home/ https://github.com/timescale/timescaledb/archive/1.3.0.tar.gz \
        && cd /home/ && tar -xvf 1.3.0.tar.gz && cd timescaledb-1.3.0 \
        && ./bootstrap \
        && cd ./build && make \
        && make install \
        && sed -i s/"shared_preload_libraries ='\/usr\/pgsql-10\/lib\/plugin_debugger.so'"/"shared_preload_libraries ='\/usr\/pgsql-10\/lib\/plugin_debugger.so,timescaledb'"/g $PGDATA/postgresql.conf \
        &&  wget -P /home https://github.com/pipelinedb/pipelinedb/releases/download/1.0.0rev4/pipelinedb-postgresql-10-1.0.0-4.centos7.x86_64.rpm \
        && cd /home \
        && rpm -ivh pipelinedb-postgresql-10-1.0.0-4.centos7.x86_64.rpm \
        && sed -i s/"shared_preload_libraries ='\/usr\/pgsql-10\/lib\/plugin_debugger.so,timescaledb'"/"shared_preload_libraries ='\/usr\/pgsql-10\/lib\/plugin_debugger.so,timescaledb,pipelinedb'"/g $PGDATA/postgresql.conf
    
    
    COPY docker-entrypoint.sh /usr/local/bin/
    RUN chmod -R 777 /usr/local/bin/docker-entrypoint.sh
    RUN ln -s /usr/local/bin/docker-entrypoint.sh /
    ENTRYPOINT ["docker-entrypoint.sh"]
    
    VOLUME $PGDATA
    
    EXPOSE 5432
    
    CMD ["postgres"]

创建docker-entrypoint.sh

    cd /home/postgresql10_docker
    vim docker-entrypoint.sh
内容如下：

    #!/bin/bash
    
    if [ "$1" = "postgres" ];then
            echo "kernel.sem = 250 64000 32 256" >> /etc/sysctl.conf
            free -b | awk '/Mem/ {print "kernel.shmmax="$2/2}' >> /etc/sysctl.conf
            free -k | awk '/Mem/ {print "kernel.shmall="$2*0.8}' >> /etc/sysctl.conf
            echo "kernel.shmmni = 819200" >> /etc/sysctl.conf
            echo "host      all     all     0.0.0.0/0       md5" >> $PGDATA/pg_hba.conf
            su - postgres -c "/usr/pgsql-10/bin/postgres -D $PGDATA -c config_file=$PGDATA/postgresql.conf"
    fi

build 镜像
    cd /home/postgresql10_docker
    docke build -t postgres_10_centos7.6:1.0 .
    
查看镜像
 
    docker images
输出结果：
 
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    postgres_10_centos7.6   1.0                 232ef0628e1f        6 hours ago         952MB
    centos              7.6.1810            f1cb7c7d58b7        7 months ago        202MB

### 2.2 启动一个postgresql容器，放后台运行
设置启动端口也为5432

    docker run -d -p 5432:5432 postgres_10_centos7.6:1.0
检查启动的容器
    
    docker ps -a
输出如下

    CONTAINER ID        IMAGE                       COMMAND                  CREATED              STATUS              PORTS                    NAMES
    534149dc7726        postgres_10_centos7.6:1.0   "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:5432->5432/tcp   sharp_pasteur


进入容器查看postgres进程是否正常启动

    [root@iZuf678t3hp8rp7xchc3aoZ ~]# docker exec -it 534149dc7726 ps -ef | grep postgres
输出结果如下

    root        10     1  0 15:53 ?        00:00:00 su - postgres -c /usr/pgsql-10/b
    postgres    11    10  0 15:53 ?        00:00:00 /usr/pgsql-10/bin/postgres -D /v
    postgres    24    11  0 15:53 ?        00:00:00 postgres: logger process   
    postgres    26    11  0 15:53 ?        00:00:00 postgres: checkpointer process  
    postgres    27    11  0 15:53 ?        00:00:00 postgres: writer process   
    postgres    28    11  0 15:53 ?        00:00:00 postgres: wal writer process   
    postgres    29    11  0 15:53 ?        00:00:00 postgres: autovacuum launcher pr
    postgres    30    11  0 15:53 ?        00:00:00 postgres: stats collector proces
    postgres    31    11  0 15:53 ?        00:00:00 postgres: bgworker: scheduler   
    postgres    32    11  0 15:53 ?        00:00:00 postgres: bgworker: TimescaleDB 
    postgres    33    11  0 15:53 ?        00:00:00 postgres: bgworker: logical repl

宿主机上查看5432端口是否启动

    netstat -nlap | grep 5432
输出结果如下

    tcp6       0      0 :::5432                 :::*                    LISTEN      22287/docker-proxy  


此时一个基于docker的postgresql已经启动成功。

## 补充说明 ##
推荐将数据目录挂载在宿主机上
