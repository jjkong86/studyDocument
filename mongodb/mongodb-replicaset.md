Replica Set(복제)
=================

복제는 여러 서버 상에서 데이터의 동일한 복사본을 유지하는 방법 _(모든 실제 서비스에서는 복제를 적용할 것을 권장)_  
→ 한 대 또는 그 이상의 서버에 이상이 발생하였을 때 애플리케이션의 정상 동작 및 데이터를 안전하게 보존할 수 있음  
※ MongoDB는 **복제 셋(replica set)** 을 생성함으로써 복제를 설정할 수 있음


복제 셋(replica set)의 구성
---------------------------

- **1대의 프라이머리(primary) 서버**
      - 클라이언트의 요청을 처리
      - 데이터에 대한 **쓰기**, 읽기 가능
      - 복제 셋에서 쓰기 동작을 하는 유일한 멤버
- **여러 대의 세컨더리(secondary) 서버**
      - 프라이머리 데이터의 복제 데이터를 가짐
      - 프라이머리의 oplog에서 하던 동작을 세컨더리의 데이터 셋에 **비동기적**으로 적용
      - 프라이머리 서버에 장애가 발생시 세컨더리 서버는 자신들 중 새로운 프라이머리 서버를 선출할 수 있음
      - 데이터의 읽기만 가능

※ 복제 셋의 노드는 최대 **12**개까지 구성할 수 있다.

복제의 작동 방식
----------------

### 오피로그(oplog)

데이터 복제를 가능하게 함

- 캡드(capped) 컬렉션으로 모든 복제 노드에서 local 이라는 데이터베이스 내에 존재
- 데이터에 대한 모든 수정사항을 기록
      1. 클라이언트가 프라이머리 노드에 대해 쓰기를 할 때마다 세컨더리에서 재생하기 위한 충분한 정보가
         프라이머리 노드의 오피로그에 자동으로 추가됨
      1. 쓰기가 세컨더리 노드에 복제되고 나면 쓰기 정보가 세컨더리 노드의 오피로그에도 기록됨
- 오피로그 항목은 BSON 타임스탬프로 인식되고, 모든 세컨더리는 타임스탬프를 이용해서 적용할 최신 항목을 추적
- local 데이터베이스의 `oplog.rs`라는 컬렉션에 오피로그가 저장됨
  > ※ local 데이터베이스에 저장된 기타 컬렉션
  >   - `replset.minvalid` - 복제 셋 멤버의 초기 동기화를 위한 정보
  >   - `system.replset` - 복제 셋 설정 도큐먼트를 저장
  >   - `me`, `slaves` - 쓰기 concern을 구현하는데 사용
  >   - `system.indexes` - 인덱스 규격에 대한 정보를 가지고 있는 표준 컬렉션

- 세컨더리가 복제 데이터를 유지하는 프로세스
      1. 프라이머리 노드에 쓰기 기록
      1. 프라이머리 노드의 오피로그에 추가
      1. 세컨더리 노드가 자신의 오피로그에 프라이머리 노드의 오피로그를 복제
- 세컨더리에서의 상세 프로세스
      1. 세컨더리 노드가 업데이트 준비가 됨
      1. 자신의 오프로그에서 가장 최근 항목의 타임스탬프를 검사
      1. 프라이머리 노드의 오피로그에서 최근 항목의 타임스탬프 이후의 모든 오피로그 항목을 질의
      1. 질의된 오피로그 항목을 자신의 오프로그에 추가
      1. 추가된 오프로그의 항목을 자신의 데이터에 적용

- 세컨더리는 롱 폴링(long polling) 방식을 사용하여 프라이머리의 변경 데이터를 즉각적으로 적용하게 됨

### 복제 중지

세컨더리가 프라이머리 노드의 오피로그에서 동기화할 시점을 발견하지 못하면 복제는 완전히 중지

※ 발생하는 예외 메시지
```
repl: replication data to stable, halting
Fri Jan 28 14:19:27 [replsecondary] caught SyncException
```

- 오피로그는 캡드 컬렉션이므로 컬렉션 항목이 시간이 지나면 삭제될 수 있음
- 세컨더리 노드가 어떠한 사정으로 프라이머리 노드의 오피로그로 부터 동기화 지점을 찾지 못하면  
  → 세컨더리가 프라이머리의 완벽한 복제본임을 확신할 수 없으므로 복제를 중지할 수 밖에 없음
