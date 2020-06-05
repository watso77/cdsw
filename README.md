# CDSW(Cloudera Data Science Workbench) 설치

###### 																															         	    ``writer`` : ``GIT bhhan`` 

## 목차

### Environment & Requirement

### CDSW Repoistory 구성

### DNS 서버 설정

### CDSW Server Preconfiguration

### CDSW Install 

### GPU Setting 

### Custom Docker Image

### Trouble shooting



### 서버구성

| 노드역할   | IP             | HOST_NAME            | DISK        |
| ---------- | -------------- | -------------------- | ----------- |
| CDH        | 10.200.101.122 | cdh02.goodmit.co.kr  |             |
| CDSW       | 10.11.2.191    | cdsw02.goodmit.co.kr | block space |
| Repository | 10.200.101.253 | repo.goodmit.co.kr   |             |
| DNS Server | 10.200.101.121 | cdh02.goodmit.co.kr  |             |



## Install Versions

- CDSW 1.7.2 기준으로 작성됨

## CDSW Requirements

- Cloudera Manager and CDH 

- Operating System 

- JDK 1.8.0.xx

- Networking and Security

- Recommnanded Hardware

  

### Cloudera Manager and CDH

| CDH                                            | Cloudera Manager                                             |
| ---------------------------------------------- | ------------------------------------------------------------ |
| CDH 5.10 or higher. <br /> CDH 6.1.x or higher | Cloudera Manager 5.13 or higher<br />Cloudera Manager 6.1.x or higher |

### Operating System

- RHEL/CentOS/Orcle Linux RHCK : 7.2 ~ 7.7까지 지원

### JDK Requirements

- JDK 8

### Network Requirements

- CDSW는 내부적으로 도메인으로 통신되기 때문에 DNS 서버가 꼭 있어야함.
- ***.cdsw.company.com** 와 같이 wildcard subdomain으로 접속이 허용되야 한다.
- SELinux Disabled 되야함(**preconfig에서 작업**)

### Recommanded Hardware

| ResourceType             | Master               | Workes               | Notes                                                        |
| ------------------------ | -------------------- | -------------------- | ------------------------------------------------------------ |
| CPU                      | 16+ CPU (vCPU) cores | 16+ CPU (vCPU) cores |                                                              |
| RAM                      | 32+GB                | 32+GB                |                                                              |
| **Diskspace**            |                      |                      |                                                              |
| Root Volume              | 100+GB               | 100+GB               | Root 볼륨 분할 시 '/'에 최소 20GB는 필요                     |
| Application Block Device | 1 TB                 |                      | mounted to **/var/lib/cdsw**                                 |
| Docker Block Device      | 1 TB                 | 1 TB                 | The Docker Block Device is required on *all* Master and Worker hosts. |



## 설치 사전 준비 작업

### CDSW Repoistory 구성

- **Repository Server ssh로 접속**

