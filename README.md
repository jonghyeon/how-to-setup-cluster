# how-to-setup-cluster

# 그림1

* 클러스터 환경이 위 그림과 같다고 가정하고 클러스터 세팅을 설명하겠습니다.(Slave의 대수에 따라 맞게 해주시면 되겠습니다.)
* Master, Slave 모든 서버에서 실행할 명령어는 (모든 서버에서) 수행하시면 됩니다.
* Master에서만 실행할 명령어는 (Master에서만 수행) 하시면 됩니다
* Slave에서만 실행할 명령어는 (Slave에서만 수행) 하시면 됩니다


## 서버 환경 구성

### 가. user/usergroup 생성(모든 서버에서 수행)

모든 서버에 hadoop 유저 그룹과 hduser 유저 생성

```{.bash}
sudo addgroup hadoop
sudo adduser --ingroup hadoop hduser
```

sudoers file에 hduser 추가(hduser에게 권한 부여)

```{.bash}
sudo visudo
-------------------------
hduser ALL=(ALL:ALL) ALL
```

hduser로 로그인
```{.bash}
su - hduser
```

### 나. hosts 파일 수정(모든 서버에서 수행)
분산 환경이기 때문에 4대의 서버 모두에 접속이 자유롭게 되어야 합니다. 이를 위해서 모든 서버들의 hosts 파일에 ip와 host 이름을 생성해 주도록 합니다. 참고로 여기서는 ubuntu0이 master 서버가 될 예정이고, 그외 서버들은 slave 서버가 될 예정입니다.

hosts 파일 수정
```{.bash}
$ sudo vi /etc/hosts
--------------------------------------------------------------------------
192.168.1.10    ubuntu0
192.168.1.11    ubuntu1
192.168.1.12    ubuntu2
192.168.1.13    ubuntu3
```

잘 변경 되었는지 확인
```
cat /etc/hosts
```

### 다. hostname 변경(모든 서버에서 수행)
각 4대 서버들의 이름을 나. 에서 지정한 host이름과 동일하게 맞춰주도록 합니다.

ubuntu0 서버의 경우
```
sudo vi /etc/hostname
--------------------------------------------------------------------------
ubuntu0
```

변경사항 적용하기(서버 재부팅 필요없음)
```
sudo /bin/hostname -F /etc/hostname
```

#### 라. ssh 설정(Master에서만 수행)
hadoop은 분산처리시에 서버들간에 ssh 통신을 자동적으로 수행하게 됩니다. 이를 위해 암호입력 없이 접속이 가능하도록 ubuntu0 서버에서 공개키를 생성하고 생성된 키를 각 서버들에 배포해줍니다.

ssh 설치
```
sudo apt-get install openssh-server
```

ssh 설정 변경(sshd_config 파일을 열고 아래 두 줄을 찾아서 수정)
```
sudo vi /etc/ssh/sshd_config
--------------------------------------------------------------------------
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys
```

공개키 생성
```
mkdir ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t rsa -P ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

공개키를 각 서버에 배포
```
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu1
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu2
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu3
```

접속 테스트
```
ssh ubuntu3
```

# 그림2

#### 마. JAVA 설치(모든 서버에서 수행)
Hadoop이 JVM을 사용하기 때문에 모든 서버에 java SDK를 설치하도록 합니다. 여기서는 현재 최신 버젼인 java 1.8을 설치하겠습니다.

java directory 생성
``` 
sudo mkdir /opt/jdkcd /opt
```

아래 링크에서 운영체제에 맞는 Java SE Development Kit (e.g. 8u144) 다운로드 <http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html>

Java가 다운로드 된 경로로 가서 아래 명령어 수행
```
sudo tar -zxf jdk-8u144-linux-x64.tar.gz -C /opt/jdk
```

Update Ubuntu alternatives symbolic links
```
sudo update-alternatives --install /usr/bin/java java /opt/jdk/jdk1.8.0_144/bin/java 100
sudo update-alternatives --install /usr/bin/javac javac /opt/jdk/jdk1.8.0_144/bin/javac 100
```

# 그림 3 alternatives 설명

설치 확인(버젼정보가 보여지면 정상적으로 설치가 잘 된 것입니다.)
```
java –version
```

# 그림 4 자바 버전 확인

## Hadoop 설치

#### 가. 디렉토리 생성

data 디렉토리 생성(모든 서버에서 수행)
```
sudo mkdir /data
sudo chown -R hduser:hadoop /data
```

hadoop, spark가 위치할 디렉토리 생성(모든 서버에서 수행)
```
cd ~
mkdir apps
```

namenode 디렉토리 생성(Master에서만 수행)
```
mkdir /data/tmp /data/namenode
```

datanode, userlogs 디렉토리 생성(Slave에서만 수행)
```
mkdir /data/tmp /data/datanode /data/userlogs
```

#### 나. hadoop 2.7.4 설치(Master에서만 수행)

아래 링크에서 hadoop 다운로드
<http://www.us.apache.org/dist/hadoop/common/hadoop-2.7.4/hadoop-2.7.4.tar.gz>

hadoop 파일이 존재하는 경로에서 아래 명령 수행

```
tar –zxf hadoop-2.7.4.tar.gz –C ~/apps
cd ~/apps
ln –s hadoop-2.7.4 hadoop
```

환경변수 설정: hadoop 디렉토리에 쉽게 접근할 수 있도록 환경변수를 설정해 줍니다.
```
vi~/.bashrc
--------------------------------------------------------------------------
# java
export JAVA_HOME=/opt/jdk/jdk1.8.0_144
export PATH=${JAVA_HOME}/bin:$PATH

# hadoop
export HADOOP_HOME=/home/hduser/apps/hadoop
export PATH=${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:$PATH
export HADOOP_CONF_DIR=${HADOOP_HOME}"/etc/hadoop"
```
hadoop 환경 설정: 해당 파일을 열고 JAVA_HOME 환경변수 선언 부분 수정
```
$ vi ~/apps/hadoop/etc/hadoop-env.sh
--------------------------------------------------------------------------
export JAVA_HOME=/opt/jdk/jdk1.8.0_144
```

#### 다. config 파일 설정(Mater에서만 수행)

hadoop과 관련된 config 파일 내용을 작성합니다.
```
vi ~/apps/hadoop/etc/hadoop
```

core-site.xml
```
vi core-site.xml
--------------------------------------------------------------------------
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://ubuntu0:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/tmp</value>
    </property>
</configuration>
```


# Reference & Acknowledgements
http://hadoop.apache.org/

http://daeson.tistory.com/277