- 유일한 해결책: 프라이머리의 데이터를 처음부터 다시 동기화하는 것
- 예방할 수 있는 방법
      - 세컨더리 노드에서 업데이트가 지체되는 상황을 모니터링
      - 쓰기 연산의 규모에 상응하는 충분한 크기의 오피로그를 가지고 있어야 함

### 오피로그 크기 산정

* 오피로그의 디폴트 크기 _(※ 일반적인 경우에는 디폴트 크기로 충분함)_
      - _3.0 이전_
            - 32bit 시스템: 50MB
            - 64bit 시스템: 1GB 또는 사용되지 않는 디스크 공간의 5%
            - OS X: 192MB (개발을 위한 시스템으로 고려)
      - **3.0 이후**
            - Unix and Windows system
                  - In-Memory Storage Engine: 물리 메모리의 5%
                  - WiredTiger Storage Engine(v3.0 이후로 사용, v3.2 부터는 기본 엔진): 사용되지 않은 디스크 공간의 5%
                  - MMAPv1 Storage Engine(v3.2 이전의 원래 스토리지 엔진): 사용되지 않은 디스크 공간의 5%
            - OS X(64bit)
                  - In-Memory Storage Engine: 물리 메모리 192MB
                  - WiredTiger Storage Engine: 디스크 공간 192MB
                  - MMAPv1 Storage Engine: 디스크 공간 192MB

* 쓰기가 많이 발생하는 애플리케이션에 대한 오피로그의 결정을 위해서는 경험적 테스트가 필요함
      - 복제를 구성하고 프라이머리에서 실제 시스템에서 발생하는 비율로 한 시간 이상 프라이머리에 쓰기를 수행해보고
        복제셋에 생성된 현새 상태를 얻어 최소한 8시간은 견딜 수 있는 크기를 추론하여 설정하는 것이 좋음

### 하트비트(heartbeat)와 장애복구

하트비트(heartbeat)의 역할은 시스템의 건강상태를 모니터링를 통해 장애 복구를 가능하게 하는 것

**MongoDB가 건강상태를 확인하는 방법**
- 복제 셋의 각 멤버는 디폴트로 다른 멤버들을 매 2초마다 한 번씩 핑(ping)을 해봄 → 시스템의 건강 상태를 확인할 수 있음
- 상태 명령을 수행했을 때 알 수 있는 정보
      - 상태에 대한 정보(1: 건강, 0: 응답 없음)
      - 각 노드의 마지막 하트비트의 타임스탬프
- 모든 모드가 건강한 상태일 때 만 복제 셋이 정상 동작하는 것이며 한 노드라도 반응이 없으면 조치를 취해야 함

**장애 발생시 처리 방식**
- 세컨더리 노드가 다운
      - 세컨더리의 과반수가 살아있는 경우  
        → 복제 셋은 상태를 변경하지 않고 다운된 세컨더리의 복구를 기다림
      - 세컨더리의 과반수가 살아있지 않은 경우
            - 프라이머리가 세컨더리로 강등
            - WHY?  
              네트워크 장애로 하트비트가 실패한 경우(여전히 온라인 상태) 중재자와 세컨더리가 여전히 살아서 통신 → 프라이머리 선출  
              프라이머리가 2개인 상황이 발생 → 쓰기가 가능한 프라이머리가 2개인 상황이라 데이터의 일관성에 문제 발생
- 프라이머리 노드가 다운
      - 세컨더리의 과반수가 살아있는 경우
            - 세컨더리가 프라이머리로 승격
            - 하나 이상의 세컨더리가 있을 경우에는 가장 최근의 세컨더리가 프라이머리로 승격
      - 세컨더리의 과반수가 살아있지 않은 경우 → _System Crush_

### 커밋과 롤백

> 복제 셋에서 프라이머리의 쓰기가 과반수의 노드에 복제되기 전까지는 커밋되지 않은 것으로 여김

**롤백**
> 과반수의 노드에 복제가 되기 전에 세컨더리가 프라이머리로 승격 → 새로운 프라이머리에 쓰기 작업을 수행  
> → 기존 프라이머리 복구 → 새로운 프라이머리로 부터 복제하는 경우
>
> 예전 프라이머리에는 새로운 프라이머리 노드에는 존재하지 않는 오피로그가 존재할 수 있음 → 롤백 발생

※ **과반수의 노드에 복제된 적이 없는 쓰기는 모두 취소**

