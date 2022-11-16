# MongoDB Cluster

MongoDB cluster는 replica set이나 sharded set으로 구성할 수 있다. 일반적으로 높은 가용성을 보장하기 위해 mongodb는 replica set을 이용한다. replica set은 primary node에 write 요청이 왔을 때, 모든 변화를 operation log (oplog)라는 공간에 저장해둔다. oplog에 기록된 데이터는 비동기로 replica set으로 복제된다. mongoDB의 primary node가 죽었을 때는 secondary node(replica)에서 election을 통해 다음 primary node를 선발한다. 이런 방식으로 MongoDB는 HA(높은 가용성)을 보장한다.

하지만, 여전히 데이터의 일관성에 대해서는 의문이 남는다. secondary node에는 비동기로 동기화 하기 때문에, primary에서 조회한 결과와 secondary에서 조회한 결과가 다르게 나타날 수 있다는 의미이다. 이러한 문제는 replica set으로 구성된 mongoDB에 질의 할 때, primary만을 이용할 것인지, secondary를 이용할 것인지에 따라 다른 결과를 얻을 수 있다. 즉, 일관성에 대한 내용은 설정을 통해 조절을 할 수가 있다.
읽기,쓰기를 primary node로만 진행한다면 높은 일관성을 보장할 수 있다. 반면 모든 요청이 primary로 몰리기 때문에 분산 네트워크 관점에서는 좋지 않을 수 있다. 반대로 secondary preferred로 설정한다면, replica set에서 데이터를 읽기 때문에 최신 데이터가 조회/되지 않을 수 있다. 따라서 어플리케이션의 특성에 따라서 primary preferred인지 secondary preferred인지를 선택할 수 있다.

다만 MongoDB의 Transaction을 사용하는 경우에는 primaryPreferred나 secondaryPreferred도 사용할 수 없고 오직 primary만 사용해야 한다.

그 외의 옵션으로 write concern 같은 옵션도 존재한다. 이 옵션은 write operation의 결과가 모든 replica에 반영되어야 성공으로 볼 것인지에 대한 여부를 체크할 수 있다. performance가 떨어지는 대신에 높은 데이터 일관성과 가용성을 보장할 수 있다.

sharding cluster는 replica set과 같이 사용할 수 있다. sharding cluster의 특징으로는 데이터를 여러 노드에 분산 시켜서 저장하는 것이다. 여러 노드에 데이터를 나누어서 저장하기 가장 적합한 상황은 데이터의 개수가 많을 때이다. shard cluster는 `mongos`라는 쿼리 라우터가 존재하여, 어떤 서버에 요청을 해야 하는지를 해결해주는 router역할을 한다. 