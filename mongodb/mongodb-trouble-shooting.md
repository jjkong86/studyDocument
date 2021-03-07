# Mongodb Trouble Shooting

---

   운영 중인 서비스에서는 몽고디비와 레디스를 조합하여 사용하고 있다. 언제부턴가 특정 api에서 slow query가 발생하고 있었음.
   - 코드에서 explain data를 사용해서 분석해보니 엉뚱한 index를 타는 경우가 발생하고 있었음
   - 코드에서 힌트를 추가하여 명시적으로 index를 사용하도록 함 
   - 임시방편으로 힌트를 사용하여 문제를 해결했지만 근본적인 해결책은 아님
   - 문제는 compound index를 제대로 알지 못하고 사용하였기에 발생했던 것

   ```
   db.User.createIndex({age : 1, location : 1});
   db.User.find({age : {$gt : 20}, location : '서울'});
   ```

예를 들어 위와 같은 index와 find 쿼리를 보자. 얼핏 보기엔 index를 정상적으로 탈것만 같다.
결과는?

---

### 데이터 준비

   ```
   db.User.insertMany([
   {userId : NumberInt(200), age : NumberInt(20), location : '서울'},
   {userId : NumberInt(201), age : NumberInt(23), location : '서울'},
   {userId : NumberInt(202), age : NumberInt(22), location : '부산'},
   {userId : NumberInt(203), age : NumberInt(31), location : '서울'}
   ])
   ```
---

1. index 없이 조회

   ```
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'})
      .explain("executionStats");
   
   {
    "stage" : "COLLSCAN"
    "nReturned" : 2.0,  // result count
    "totalKeysExamined" : 0.0, // index scan count
    "totalDocsExamined" : 4.0, // document scan count
   }
    ```

   index를 생성하지 않았기 때문에 컬렉션 전체를 순회하는 COLLSCAN 스테이지가 사용되었다.

---

2. 단일 index 생성 후 조회

   ```
   db.User.createIndex({age : 1});
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'})
      .explain("executionStats");
   
   {
      "stage" : "IXSCAN",
      "indexName" : "age_1",
      "nReturned" : 2.0,
      "totalKeysExamined" : 3.0,
      "totalDocsExamined" : 3.0,
   }
      ```

예상대로 생성한 age_1 index를 활용하여 totalKeysExamined가 3으로 나왔다. 찾고자 하는 데이터는 nReturned 2로 나왔기 때문에 개선이 필요하다.
조금 더 최적화를 해보자.

---

3. compound index 생성

   ```
   db.User.createIndex({age : 1, location : 1});
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'})
      .explain("executionStats");

   {
      "stage" : "IXSCAN",
      "indexName" : "age_1_location_1",
      "nReturned" : 2.0,
      "totalKeysExamined" : 4.0,
      "totalDocsExamined" : 2.0,
   }
   ```

예상 시나리오는 age > 20인 조건은 2개뿐이기 때문에 totalKeysExamined은 2라고 예상 했지만, 결과는 4이다. 
모든 index를 탐색한것이다. 
이는 collection scan 보다 성능이 더 나쁘다. index를 탐색하고 collection을 조회하기 때문이다. 
 - 일반적으로 collection scan 대비 index scan 비율이 50%를 넘으면 index scan이 더 느리다. 
 - 메모리 효율을 생각한다면 최소 10%로는 넘지 않아야 한다. cardinality가 중요! 

---

### 문제점을 찾아보자.

**age_1_location_1 index 구조**는 아래와 같다.
   ```
      {20, '서울'} -> {userId : NumberInt(200), age : NumberInt(20), location : '서울'}
      {22, '부산'} -> {userId : NumberInt(202), age : NumberInt(22), location : '부산'}
      {23, '서울'} -> {userId : NumberInt(201), age : NumberInt(23), location : '서울'}
      {31, '인천'} -> {userId : NumberInt(203), age : NumberInt(31), location : '서울'}
   ```

query는 age 범위를 먼저 찾고 location 조건을 찾아야 하므로 index 탐색은 3이라고 예상했다.<br> 
결과로 추정해보자면 해당 index를 사용했지만 index prefix가 달라서 전체를 조회 했다고 추정 해볼 수 있다.  


4. 다시 compound index 생성

   ```
   db.User.createIndex({location : 1, age : 1});
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'})
      .explain("executionStats");
   
   {
      "stage" : "IXSCAN",
      "indexName" : "location_1_age_1",
      "nReturned" : 2.0,
      "totalKeysExamined" : 2.0,
      "totalDocsExamined" : 2.0,
   }
   ```

---

location_1_age_1 index 구조는 아래와 같다.

   ```
   {'부산', 22} -> {userId : NumberInt(202), age : NumberInt(22), location : '부산'}
   {'서울', 20} -> {userId : NumberInt(200), age : NumberInt(20), location : '서울'}
   {'서울', 23} -> {userId : NumberInt(201), age : NumberInt(23), location : '서울'}
   {'인천', 31} -> {userId : NumberInt(203), age : NumberInt(31), location : '서울'}
   ```

