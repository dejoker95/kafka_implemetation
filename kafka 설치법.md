# Kafka 설치법

## 순서

1. java1.8 이상 설치
2. 주키퍼 설치 및 설정
3. 카프카 설치 및 설정



## 환경

- Ubuntu 20.04 LTS
- t2.large (2cpu 8mem)



## 1. 자바 설치

1.8 이상의 버전 추천

```bash
sudo apt-get update
sudo apt install openjdk-8-jdk
```



## 2. 주키퍼 설치 및 설정

올바르게 설정해도 실행이 안될 때 파일명에 bin이 들어간 것을 다운 받으니 문제 없이 실행됨

1. 주키퍼 바이너리 파일 다운로드 및 압축해제

```bash
wget https://mirror.navercorp.com/apache/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz

tar xvf apache-zookeeper-3.6.3-bin.tar.gz

# 심볼릭 링크 만들기
ln -s apache-zookeeper-3.6.3-bin zookeeper
```

압축해제한 파일의 경로를 앞으로 {ZK_HOME}으로 명시

2. zoo.cfg 파일 만들기

```
vi {ZK_HOME}/conf/zoo.cfg
```

```bash
# zoo.cfg 내용
tickTime=2000
dataDir={원하는 경로}	#/data 라는 경로 만들어 줬음
clientPort=2181
initLimit=5
syncLimit=2

# 클러스터를 만들 경우 아래 내용 추가
server.1={브로커1 주소}:2888:3888
server.2={브로커2 주소}:2888:3888
server.2={브로커2 주소}:2888:3888
```

클러스터를 위해 서버 정보를 줄 경우 각 서버에 myid라는 파일을 만들어줘야함

```bash
mkdir /data
vi /data/myid
```

```bash
# server.myid 형식임
# server.1의 myid 파일 내용
1
```

3. 주키퍼 실행

```bash
{ZK_HOME}/bin/zkServer.sh start
```

4. 팁

브로커 주소를 고정 ip를 사용한다면 /etc/hosts 파일에 이름과 주소를 설정해놓는 것이 편함

```bash
vi /etc/hosts

# hosts 파일 내용
{broker1 주소} {broker1 이름}
{broker2 주소} {broker2 이름}
{broker3 주소} {broker3 이름}
```

5. 편리한 실행을 위한 systemd 등록

```bash
vi /etc/systemd/system/zookeeper-server.service

# zookeeper-server.service 내용
[Unit]
Description=zookeeper-server
After=network.target

[Service]
Type=forking
User=root
Group=root
SyslogIdentifier=zookeeper-server
WorkingDirectory={ZK_HOME}
Restart=always
RestartSec=0s
ExecStart={ZK_HOME}/bin/zkServer.sh start
ExecStop={ZK_HOME}/bin/zkServer.sh stop

# systemd 재시작
systemctl daemon-reload

# 실행 명령어
systemctl start zookeeper-server.service

# 상태 확인
systemctl status zookeeper-server.service

```



## 3. Kafka 설치 및 설정

1. 카프카 파일 다운로드 및 압축해제

```bash
wget https://archive.apache.org/dist/kafka/2.6.0/kafka_2.13-2.6.0.tgz

tar xvf kafka_2.13-2.6.0.tgz

ln -s kafka_2.13-2.6.0 kafka
```

압축해제한 카프카 경로를 앞으로 {KAFKA_HOME}이라 명시

2. server.properties 수정

```bash
vi {KAFKA_HOME}/config/server.properties

# 파일에서 해당 부분만 수정

broker.id=0	#브로커 마다 unique하게 주면 됨

log.dirs=/data1 # 로그 데이터 저장 경로, 디스크가 여러개면 더 추가해도 됨

# zookeeper.connect
# standalone의 경우
zookeeper.connect=localhost:2181
# 클러스터의 경우
zookeeper.connect={브로커1 주소}:2181,{브로커2 주소}:2181,{브로커3 주소}:2181/{지노드 이름}
ex) zookeeper.connect=kafka1:2181,kafka2:2181,kafka3:2181/mykafka
```

3. 카프카 실행

```bash
{KAFKA_HOME}/bin/kafka-server-start.sh {KAFKA_HOME}/config/server.properties -daemon
```

4. 편리한 실행을 위한 systemd 등록

```bash
vi /etc/systemd/system/kafka-server.service

# kafka-server.service 내용
[Unit]
Description=kafka-server
After=network.target

[Service]
Type=simple
User=root
Group=root
SyslogIdentifier=kafka-server
WorkingDirectory={KAFKA_HOME}
Restart=no
RestartSec=0s
ExecStart={KAFKA_HOME}/bin/kafka-server-start.sh {KAFKA_HOME}/config/server.properties
ExecStop={KAFKA_HOME}/bin/kafka-server-stop.sh

# systemd 재시작
systemctl daemon-reload

# 실행 명령어
systemctl start kafka-server.service

# 상태 확인
systemctl status kafka-server.service
```



5. CLI로 프로듀서, 컨슈머 실행

```bash
# 토픽 만들기
{KAFKA_HOME}/bin/kafka-topics.sh \
--zookeeper kafka1:2181,kafka2:2181,kafka3:2181/mykafka	\
--replication-factor 1 --partitions 1 --topic mytopic --create

# 프로듀서 실행
{KAFKA_HOME}/bin/kafka-console-producer.sh \
--broker-list kafka1:9092,kafka2:9092,kafka3:9092 \
--topic mytopic

# 컨슈머 실행
{KAFKA_HOME}/bin/kafka-console-consumer.sh \
--bootstrap-server kafka1:9092,kafka2:9092,kafka3:9092 \
--topic mytopic --from-beginning
```

