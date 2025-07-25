# Secondary indexing
Redis에서 세컨더리 인덱스를 사용하는 방법

Redis는 값에 여러 복잡한 자료 구조들이 들어갈 수 있기 때문에, 정확히는 키-값 저장소는 아닙니다. 하지만, 외부적으로
키-값 쉘을 가지고 있습니다: API 수준에서는 키의 이름으로 접근합니다. Redis는 프라이머리 키만을 통한 접근을 허용한다고
해도 무방합니다. 하지만, Redis는 data structures 서버로서 이를 활용해 다양한 형태의 보조 인덱스를 생성하고, 복합
인덱스도 생성이 가능합니다.

이 문서는 다음의 자료 구조들을 가지고 Redis가 어떻게 인덱스를 생성하는지 설명합니다:
- 해시, JSON은 다양한 필드 타입을 가지고 있고, Redis query 엔진과 함께 사용됩니다.
- Sorted Set은 ID나 다른 숫자형 필드들을 가지고 보조 인덱스를 생성합니다.
- Sorted Set을 사전식 범위와 사용하면, 좀 더 발전된 보조 인덱스, 복합 인덱스, 그래프 탐색 인덱스를 사용할 수 있습니다.
- Set으로 무작위 인덱스를 생성합니다.
- List는 단순한 반복 인덱스나, 최신 N개 항목에 대한 인덱스를 생성할 수 있습니다.
- 시계열에 라벨 인덱스를 생성할 수 있습니다.

Redis를 가지고 인덱스를 생성하고 관리하는 것은 심화 주제입니다. 복잡한 쿼리를 사용해야 하는 경우에는 관계형 저장소가
더 적합한지 알아보아야 합니다. 하지만, 특히 캐싱이 필요한 상황에서는 빠른 조회를 위해 인덱싱된 데이터를 Redis에 저장해야
할 필요가 있습니다. 이는 실행을 위해 인덱스가 필요한 일반적인 쿼리 성능을 높이기 위해서입니다.

## Hashes and JSON indexes
***
Redis query 엔진은 다양한 형태의 필드를 가지고 JSON 키, 해시 모두를 인덱싱할 수 있게 해줍니다:
- TEXT
- TAG
- NUMERIC
- GEO
- VECTOR
- GEOSHAPE

`FT.CREATE`를 통해 해시와 JSON 키에 인덱스가 추가되면, 인덱스에 추가된 접두사를 사용하는 키들은 `FT.SEARCH`와
`FT.AGGREGATE` 통해 질의될 수 있습니다.

해시와 JSON 인덱스 생성에 더 많은 정보를 보고 싶으시다면 다음의 페이지들을 참조해주세요.
- [Hash indexes](https://redis.io/docs/latest/develop/ai/search-and-query/indexing/schema-definition/)
- [JSON indexes](https://redis.io/docs/latest/develop/ai/search-and-query/indexing/)

## Simple numerical indexes with sorted sets
***
Redis를 통해 생성할 수 있는 가장 간단한 보조 인덱스는 Sorted Set을 사용하는 것입니다. Sorted Set은 각 요소에
점수를 바탕으로 오름차순 정렬이 이루어집니다.

이때 점수가 배정밀도 부동소수점이기 때문에, 일반적인 Sorted Set으로 만들 수 있는 인덱스는 특정 범위의 숫자 값을 갖는
필드를 기준으로만 인덱스를 생성할 수 있습니다.

이러한 인덱스를 만드는 두 명령어는 `ZADD`와 `ZRANGE`입니다. `ZRANGE`는 `BYSCORE` 인자를 통해 지정한 범위 내의
요소들을 조회할 수 있습니다.

예를 들어, 사람 이름의 집합을 나이로 sorted set을 활용해 인덱싱할 수 있습니다. 요소는 사람의 이름이고, 점수는 나이입니다.
~~~redis
ZADD myindex 25 Manuel
ZADD myindex 18 Anna
ZADD myindex 35 Jon
ZADD myindex 67 Helen
~~~

20세에서 40세 사이의 사람들을 조회하기 위해서는 다음과 같이 명령하면 됩니다:
~~~redis
ZRANGE myindex 20 40 BYSCORE
1) "Manuel"
2) "Jon"
~~~

`ZRANGE`의 `WITHSCORES` 옵션을 통해 반환된 요소들의 점수도 같이 조회가 가능합니다.

