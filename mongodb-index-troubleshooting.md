
경험담
   - 서비스에서는 실제로 인덱스를 추가 했지만 타지 않는 경우가 발생하여 time out이 발생하고 있었음.
   - 코드에서 explain data를 저장해서 분석해보니 실제로도 인덱스를 타지 않는 경우가 발생해서 코드에서 힌트를 추가하여 명시적으로 인덱스를 사용하도록함 
   - 임시방편으로 힌트를 사용하여 문제를 해결했지만 근본적인 해결책은 아님
   - 문제는 복합 인덱스를 재대로 알지 못하고 사용하였기에 발생했던것

   ```
   db.User.createIndex({age : 1, location : 1});
   db.User.find({age : {$gt : 20}, location : '서울'});
   ```

예를 들어 위와 같은 인덱스와 find 쿼리를 보자. 얼핏 보기엔 인덱스를 정상적으로 탈것만 같다.
결과는?

데이터 준비

   ```
   db.User.insertMany([
   {userId : NumberInt(200), age : NumberInt(20), location : '서울'},
   {userId : NumberInt(201), age : NumberInt(23), location : '서울'},
   {userId : NumberInt(202), age : NumberInt(22), location : '부산'},
   {userId : NumberInt(203), age : NumberInt(31), location : '인천'}
   ])
   ```

1. index 없이 full sacn

   ```
   db.User.find({age : {$gte : 20, $lte : 25}, location : '서울'})
      .explain("executionStats");
   
   {
    "stage" : "COLLSCAN"
    "nReturned" : 2.0,  // result count
    "totalKeysExamined" : 0.0, // index scan count
    "totalDocsExamined" : 4.0, // document scan count(full scan)
   }
    ```

   index를 생성하지 않았기 때문에 document 풀스캔 하여 조회 했다.

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

예상대로 생성한 age_1 인덱스를 활용하여 totalKeysExamined가 3으로 나왔다. 
조금 더 최적화를 해보자.

3. 복합 index 생성

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

예상 시나리오는 age > 20인 조건은 2개뿐이기 때문에 index scan은 2일꺼라고 생각했지만 결과는 4이다. 
모든 인덱스를 탐색한것이다. 
이는 document full scan 보다 성능이 더 나쁘다. 인덱스를 탐색하고 document를 조회 하기 때문이다. 
 - 일반적으로 index document full scan 대비 index 탐색 비율이 50%를 넘으면 index scan이 더 느리다고 한다.

문제점을 찾아보자.
먼저 인덱스 구조를 보자.

   ```
   {
      {userId : NumberInt(200), age : NumberInt(20), location : '서울'},
      {userId : NumberInt(202), age : NumberInt(22), location : '부산'},
      {userId : NumberInt(201), age : NumberInt(23), location : '서울'},
      {userId : NumberInt(203), age : NumberInt(31), location : '인천'}
   }
   ```

age 범위를 먼저 찾고 location 조건을 찾아야 하기 때문에 3개를 탐색했다. 인덱스 탐색을 2로 낮출 수 있을까?


4. 다시 복합 index 생성

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

인덱스 구조

   ```
   {
      {userId : NumberInt(202), age : NumberInt(22), location : '부산'},
      {userId : NumberInt(200), age : NumberInt(20), location : '서울'},
      {userId : NumberInt(201), age : NumberInt(23), location : '서울'},
      {userId : NumberInt(203), age : NumberInt(31), location : '인천'}
   }
   ```

   드디어 totalKeysExamined 2로 되어 원하는 결과가 나왔다. 
   위의 결과로 인덱스는 동등조건이 범위조건보다 우선한다는 것을 알았다. 
   논리적으로 생각을 해봐도 범위조건을 먼저 다 찾고나서 동등조건을 찾는 3번의 구조로는 최적화가 안된다.

그렇다면 저의 경우 왜 문제가 발생 했을까요?

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

위와 같이 인덱스가 생성되어 있을 때 
query planner가 캐싱된 쿼리 플랜이 없다면 가능한 모든 쿼리 플랜을 조회 후 첫 batch(101)를 가장 좋은 성능으로 가져오는 플랜을 캐싱함
인덱스가 최적화가 되어있지 않기 때문에 age_1_location_1 인덱스가 채택되지 않는 경우가 발생한다.
따라서 인덱스를 location_1_age_1를 생성하고 location_1_age_1, location_1 둘다 삭제 해야한다.

그리고 중요한 한가지