- cdsw 설치용 pacer file donwload - [CDSW archive](https://archive.cloudera.com/cdsw1/1.7.2/parcels/)에서 아래와 같이 1.7.2 과 manifest.json파일(총 3개)을 버전을 다운로드 한다.

  - [CDSW-1.7.2.p1.2066404-el7.parcel](https://archive.cloudera.com/cdsw1/1.7.2/parcels/CDSW-1.7.2.p1.2066404-el7.parcel)
  - [CDSW-1.7.2.p1.2066404-el7.parcel.sha](https://archive.cloudera.com/cdsw1/1.7.2/parcels/CDSW-1.7.2.p1.2066404-el7.parcel.sha)
  - [manifest.json](https://archive.cloudera.com/cdsw1/1.7.2/parcels/manifest.json)

- repository 구성된 서버의 /var/www/html/cdsw/cdsw1.7에 다운로드 받은 3개의 파일을 복사해 놓는다.

- Repository 만들기 (안해도 됨)

  ```bash
  ## repository 생성
  createrepo /var/www/html/cdsw/cdsw1.7/
  
  ## 폴더 및 파일 접근 권한 수정
  chmod -R ugo+rX /var/www/html/cdsw/
  
  ## yum repository 설정
  vi /etc/yum.repos.d/cdsw1.7.repo
  [CDSW1.7]
  name=cdsw1.7 repository
  baseurl=http://10.200.101.253/cdsw/cdsw1.7
  gpgcheck=0
  enabled=1
  
  ## yum clean & repository 확인
  yum clean all
  yum repolist
  ```

- 접속 확인  - http://10.200.101.253/cdsw/cdsw1.7 접속 후 다운로드 한 파일이 보이면 성공!

### DNS 설정

- dns 서버가 없을 경우 cdsw용 dns 서버를 구성해야한다.

#### dnsmasq : 간단하게 설정 가능하므로 권장

```ba
yum install -y dnsmasq

vi /etc/dnsmasq.conf
# web-server.
#address=/double-click.net/127.0.0.1
## cdsw.goodmit.co.kr 및  wildcard를 위해 추가
address=/cdsw.goodmit.co.kr/10.11.2.191
address=/.cdsw.goodmit.co.kr/10.11.2.191
address=/cdh.goodmit.co.kr/10.11.2.192  ## cdh(옵션)
server=8.8.8.8													## 외부dns (옵션)

systemctl restart dnsmasq
```



#### dns server : dnsmasq 사용 할 수 없을 경우에 권장

- dns server 설치

  ```bash
  yum install bind bind-chroot
  ## 파일수정
  vi /etc/named.conf 
  listen-on port 53 { any; }; ## <== 변경
  listen-on-v6 port 53 { none; }; ## <== 변경
  directory       "/var/named";
  dump-file       "/var/named/data/cache_dump.db";
  statistics-file "/var/named/data/named_stats.txt";
  memstatistics-file "/var/named/data/named_mem_stats.txt";
  recursing-file  "/var/named/data/named.recursing";
  secroots-file   "/var/named/data/named.secroots";
  allow-query     { any; }; ## <== 변경
  ```

- 정방향 규칙 설정

  ```bash
  /etc/named.rfc1912.zones (하단에 추가)
  				zone "cdsw02.goodmit.com" IN {
  					  type master;
  					  file "cdsw.zone";
  					  allow-update { none; };
  				};
  				
  				zone "2.11.10.in-addr.arpa" IN {
  					  type master;
  					  file "cdsw.rev";
  					  allow-update { none; };
  				};
  				
  ## 정방향
  vi /var/named/cdsw.zone (정방향)
  			$TTL    3H
  			@       SOA     @       root. (  2016060801   1D   1H   1W   1H  )
  											IN      NS      @
  											IN      A       10.11.2.192 ##cdsw02.goodmit.com 
  			*       IN      A       10.11.2.192 ##*.cdsw02.goodmit.com
  
  ## 역방향
  vi /var/named/cdsw.rev (역방향)
  			$TTL    3H
  			@       SOA     @       root. (  2016060801   1D   1H   1W   1H  )
  							IN      NS      @
  							IN      A       10.11.2.192
  			199       IN        PTR       cdsw02.goodmit.com.			
  ```



#### CDSW 서버 접속 => 네임서버 추가

- ```bash
  echo "nameserver 10.200.101.198"  >> /etc/resolv.conf
  ## 확인
  cat /etc/resolv.conf 
  ```

- 네임서버 접속 => DNS Service 재시작 및 동작 확인

  ```bash
  ## 재시작(네임서버 구성 할때만)
  systemctl restart named
  
  ## 서비스 확인
  nslookup cdsw02.goodmit.com
  nslookup test.cdsw02.goodmit.com
  
  ## nslookup 안될 때 package 설치
  yum install -y bind-utils
  ```




### CDSW, Spark2 서비스 추가하기

- CM Server 에서 수행

  - [Cloudera Data Science Workbench CSD](https://archive.cloudera.com/cdsw1/1.7.2/csd/CLOUDERA_DATA_SCIENCE_WORKBENCH-CDH6-1.7.2.jar) 및 [Spark2 CDS](http://archive.cloudera.com/spark2/csd/SPARK2_ON_YARN-2.4.0.cloudera2.jar) 파일 다운로드

    (단, CDH 6.X 버전은 Spark2가 기본 패키지에 포함되어 있어 별도 설치 필요 없음)

  - /opt/cloudera/csd 에 다운로드 파일 복사 및 권한 수정

    ```bash
    hdfs dfs -mkdir /user/<username>
    hdfs dfs -chown <username>:<username> /user/<username>
    
    cp SPARK2*.jar /opt/cloudera/csd
    cp CLOUDERA*.jar /opt/cloudera/csd
    
    cd /opt/cloudera/csd
    chown cloudera-scm:cloudera-scm *.jar
    chmod 644 *.jar
    systemctl restart cloudera-scm-server
    
    
    ```


  - CM Admin > Clusters > Cloudera Management Service > Action >  Restart

  

### CDSW 서버 초기설정

cdsw  서버 접속 ( ex : root/root)

```bash
#hostname 설정
hostnamectl set-hostname cdsw02.gitcluster.com

## hostname 확인
cat /etc/hosts
echo -e "10.200.101.122 cdh02.goodmit.com cdh02 
10.11.2.192 cdsw02.goodmit.com cdsw02" >> /etc/hosts	

##ssh key 생성
ssh-keygen 
# enter key 3번 (패스워드 입력없이 ssh 접속하고 싶을 때는 특별한 입력없이 엔터 3번)
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.200.101.122
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.11.2.191

#방화벽 해제
systemctl stop firewalld
systemctl disable firewalld

## swappiness 설정 
vi /etc/sysctl.conf
vm.swappiness=1  #<== 추가 및 저장

## 적용 및 확인
sysctl -p 
sysctl vm.swappiness

##SELinux 해제 
vi /etc/selinux/config
SELINUX=disabled  #<== 변경

## 재부팅
reboot

## NTP 설치 및 동기화
yum install -y ntp --nogpgcheck

vi /etc/ntp.conf
중간에 server0~3 전부 삭제 또는 주석처리 후 ntp 서버 ip로 대체
server npt_host_id iburst

# ntp start & enable
systemctl start ntpd
systemctl enable ntpd

# 서버 시간 동기화
ntpq -pn

# ntp 서버 시간 설정
timedatectl set-timezone Asia/Seoul

# 시간 확인
date

#ntpd를 enable 했는데 재부팅시 자동으로 올라오지 않을떄 확인
systemctl disable chronyd
```



### OpenJDK 설치 

```bash
yum install -y java-1.8.0-openjdk*
```



### JDK 1.8 설치 ( OpenJDK 사용 불가 시)

```bash
# 기존 open jdk 삭제 ( 설치되 있으면...)
rpm -qa | grep jdk
java-1.8.0-openjdk-headless-1.8.0.181-7.b13.el7.x86_64
copy-jdk-configs-3.3-10.el7_5.noarch

yum remove java-1.8.0-openjdk-headless-1.8.0.181-7.b13.el7.x86_64

yum install -y java-1.8.0-openjdk*

#java home path 설정
echo -e "export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera 
			export PATH=$JAVA_HOME/bin:$PATH" >> ~/.bashrc
			
#적용
source ~/.bashrc
```



### Transparent Huge Page Compaction 설정해제

```bash
vi /etc/rc.d/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 임시
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# AnonHugePages, HugePages_Total, HugePages_Free, HugePages_Rsvd, HugePages_Surb -> 0, 0, 0, 0 확인하기
cat /proc/meminfo  

chmod +x /etc/rc.d/rc.local
```

### User Process Limit & File ulimit 설정

```bash
ulimit -u 65536
ulimit -n 1048576

vi /etc/security/limits.conf
root hard nofile 1048576
root soft nofile 1048576
```

### Block device 확인

Docker 이미지가 배포될 block device 준비

- fdisk /dev/sda를 파티션 나누는예

- 400GB로 용량 지정(단, cloudera에서는  1TB로 권장)

  (단, format이나 mount 필요없음)

```bash
[root@cdsw02 ~]# fdisk /dev/sda
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): 
Using default response p
Partition number (3,4, default 3): 
First sector (978733056-1953525167, default 978733056): 
Using default value 978733056
Last sector, +sectors or +size{K,M,G} (978733056-1953525167, default 1953525167): +400G  # <== 400GB로 설정 
Partition 3 of type Linux and of size 400 GiB is set

Command (m for help): p # <== 확인

Disk /dev/sda: 1000.2 GB, 1000204886016 bytes, 1953525168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000f17f2

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   978733055   488316928   8e  Linux LVM
/dev/sda3       978733056  1817593855   419430400   83  Linux

Command (m for help): w
The partition table has been altered!

reboot -h now  ## rebook 안하면 lsblk로 새로 분할된 파티션 안나오며, 바로 설치 시 block device 오류남. 

## lsblk로 확인!!!
[root@cdsw02 ~]# lsblk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0 931.5G  0 disk 
├─sda1            8:1    0     1G  0 part /boot
├─sda2            8:2    0 465.7G  0 part 
│ ├─centos-root 253:0    0   450G  0 lvm  /
│ └─centos-swap 253:1    0  15.7G  0 lvm  [SWAP]
└─sda3            8:3    0   400G  0 part 

[root@cdsw02 ~] fdisk -l
/dev/sda3       978733056  1817593855   419430400   83  Linux  
```



## CDSW 설치



### Add Host



### cm에서 cdsw 서비스 추가

- CDSW Parcel download

  - CM Admin console login

  - Parcel > 구성 클릭 >  원격 Parcel 리포지토리 URL에 http://10.200.101.253/cdsw/cdsw1.7  다운로드 URL 추가

    ![스크린샷 2020-05-21 오후 6.05.15](/Users/roger/Desktop/스크린샷 2020-05-21 오후 6.05.15.png)

  - Hosts > Parcels 클릭 

    - cdsw ~~및 spark~~ **다운로드** 클릭 
    - 다운로드 완료 후 **활성화 클릭**

- CDSW 서비스 추가 

  - Home > 상태 > 서비스 추가 클릭

    ![스크린샷 2020-05-15 오후 3.16.39](/Users/roger/Desktop/스크린샷 2020-05-15 오후 3.16.39.png)

  

  - CDSW 서비스 선택 > 다음

    ![스크린샷 2020-05-18 오후 5.34.59](/Users/roger/Desktop/스크린샷 2020-05-18 오후 5.34.59.png)

  

  - 역할할당, 1대에 설치 시 Master Node만 지정(Worker노드 지정시 설치 오류남) > 다음

    ![스크린샷 2020-05-18 오후 5.19.51](/Users/roger/Desktop/스크린샷 2020-05-18 오후 5.19.51.png)

  

  - 서버 및 block device 정보 입력 > 다음

  ![스크린샷 2020-05-18 오후 5.20.01](/Users/roger/Desktop/스크린샷 2020-05-18 오후 5.20.01.png)

  

  - cdsw intall & compleate

    ```bash
    ## docker 이미지 정상 load 확인 ( Status가 모두 Running으로 되야 정상 )
    cdsw status
    ---------------------------------------------------------------------------------------
    |                       NAME                      |   READY   |     STATUS    |   RESTARTS   |           CREATED-AT          |      POD-IP     |     HOST-IP     |            ROLE            |
    -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    |             archiver-5b75675b8-c798d            |    1/1    |    Running    |      0       |   2020-05-19 15:00:40+00:00   |   100.66.0.19   |   10.11.2.192   |          archiver          |
    |   cdsw-compute-pod-evaluator-5bdf55d48b-rppbs   |    1/1    |    Running    |      0       |   2020-05
    ... 중략 ...
    
    All required pods are ready in cluster default.
    All required Application services are configured.
    All required secrets are available.
    Persistent volumes are ready.
    Persistent volume claims are ready.
    Ingresses are ready.
    Checking web at url: http://cdsw02.goodmit.co.kr
    OK: HTTP port check
    Cloudera Data Science Workbench is ready!
    ```



## GPU Setting 

```bas
## GPU pci 슬롯 확인용(선택)
# yum install pciutils
lspci | grep -i VGA
01:00.0 VGA compatible controller: NVIDIA Corporation GM107 [GeForce GTX 750] (rev a2)


## Cloudera Data Science Workbench to use GPUs

# Disable Nouveau Driver
cat <<EOT >> /etc/modprobe.d/blacklist.conf
blacklist nouveau
EOT
mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak  
dracut -v /boot/initramfs-$(uname -r).img $(uname -r)
yum update -y
reboot
# Set Up the Operating System and Kernel
# strace64
yum install -y flex gcc gcc-c++  redhat-rpm-config strace \
  rpm-build make pkgconfig gettext automake \
  gdb bison libtool autoconf gcc-c++ gcc-gfortran \
  binutils rcs patchutils wget
yum install -y kernel-devel-`uname -r`
# Install the NVIDIA Driver on GPU Nodes
export NVIDIA_DRIVER_VERSION=390.67
wget http://us.download.nvidia.com/XFree86/Linux-x86_64/390.67/NVIDIA-Linux-x86_64-${NVIDIA_DRIVER_VERSION}.run
chmod 755 ./NVIDIA-Linux-x86_64-$NVIDIA_DRIVER_VERSION.run
./NVIDIA-Linux-x86_64-$NVIDIA_DRIVER_VERSION.run -asq
/usr/bin/nvidia-smi -pm 1
/usr/bin/nvidia-smi

## /usr/bin/nvidia-smi 실행결과 
Wed May 20 01:25:14 2020       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.67                 Driver Version: 390.67                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 750     On   | 00000000:01:00.0 Off |                  N/A |
| 35%   43C    P8     1W /  38W |      0MiB /  2001MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

- CM에서 GPU 활성화

  - CM > CDSW > Configuration > **Enable GPU Support** 체크(활성화) > 서비스 재기동

  - CDSW CLI > cdsw status로 모든 이미지 Status Running 확인

  - CDSW Web UI 로그인 > Dashboard에서 GPU 그래프 활성화 확인

    ![스크린샷 2020-05-19 오후 5.07.11](/Users/roger/Desktop/스크린샷 2020-05-19 오후 5.07.11.png)


## Custom Docker Image 

### docker 설치



![common_docker_structure](/Users/roger/Documents/goodmit/hadoop/02_cdsw/common_docker_structure.png)

### docker 이미지 build

#### cuda build

- 01.cuda.Dockerfile

```bash
FROM  docker.repository.cloudera.com/cdsw/engine:10

#RUN NVIDIA_GPGKEY_SUM=d1be581509378368edeec8c1eb2958702feedf3bc3d17011adbf24efacce4ab5 && \
#    NVIDIA_GPGKEY_FPR=ae09fe4bbd223a84b2ccfce3f60f4b3d7fa2af80 && \
#    apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub && \
#    apt-key adv --export --no-emit-version -a $NVIDIA_GPGKEY_FPR | tail -n +5 > cudasign.pub && \
#    echo "$NVIDIA_GPGKEY_SUM  cudasign.pub" | sha256sum -c --strict - && rm cudasign.pub && \
#    echo "deb http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/cuda.list

RUN release="ubuntu"$(lsb_release -sr | sed -e "s/\.//g") && \
    echo $release &&  \
    apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/"$release"/x86_64/7fa2af80.pub && \
    echo "deb http://developer.download.nvidia.com/compute/cuda/repos/$release/x86_64 /" > /etc/apt/sources.list.d/cuda.list  && \
    echo "deb http://developer.download.nvidia.com/compute/machine-learning/repos/$release/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list

ENV CUDA_VERSION 10.0.130
LABEL com.nvidia.cuda.version="${CUDA_VERSION}"

ENV CUDA_PKG_VERSION 10-0_$CUDA_VERSION-1
RUN apt-get update &&  \
    apt-get install -y --no-install-recommends cuda-10-0 cuda-cudart-10-0 && \
    ln -s cuda-10.0 /usr/local/cuda && \
    rm -rf /var/lib/apt/lists/*

RUN echo "/usr/local/cuda/lib64" >> /etc/ld.so.conf.d/cuda.conf && \
    ldconfig

RUN echo "/usr/local/nvidia/lib" >> /etc/ld.so.conf.d/nvidia.conf && \
    echo "/usr/local/nvidia/lib64" >> /etc/ld.so.conf.d/nvidia.conf

ENV PATH /usr/local/nvidia/bin:/usr/local/cuda/bin:${PATH}
ENV LD_LIBRARY_PATH /usr/local/nvidia/lib:/usr/local/nvidia/lib64


ENV CUDNN_VERSION 7.5.1.10
LABEL com.nvidia.cudnn.version="${CUDNN_VERSION}"

RUN apt-get update && apt-get install -y --no-install-recommends \
            libcudnn7=$CUDNN_VERSION-1+cuda10.0 && \
    apt-mark hold libcudnn7 && \
    rm -rf /var/lib/apt/lists/*
```

- 01_make_file.sh : cuda 이미지 build 실행

```bash
docker build --network=host -t cuda10.docker.repository.cloudera.com/cdsw/engine:10 . -f 01.cuda.Dockerfile
```



#### conda Image 빌드

- 02.conda.Dockerfile

```bash
FROM  cuda10.docker.repository.cloudera.com/cdsw/engine:10

RUN cd /tmp/ && \
    apt-get update &&  \
    apt-get install -y --no-install-recommends  \
            libssl-dev \
            libmariadb-client-lgpl-dev \
            mysql-client libmysqlclient20 \
            libxml2-dev  libnlopt-dev  \
            unixodbc-dev iodbc libiodbc2  \
            xorg libx11-dev  libglu1-mesa-dev  libfreetype6-dev   \
            libgmp-dev   libblas-dev libblas3  && \
    wget -O impala.deb --no-check-certificate https://downloads.cloudera.com/connectors/impala_odbc_2.5.41.1029/Debian/clouderaimpalaodbc_2.5.41.1029-2_amd64.deb && \
    wget -O hive.deb --no-check-certificate  https://downloads.cloudera.com/connectors/ClouderaHive_ODBC_2.6.4.1004/Debian/clouderahiveodbc_2.6.4.1004-2_amd64.deb && \
    dpkg -i  impala.deb hive.deb  && \
    rm -rf  *.deb && rm -rf /var/lib/apt/lists/* 
  

RUN mkdir -p /opt/conda/envs/python3.6  && \
    conda install -y nbconvert python=3.6.9 -n python3.6 && \
    conda install -y -n python3.6 bokeh  && \
    conda install -y -n python3.6 gensim  && \
    conda install -y -n python3.6 glob2  && \
    conda install -y -n python3.6 h5py  && \
    conda install -y -n python3.6 joblib  && \
    conda install -y -n python3.6 mpi4py  && \
    conda install -y -n python3.6 multiprocess  && \
    conda install -y -n python3.6 nltk  && \
    conda install -y -n python3.6 pandas  && \
    conda install -y -n python3.6 pymysql  && \
    conda install -y -n python3.6 pyodbc  && \
    conda install -y -n python3.6 scipy  && \
    conda install -y -n python3.6 statsmodels  && \
    conda install -y -n python3.6 statsd  && \
    conda install -y -n python3.6 tqdm  && \
    conda install -y -n python3.6 seaborn  && \
    conda install -y -n python3.6 matplotlib  && \
    conda install -y -n python3.6 pymysql scikit-learn 
     
RUN /opt/conda/envs/python3.6/bin/pip install gputil gym  jupyterlab  && \
    pip3 install jupyterlab 

RUN cd /tmp/ && \
    wget --no-check-certificate  https://jaist.dl.sourceforge.net/project/libpng/zlib/1.2.9/zlib-1.2.9.tar.gz  && \
    tar -xvf zlib-1.2.9.tar.gz && \
    cd zlib-1.2.9   && \
    ./configure &&  make && make install && \
    cd /usr/lib/x86_64-linux-gnu/  && \
    ln -s -f /usr/local/lib/libz.so.1.2.9/lib libz.so.1 && \
    cd /tmp/ && rm -rf zlib-1.2.9

```

- 02_make.sh

```bash
docker build --network=host -t conda.docker.repository.cloudera.com/cdsw/engine:10 . -f  02.conda.Dockerfile
```



#### Tensorflow 이미지 빌드

- 03.01.tensorflow2.0.Dockerfile

```bash
FROM  conda.docker.repository.cloudera.com/cdsw/engine:10

RUN /opt/conda/envs/python3.6/bin/pip  install tensorflow-gpu==2.0

```

- 03_01_make.sh

```bash
SITE_DOMAIN=$1

echo "SITE_DOMAIN=$SITE_DOMAIN"

if [ "$SITE_DOMAIN"  == "" ]; then

   echo "example : $0  goodmit.com "
   exit 1

fi

docker build --network=host -t tensorflow2.0.${SITE_DOMAIN}/cdsw/engine:10 . -f  03.01.tensorflow2.0.Dockerfile


```



- 03.02.tensorflow1.12.Dockerfile

```bash
FROM  conda.docker.repository.cloudera.com/cdsw/engine:10

RUN /opt/conda/envs/python3.6/bin/pip  install tensorflow-gpu==1.12.0
```

- 03_02_make.sh

```ba
SITE_DOMAIN=$1

echo "SITE_DOMAIN=$SITE_DOMAIN"

if [ "$SITE_DOMAIN"  == "" ]; then

   echo "example : $0  goodmit.com "
   exit 1

fi

docker build --network=host -t tensorflow1.12.${SITE_DOMAIN}/cdsw/engine:10 . -f  03.02.tensorflow1.12.Dockerfile
```



#### Pytouch 이미지 빌드

- 03.03.pytorch1.3.Dockerfile

```bash
FROM  conda.docker.repository.cloudera.com/cdsw/engine:10

RUN conda install -n python3.6  -c anaconda pytorch-gpu=1.3.1 torchvision 
```

- 03_03_make.sh

```bash
SITE_DOMAIN=$1

echo "SITE_DOMAIN=$SITE_DOMAIN"

if [ "$SITE_DOMAIN"  == "" ]; then

   echo "example : $0  goodmit.com "
   exit 1

fi

docker build --network=host -t pytorch1.3.${SITE_DOMAIN}/cdsw/engine:10 . -f  03.03.pytorch1.3.Dockerfile
```



#### All Build & Save File

- all_sitedomain.sh : 모든 파일을 한번에 build 할 때 사용

```bash
SITE_DOMAIN=goodmit.com

 sh 01_make.sh  && \
 sh 02_make.sh  && \
 sh 03_01_make.sh ${SITE_DOMAIN}  && \
 sh 03_02_make.sh ${SITE_DOMAIN}  && \
 sh 03_03_make.sh ${SITE_DOMAIN}  && \
 sh save_docker.sh ${SITE_DOMAIN}
```

- save_docker.sh

```bash
SITE_DOMAIN=$1

echo "SITE_DOMAIN=$SITE_DOMAIN"

if [ "$SITE_DOMAIN"  == "" ]; then

   echo "example : $0  goodmit.com "
   exit 1

fi


docker save  cuda10.docker.repository.cloudera.com/cdsw/engine:10  | gzip > cuda10.docker.repository.cloudera.com.tar.gz 

docker save  conda.docker.repository.cloudera.com/cdsw/engine:10  | gzip > conda.docker.repository.cloudera.com.tar.gz 

docker save  tensorflow2.0.${SITE_DOMAIN}/cdsw/engine:10  | gzip > tensorflow2.0.${SITE_DOMAIN}.tar.gz 

#docker save  tensorflow1.12.${SITE_DOMAIN}/cdsw/engine:10  | gzip > #tensorflow1.12.${SITE_DOMAIN}.tar.gz 

#docker save  pytorch1.3.${SITE_DOMAIN}/cdsw/engine:10  | gzip >  #pytorch1.3.${SITE_DOMAIN}.tar.gz

```



- cuda image 빌드 후 결과

```bash
#build 성공 
Successfully built eba08ddfe70c
Successfully tagged cuda10.docker.repository.cloudera.com/cdsw/engine:10
```



#### Custom Image를 CDSW에 적용

- docker 이미지  load

```bash
#docker => cdsw server로 파일 전송 후

#cdsw 설치 서버에서
docker load --input cuda10.docker.repository.cloudera.com.tar.gz
```

- CDSW Engine에 이미지 추가
  - CDSW 접속 > admin login > Admin 메뉴클릭 > Engine Tab 클릭 > Engines Images에서
    - Description : 이미지 alias 입력
    - Repository:Tag : 이미지 생성 시 이름  입력
    - Add 버튼 클릭



## Trouble shooting

### conntrack-tools.x86_64 lib install ERROR 로 설치 진행 안될 경우

- 1.7.2에서 발생하며, OS 배포 버전 rpm package에는 포함되지 않아 별도로 다운 받아서 설치 해 줘야함.

- 해결방법

  - conntrack-tools download 

    ```bash
    wget http://mirror.centos.org/centos/7/os/x86_64/Packages/conntrack-tools-1.4.4-7.el7.x86_64.rpm
    wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libnetfilter_cttimeout-1.0.0-7.el7.x86_64.rpm
    wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libnetfilter_cthelper-1.0.0-11.el7.x86_64.rpm
    wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libnetfilter_queue-1.0.2-2.el7_2.x86_64.rpm
    ```

    

  - RPM-GPG-KEY-EPEL-7 Key 다운로드 : /etc/pki/rpm-gpg/ 경로에 RPM-GPG-KEY-EPEL-7 파일을 다운로드 한다. 

    ```bas
    cd /etc/pki/rpm-gpg/
    wget https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7
    ```

### cdsw status 시 Fail to connect to Kubernetes Master api. 오류 메시지 시

- 쿠버네티스는 광범위한 iptables 규칙을 사용하는데 기존 규칙이 쿠버네티스에 어떤 영향을 미칠지 파악하기 힘드므로 다음의 커맨드를 사용하여 iptables 규칙을 초기화한다.

```bash
[root@cdsw02 rpm-gpg]# cdsw status
Sending detailed logs to [/tmp/cdsw_status_14Tmlf.log] ...
CDSW Version: [1.7.2.2066404:fc261c3]
Installed into namespace 'default'
OK: Application running as root check
OK: NFS service check
Processes are not running: [kubelet]. Check the CDSW role logs in Cloudera Manager for more details.
OK: Sysctl params check
OK: Kernel memory slabs check
Failed to connect to Kubernetes Master api.

## 해결방법
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -F
```

### Error, CM server guid updated

- cm_guid파일에 등록된 ID와 CM server가 보내주는 ID가 다르게 설정되어 있어서 발생하는 문제이므로 cm_guid파일을 삭제후 agent를 재시작한다

```ba
rm /var/lib/cloudera-scm-agent/cm_guid
service cloudera-scm-agent restart
```



#### cm > 서비스 추가 > docker daemon 설치중 docker-thinpool 관련 오류 발생 시

- Block device를  fdisk로  파티션 만들어 주면 설치됨
  - Ex) /dev/sdb로 full로 잡혀져 있는데, 설치 안되는 경우
    - fdisk /dev/sdb에 파시션 new partition 생성 
    - 생성된 /dev/sdb1(첫번째 파티션일 경우)를 block device로 하여 다시 설치 진행 하여 처리.

#### lvm 초기화

- block device가 clear 하지 않은 경우 docker 설치 시 오류가 날 수 있어 초기화가 필요함.

```bash
lvdisplay
vgdisplay

lvremove /dev/VG이름/LV이름
vgremove /dev/VG이름
pvremove /dev/sdb1 /dev/sdb2 /dev/sdc1
```