`ZCOUNT`는 주어진 범위 요소들의 개수를 조회합니다. 실제 요소들을 가져오지 않고, 개수만 반환하기 때문에 효율적이며,
범위의 크기와 상관없이 로그 시간안에 수행됩니다.

범위는 포함적일 수도 있고, 예외적일 수도 있습니다. 다음의 `ZRANGE` [문서](https://redis.io/docs/latest/commands/zrange/)를 참고하세요.

참고: `ZRANGE`를 `BYSCORE`와 `REV` 인자와 함께 사용하는 경우, 범위를 역순으로 질의하는 것도 가능합니다. 이는
데이터가 특정 방향(오른차순 또는 내림차순)으로 인덱싱되어 있을 때, 반대 방향으로 데이터를 조회하고 싶을 때 유용합니다.

### Use object IDs as associated values
위의 예시에서는 이름을 나이에 묶었습니다. 하지만, 일반적으로는 인덱싱하려는 필드와 관련된 실제 데이터가 다른 곳에 저장되어
있을 수 있습니다. 이럴 경우, sorted set의 값에 해당 데이터를 직접 저장하는 대신, 그 객체의 ID만 저장할 수도 있습니다.

예를 들어, 사용자들을 대표하는 Redis 해시가 있습니다. 각 사용자는 단일 키로 표현이 되며, ID로 바로 접근할 수 있습니다:
~~~redis
HMSET user:1 id 1 username antirez ctime 1444809424 age 38
HMSET user:2 id 2 username maria ctime 1444808132 age 42
HMSET user:3 id 3 username jballard ctime 1443246218 age 33
~~~

사용자들의 나이로 질의할 수 있게 인덱스를 만들고 싶다면 다음과 같이 할 수 있습니다:
~~~redis
ZADD user.age.index 38 1
ZADD user.age.index 42 2
ZADD user.age.index 33 3
~~~

이번에는 점수에 연관된 값이 객체의 ID입니다. 띠라서 `ZRANGE`와 `BYSCORE`로 인덱스를 질의를 하면, `HGETALL` 또는
비슷한 명령어들로 필요한 정보들을 추가로 조회해야 합니다. 장점은 인덱스를 건들지 않고도 객체들을 수정할 수 있다는 것입니다
(인덱싱된 필드를 건드리지만 않는다면).

다음의 예시들에서는 인덱스에 연결된 값으로 ID를 거의 사용할 것입니다. 일부 예외적인 경우를 빼면 일반적인 경우이기 때문입니다.

### Update simple sorted set indexes
시간에 따라 변하는 값을 종종 인덱싱하기도 합니다. 위의 예시에서는 사용자들의 나이는 매년 바뀝니다. 이러한 경우에서는 
나이 그 자체보다는 출생년월일에 인덱싱을 적용하는 것이 맞을 수도 있습니다. 하지만 다른 상황에서는 특정 필드가 가끔씩 변경되며,
그에 따라 인덱스도 함께 갱신되길 원하는 경우도 있습니다.

`ZADD` 명령어는 단순한 인덱스를 갱신하는 것을 매우 간단하게 만들어줍니다. 같은 요소를 다른 점수로 추가하면, 해당 요소를
맞는 순위로 옮기고, 점수를 갱신합니다. 따라서, 만약 antirez가 39세가 되면, 해당 사용자를 나타내는 해시 정보와 인덱스를
갱신하기 위해 다음의 두 명령어들을 실행하면 됩니다:
~~~redis 
HSET user:1 age 39
ZADD user.age.index 39 1
~~~

이 작업은 MULTI/EXEC 트랜잭션으로 묶어 실행함으로써, 두 필드가 모두 함께 수정되거나, 전혀 수정되지 않도록 보장할 수 있습니다.

### Limits of the score
Sorted set의 요소들의 점수는 배정밀도 부동소수(double precision float)입니다. 이것은 내부적으로 지수 표현을 사용하기 
때문에 다양한 소수와 정수형 값들을 표현할 수 있지만, 정밀도에 따라 오차가 발생할 수 있습니다. 그러나, 인덱싱 측면에서
중요한 점은 -9007199254740992 ~ 9007199254740992(-/+ 2^53) 사이의 숫자들은 오차없이 표현할 수 있습니다.