드디어 totalKeysExamined 2로 되어 원하는 결과가 나왔다. 
위의 결과로 index는 동등조건이 범위 조건보다 우선한다는 것을 알았다. 
논리적으로 생각을 해봐도 범위 조건을 먼저 다 찾고 나서 동등조건을 찾는 3번의 구조로는 최적화가 안 된다.

그렇다면 저의 경우 왜 문제가 발생했을까?

```
   db.User.createIndex({age : 1, location : 1});
   db.User.createIndex({location : 1});
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'})
      .explain("executionStats");

   {
      "stage" : "IXSCAN",
      "indexName" : "location_1",
      "nReturned" : 2.0,
      "totalKeysExamined" : 4.0,
      "totalDocsExamined" : 2.0,
   }
   ```

위와 같이 index가 생성되어 있을 때 **query planner**가 캐싱된 쿼리 플랜이 없다면  
- 최적의 index를 먼저 찾아 보고, 존재하지 않는다면 가능한 모든 쿼리 플랜을 조회 후 첫 batch(101)를 가장 좋은 성능으로 가져오는 플랜을 캐싱한다.
index가 최적화가 되어있지 않기 때문에 age_1_location_1 index가 채택되지 않는 경우가 발생한다. <br>
- 따라서 index를 location_1_age_1를 생성해야 했고, location_1_age_1 index는 삭제하고 prefix가 같은 location_1 또한 삭제 해야 한다.

---

### 주의할 점들

---

1. 너무 많은 index
   - index를 많이 만든다고 해서 빨라지지 않을 수 있다.
   - 메모리에 index가 차지하는 용량이 커져서 메모리와 디스크 사이에
     Frequent Swap이 많이 발생하게 되고 성능이 떨어짐
   - 데이터가 업데이트될 때마다 index 또한 업데이트 되어야 하기 때문에 write 성능이 감소 -> read 성능 감소로 이어짐
   - index prefix 같다면 정리 대상

   ```
   ex) {location : 1, age : 1}, {location : 1} -> location_1은 정리 대상
   ```

---

2. 멀티 소팅

   - 정렬 또한 index와 일치하게 해야 한다.
   
   ```
   db.User.createIndex({location : 1, age : 1});
   
   db.User.find({}).sort({location : 1, age : 1}); // covered
   db.User.find({}).sort({age : 1}); // not covered
   ```
   
---

2. 멀티 소팅 방향
   - single index는 고려하지 않아도 됨. 양방향 모두 지원하기 때문.
   - compound index의 경우 방향이 중요

   ```
   db.User.createIndex({location : 1, age : 1});
   
   - covered
      db.User.find({}).sort({location : 1, age : 1});
      db.User.find({}).sort({location : -1, age : -1});
      db.User.find({}).sort({location : -1});
   
   - not covered
      db.User.find({}).sort({location : -1, age : 1});
      db.User.find({}).sort({location : 1, age : -1});
      db.User.find({}).sort({age : 1});
   
   - 적절한 index 생성
      db.User.createIndex({location : 1, age : 1});
      db.User.createIndex({location : -1, age : 1});
      db.User.createIndex({age : 1, location : 1});
      db.User.createIndex({age : -1, location : 1});
   
      두 필드의 모든 방향을 커버할 수 있음.
   
   ex) db.User.find({}).sort({location : 1, age : -1});
    - {location : -1, age : 1} //역방향
   
   
   ```
3. Scan And Order

   ```
   db.User.createIndex({age : 1, location : 1});
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'}).sort({userId : 1})
      .explain("executionStats");
   
   {
       "stage" : "SORT",
      "indexName" : "location_1_age_1",
      "nReturned" : 2.0,
      "totalKeysExamined" : 2.0, 
      "totalDocsExamined" : 2.0, 
   }
   ```
 - 위와 같은 쿼리를 사용 하면 인덱스 탐색 후 결과를 메모리와 cpu 자원을 사용하여 정렬하게 됨
 - 메모리 정렬은 32MB 제한 하기 때문에 주의해야함

그렇다면 정렬을 활용하기 위한 인덱스 구성은?

```
   db.User.createIndex({location : 1, userId : 1, age : 1});
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'}).sort({userId : 1})
      .explain("executionStats");

   {
      "stage" : "IXSCAN",
      "indexName" : "location_1_userId_1_age_1",
      "nReturned" : 2.0,
      "totalKeysExamined" : 3.0, 
      "totalDocsExamined" : 2.0, 
   }
```
 - 위 쿼리는 totalKeysExamined 3이라서 효율적이지 않다고 생각할 수 있으나, IXSCAN을 했다는게 의미가 있음
 - query planner는 totalKeysExamined 값이 낮은 index를 채택하게 됨
 - 따라서 totalKeysExamined 값이 더 크더라도 메모리를 사용하지 않는 index를 채택하는 것이 효율적일 수 있음
 - hint 옵션을 활용하면 됨

---
출처

- https://tv.naver.com/v/11267386
- https://emptysqua.re/blog/optimizing-mongodb-compound-indexes/#comment-777924667
- https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=144346346