롤백으로 인해 취소된 데이터는 데이터 경로의 `rollback` 이라는 서브디렉토리에 저장됨  
롤백된 데이터를 복구할 필요가 있을 때는 `bsondump` 유틸리티로 BSON 파일 검사하고 `mongorestore`를 사용해서 복구할 수 있음

**롤백 상황을 방지하는 방법**: 쓰기 concern을 사용해서 데이터가 각 쓰기에 대해 과반수의 노드에 복제되는 것이 확실하게 할 수 있음

### 관리

> **로컬에서 복제 셋을 만들기 위한 준비**
> ```
> $ mongod --replset [name] --dbpath /data/node1 --port 40000
> $ mongod --replset [name] --dbpath /data/node2 --port 40001
> $ mongod --replset [name] --dbpath /data/arbiter --port 40002
> ```
>
> **복제 셋 생성**
>  ```
>  > rs.initiate()
>  > rs.add("localhost:40001")
>  > rs.add("localhost:40002", {arbiterOnly: true})
>  > db.isMaster()
>  > rs.status()
>  ```
> `arbiterOnly`: 세컨더리가 짝수일 때 투표에서 중재자 역할을 수행하여 프라이머리 선출에 투표를 함
> `db.isMaster()`: 현재 복제 셋의 상태에 대한 간략한 요약을 표시
> `db.status()`: 복제 셋 시스템에 대한 자세한 상태 정보를 표시
>
> **오피로그에 대한 기본 정보 확인**
>  ```
>  > db.getReplicationInfo()
>  ```
>
> **오피로그의 크기를 설정** (크기는 MB 단위)
>  ```mongodb
>  $ mongod --replSet myapp --oplogSize 1024
>  ```

**복제 셋의 생성을 설정 도큐먼트를 이용해서 수행**
```
> config = {_id: "myapp", members:[]}
> config.members.push({_id: 1, host: "localhost:40001"})
> config.members.push({_id: 0, host: "localhost:40000"})
> config.members.push({_id: 2, host: "localhost:40002", arbiterOnly: true})
> config
{
  "_id": "myapp",
  "members": [
    {
      "_id": 0,
      "host": "localhost:40000"
    },
    {
      "_id": 1,
      "host": "localhost:40001"
    },
    {
      "_id": 2,
      "host": "localhost:40002",
      "arbiterOnly": true
    }
  ]
}
> rs.initiate(config)
```

#### 설정 도큐먼트 옵션
* 복제 셋 개별 멤버에 대한 설정 옵션
      * `_id`(필수): 멤버의 아읻를 나타내는 고유한 정수 값 (0부터 시작해서 1씩 증가)
      * `host`(필수): 호스트 이름과 포트
      * `arbiterOnly`:  중재자인지 여부를 설정  
        **Arbitor**: 프라이머리 선정에만 참여하고 복제하지 않는 경량 멤버
      * `priority`: 장애 발생시 프라이머리로 선출될 가능성(0~1000), 0으로 설정할 경우 프라이머리로 선정되지 않음
      * `votes`: 프라이머리 선출시의 표 수를 지정(모든 복제 멤버들은 디폴드로 한 표를 받음)
      * `hidden`: `isMaster()`에 의해 생성되는 응답에서 숨김 → 애플리케이션(드라이버)에서 접속하는 것을 막을 수 있음
      * `buildIndexes`: 인덱스를 구축할 지 여부를 설정(기본 값: `true`)  
        프라이머리가 되지 않을 멤버에 대해서만 설정해야 함(`priority: 0`)
      * `slaveDelay`: 세컨더리가 프라이머리 변경을 적용하는데 걸리는 지연 시간을 초 단위로 설정  
        프라이머리가 되지 않을 멤버에 대해서만 설정해야 함(`priority: 0`)  
        데이터베이스의 의도되지 않은 문제 발생(예: 실수에 의한 프라이머리 삭제)에 대응(복구)하기 위해 활용할 수 있음
      * `tags`: 임의의 키-값 쌍인 도큐먼트, 데이터센터나 랙에서의 위치등을 설명하기 위해 사용할 수 있음  
        쓰기 concern과 읽기 세팅을 지정하는데 사용될 수 있음
* `settings`(글로벌 복제 셋 설정)의 옵션
      *  `getLastErrorDefaults`: 클라이언트가 파라미터 없이 `getLastError`를 호출할 때 사용되는 기본 매개변수 설정
      * `getLastErrorModes`: `getLastError` 명령에 대한 추가적인 모드를 정의

### 복제 셋 상태

```
> rs.status()
```

