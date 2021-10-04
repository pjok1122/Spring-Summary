# Kafka CLI

Kafka를 설치하고 띄우는 과정까지의 간단한 명령어만 살펴보고, 앞서 배운 용어들이 어떻게 사용되는지 살펴보는 것이 목적입니다. 따라서 본 문서로 실제 실습을 진행하기에는 내용이 빈약합니다. 만약 실습을 진행하고 싶다면, [Tacademy-kafka](https://github.com/AndersonChoi/tacademy-kafka/blob/master/%EC%95%84%ED%8C%8C%EC%B9%98%20%EC%B9%B4%ED%94%84%EC%B9%B4%20%EC%9E%85%EB%AC%B8%EA%B3%BC%20%ED%99%9C%EC%9A%A9%20%EA%B0%95%EC%9D%98%EC%9E%90%EB%A3%8C.pdf) 자료를 참고하시는 것을 권해드립니다.


<hr>

### OpenJDK 설치

```
$ sudo yum install -y java-1.8.0-openjdk-devel.x86_64
```

Kafka는 JVM 위에서 동작하기 때문에 버전에 맞는 JDK를 설치해주어야 합니다.

### Download Kafka

```
$ wget http://mirror.navercorp.com/apache/kafka/2.5.0/kafka_2.12-2.5.0.tgz
$ tar xvf kafka_2.12-2.5.0.tgz
```

### Kafka Heap 사이즈 설정

```
$ export KAFKA_HEAP_OPTS="-Xmx400m -Xms400m"
```

EC2 무료버전을 사용하기 위해 Heap 사이즈를 400MB로 낮춘 작업. 필요에 따라 조정.

```
-Xmx6g -Xms6g -XX:MetaspaceSize=96m -XX:+UseG1GC
-XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M
-XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80
```

링크드인에서 테스트한 Java option 추천 값이라고 합니다.

### config/server.properties

```
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://{aws ec2 public ip}:9092
```

- borker.id : 클러스터 내에서 브로커를 구분하기 위한 브로커 번호 (정수)
- listeners : Kafka 통신에 사용되는 호스트 정보
- advertised.listeners : Kafka Client(Procuder/Consumer)가 접속할 호스트 정보
- log.dirs : 메시지를 저장할 디스크 디렉토리. 세그먼트가 저장된다.
- log.segment.bytes : 세그먼트 파일의 크기
- log.retention.ms : 닫힌 세그먼트를 얼마나 오래 보존할 지 지정 (ms)
- zookeeper.connect : 브로커의 메타데이터를 저장하는 주키퍼의 위치
- auto.create.topics.enable : 자동으로 토픽을 생성할 것인지에 대한 여부
- num.partitions : 토픽 별 자동으로 생성되는 파티션의 개수
- message.max.bytes : Kafka 브로커에 쓸 수 있는 최대 메시지 크기

### 실행

```
$ bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
$ bin/kafka-server-start.sh -daemon config/server.properties
```

- 주키퍼는 카프카의 메타데이터를 저장한다. (브로커ID, Controller ID 등)


### Kafka Client 설치

```
$ curl https://archive.apache.org/dist/kafka/2.5.0/kafka_2.13-2.5.0.tgz --output kafka.tgz 
$ tar -xvf kafka.tgz
$ cd kafka_2.13-2.5.0/bin
```

- kafka-topics.sh : 토픽 생성, 조회, 수정 등을 수행하는 스크립트
- kafka-console-consumer.sh : 받은 메시지를 콘솔에 출력해주는 컨슈머 실행 스크립트
- kafka-console-producer.sh : 토픽에 메시지를 전달하는 프로듀서 실행 스크립트
- kafka-consumer-groups.sh : 컨슈머그룹 조회, 컨슈머 오프셋 확인 수정 스크립트

### CLI 예시 스크립트

```
$ ./kafka-topics.sh --create --bootstrap-server {aws ec2 public ip}:9092 --replication-factor 1 --partitions 3 --topic test
```

- topic이름 test로 복제본 1개, 파티션 3개 생성. (여기서 복제본은 Leader 파티션을 포함한 개수이므로 복제본이 없는 상태입니다.)
- --topic : 토픽 이름 설정
- --bootstrap-server : 토픽관련 명령어를 수행할 대상 카프카 클러스터
- --replication-factor : replica 개수 지정 (브로커 수보다 작아야 합니다.)
- --partitions : 파티션 개수 설정
- --config : 각종 토픽 설정 가능 (retention.ms, segment.byte)
- --create : 토픽 생성
- --delete : 토픽 제거
- --describe : 토픽 상세 확인
- --list : 카프카 클러스터의 토픽 리스트 확인
- --version : 대상 카프카 클러스터 버전 확인

<br>

```
$ ./kafka-console-producer.sh --bootstrap-server {aws ec2 public ip}:9092 --topic test
```

- topic이름 test에 메시지를 넣을 수 있는 콘솔 프로듀서 실행.

<br>

```
$ ./kafka-console-consumer.sh --bootstrap-server {aws ec2 public ip}:9092 --topic test --from-beginning
```

- topic이름 test에 있는 메시지를 컨슘하는 컨슈머를 생성. --from-beginning은 offset을 0번부터 읽음을 의미합니다. 
- segement가 기간이 다되어 사라진 경우, 남아있는 데이터의 처음부터 읽습니다.

<br>

```
$ ./kafka-console-consumer.sh --bootstrap-server {aws ec2 public ip}:9092 --topic test -group testgroup --from-beginning
```

- group을 testgroup으로 지정하여 메시지를 읽을 수 있습니다. 앞서 실행한 consumer와는 다른 그룹으로 관리됩니다.

<br>

```
$ ./kafka-consumer-groups.sh --bootstrap-server {aws ec2 public ip}:9092 --list
```

- 컨슈머 그룹 리스트 확인

<br>

```
$ ./kafka-consumer-groups.sh --bootstrap-server {aws ec2 public ip}:9092 --group testgroup --describe
```

- 각 컨슈머 그룹 별로 파티션의 어디까지 메시지를 읽었는지 확인할 수 있다.
- LAG : 현재 컨슈머 그룹이 읽은 오프셋과 파티션의 마지막 오프셋 간의 차이를 의미한다.

<br>

```
$ ./kafka-consumer-groups.sh --bootstrap-server {aws ec2 public ip}:9092 --group testgroup --topic test --reset-offsets --to-earliest --execute
$ ./kafka-consumer-groups.sh --bootstrap-server {aws ec2 public ip}:9092 --group testgroup --topic test:1 --reset-offsets --to-offset 10 --execute
```

- 토픽이나 파티션의 오프셋 위치를 변경할 수 있다.