상태|상태 문자열|설명
:---:|:---------:|:-----------------------------------------------------:
0 | STARTUP | 복제 셋이 모든 복제 셋 멤버를 핑하고 설정 데이터를 공유함으로써 다른 노드와 교섭 중
1 | PRIMARY | 프라이머리 노드
2 | SECONDARY | 세컨더리 읽기 전용 노드
3 | RECOVERING | 읽기/쓰기를 진해할 수 없음, 장애 복구 후 또는 노드 추가 시 볼수 있음
4 | FATAL | 네트워크가 연결되었으나 노드가 반응하지 않음, 해당 노드의 호스트 서버에 오류가 있을 경우
5 | STARTUP2 | 초기 데이터 파일 동기화가 진행 중
6 | UNKNOWN | 네트워크 연결이 이루어져야 함
7 | ARBITER | 노드가 중재자 임
8 | DOWN | 이 노드가 액세스 가능하고 어느 시점에서는 안정적이지만 현재 하트비트 핑에 응답하지 않음
9 | ROLLBACK | 롤백이 수행되고 있음

### 장애조치와 복구

> 장애 발생 시 새로운 프라이머리를 선출하기 위해서는 과반수 이상이 필요하고,  
> 프라이머리는 자신이 과반수 이상에 머물러 있는 한 프라이머리로서의 역할을 유지할 수 있음  
> 과반수 이상이 복제되면 쓰기가 안전해짐
>
> **과반수**: 복제 셋의 모든 멤버의 절반보다 많은 것

**복제 셋의 멤버 수에 따른 과반수의 예**

복제 셋의 멤버 수|복제 셋의 과반수
:---------------:|:--------------:
1 | 1
2 | 2
3 | 2
4 | 3
5 | 3
6 | 4
7 | 4

※ __중재자(arbitor)의 역할__   
프라이머리 1대와 세컨더리 1대로 이뤄진 복제 셋의 과반수는 2인데 프라이머리의 장애시 투표를 행사할 수 있는
서버는 1대 뿐이므로 프라이머리를 선출할 수 없다.  
그러므로, 데이터의 복제에는 참여하지 않지만 프라이머리 선출에만 참여하는 중재자가 필요하다.

#### 장애 모드

* __깨끗한 장애(clean failure)__: 주어진 노드의 데이터 파일이 아직 손상되지 않은 상태
      * 발생 케이스: 네트워크 장애, mongodb 프로세스의 종료
      * 복구 방식: 네트위크 복귀, 프로세스의 재시작
* __확실한 장애(categorical failure)__: 주어진 노드의 데이터 파일이 더 이상 존재하지 않거나 데이터 파일이 깨진 상태
      * 발생 케이스: mongod 프로세스가 저널링이 사용되지 않은 상태에서 불시에 셧다운된 경우, 하드 디스크가 깨진 경우
      * 복구 방식: 재동기화를 통한 데이터 파일을 완전 대치, 최근 백업에서 복구

#### `rs.reconfigure()`를 이용한 복제 셋 재설정

복구가 불가능한 노드(`localhost:30001`)를 새로운 노드(`localhost:40001`)를 만들어서 대체하는 경우
```
> use local
> config = db.system.replset.findOne()
{
  "_id": "myapp",
  "version": 1,
  "members": [
    {
      "_id": 0,
      "host": "localhost:30000"
    },
    {
      "_id": 1,
      "host": "localhost:30001"
    },
    {
      "_id": 2,
      "host": "localhost:30002",
      "arbiterOnly": true
    }
  ]
}
> config.members[1].host = "localhost:40001"
> config.reconfigure(config)
```
→ 복제 셋은 새 노드를 인식하고 새 노드는 기존 멤버로 부터 동기화를 시작함

#### 백업으로 부터 복구

* 백업에 의한 복구는 백업 내의 오피로그가 현재 복제셋의 오피로그와 비교해서 같은 경우에만 사용할 수 있음
      * 백업 오피로그의 최근 연산이 현재 복제 셋의 오피로그에 여전히 존재해야 함 → `db.getReplicationInfo()`가 제공하는 정보로 확인할 수 있음
      * 백업의 오피로그 항목이 백업이 복구되는 시점에서 오래 되었다면 재동기를 수행하는 것이 효율적일 수 있음
* 백업 데이터 파일을 mongod 데이터 경로로 복사 → 동기화는 자동 시작 → `rs.status()`로 확인 가능
      * 인덱스를 재구축할 필요가 없으므로 대부분의 경우 빠르게 복구가 가능하다

### 배포 전략

> 복제 셋의 최대 갯수: 12

#### 자동 장애 조치를 제공할 수 있는 최소 구성
* 2개의 복제 노드 + 1개의 중재자
* 중재자(arbiter)는 애플리케이션 서버에 존재 + 복제 노드는 개별 서버를 구성

#### 업타임이 중요한 서비스의 구성
3개의 완전한 복제 노드로 구성    
→ 한 노드가 완전히 기능 정지힌 경우에도 2개의 노드는 정상 동작하며 복구 중에도 자동 장애조치를 처리할 수 있음

#### 2개의 데이터 센터로 구성
두 데이터 센터 중 하나는 재해 복구 목적(세컨더리 데이터 센터)으로만 사용하는 경우
※ 복제 셋의 프라이머리는 항상 프라이머리 데이터 센터에 존재하도록 설정
* __프라이머리 데이터 센터__: 프라이머리(1) + 세컨더리(1)
* __세컨더리 데이터 센터__: 세컨더리(1: priority=0)
* 프라이머리 데이터 센터 내의 노드들 모두를 사용할 수 없을 경우
      * 세컨더리 데이터 센터의 세컨더리 노드를 임시로 사용: mongodb 서버의 셧다운 → `--replSet` 옵션 없이 재시작
      * 세컨더리 데이터 센터에 노드 2개를 생성 → 복제 셋의 재설정을 강제적으로 수행 (`force`옵션 사용)
        ```
        > rs.reconfigure(config, {force: true})
        ``` 

드라이버와 복제
---------------

복제 셋을 사용하는 어플리케이션 개발시 고려 주제
* 연결과 장애조치
* 쓰기 concern
* 읽기 확장

### 연결과 장애 조치

> 드라이버는 MongoDB에 접속하면 `isMaster` 명령을 수행

#### 단일 노드 연결
복제 셋의 마스터로 지정된 노드에 연결하는 것 (독립 MongoDB 노드로 접속하는 것과 동일)
※ 만일 세컨더리 노드에 접속하려면 세컨더리 노드에 접속하고 있음을 명기해야만 한다.

#### 복제 셋 연결
복제 셋을 하나의 전체로 보고 연결  
→ 드라이버는 어떤 노드가 프라이머리인지 파악하고 장애조치의 경우에 새로운 프라이머리가 된 노드에 재연결

### 쓰기 concern

> 개발자로 하여금 애플리케이션이 진행되기 전에 쓰기가 어느정도로 복재되어야 하는지를 지정하는 것

`getlasterror` 명령에 대해 `w`와 `wtimeout` 필드를 통해 조정됨
- `w`: 쓰기 연산을 복제할 서버의 수
      - 쓰기가 최소한 하나의 서버에 복제되어야 한다면 `w`를 `2`로 설정
      - `w` 값이 1보다 큰 쓰기 concern을 사용할 경우 추가적인 지연이 발생한다는 점을 명심해야 함
      - 저널링을 사용하는 경우 `w`를 `1`로 정하는 것이 대부분의 애플리케이션에서 효율적임
- `wtimeout`: 쓰기가 지정된 시간 내에 복제되지 못할 경우 에러를 리턴하도록 하는 타임아웃 시간 (단위: 밀리초)
  ※ `wtimeout`을 설정하지 않은 상태에서 복제가 발생하지 않으면 해당 연산은 무한 블록됨

### 읽기 스케일링

> 복제 셋은 쓰기는 프라이머리에만 가능하지만 읽기는 세컨더리에도 가능하므로 읽기 부하를 분산을 시킬 수 있다.
> 각 드라이버들은 쿼리를 하나 이상의 세컨더리 노드로 보내는 옵션을 제공하고 있다.

※ 자바에서 `slaveOk` 옵션을 `true`로 지정하면 스레드당 로드 밸런싱을 사용

* 읽기 스케일링을 자제 해야할 경우
      * 애플리케이션의 쓰기 부하가 많을 경우  
        쓰기 연산을 빈번하게 수행하는 세컨더리에 읽기 요청을 보내는 것은 원활한 복제를 가로막을 수 있음
      * 일관성이 요구되는 읽기를 해야할 경우  
        복제는 비동기적이므로 최근에 프라이머리에 수행된 쓰기가 반영하지 않았을 수도 있음

### 태깅

쓰기 concern 이나 읽기 확장을 사용하고 있을 경우 사용할 세컨더리 노드를 세밀하게 제어하기 위해 사용할 수 있다.

