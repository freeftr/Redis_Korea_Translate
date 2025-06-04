# Redis sorted sets

Redis Sorted Set은 고유한 연관된 점수에 따라 고유한 문자열들이 정렬된 상태로 저장하는
집합입니다. 만약 연관된 점수가 같은 문자열이 존재할 경우, 사전순으로 정렬됩니다.Sorted sets의 주요 활용 사례는 다음과 같습니다:
- 리더보드: 대규모 온라인 게임같은 상황에서 점수에 따라 순위표를 Sorted Set으로 간단히 구현할 수 있습니다.
- Rate Limiters: 너무 많은 API 요청을 제한하기 위한 슬라이딩 윈도우 기반 요율 제한기를 Sorted Set으로 간단히 구현할 수 있습니다.

Sorted Set은 Set과 Hash의 중간 형태로 볼 수 있습니다. Sorted Set은 Set처럼
고유하고 중복되지 않는 문자열 요소들로 구성되어 있기 때문에, 어떤 면에서는 Set과 공통됩니다.

하지만 Set에서는 요소들이 정렬된 상태로 저장되어 있지 않지만, Sorted Set에서는 모든 요소가
"스코어"라고 불리는 실수형 값에 연관되어 있습니다. (스코어에 모든 요소가 매핑되어 있기 때문에 해시와
비슷한 성격을 가집니다.)

또한 Sorted Set의 모든 값들은 정렬된 상태를 유지합니다(이때 정렬은 요청에 의해서 수행되는 것이 아닌
Sorted Set의 특성에 의해 자동으로 정렬됩니다). 다음의 규칙에 의해 정렬됩니다:
- 만약 B와 A라는 요소들이 있고, 서로 다른 스코어를 가진다면, A.score > B.score 일때 A > B가 
성립합니다.
- 만약 B와 A가 같은 스코어를 가진다면, A 문자열이 B 문자열보다 사전순으로 앞설 때 A > B가 성립합니다.

간단한 예제를 통해 알아보겠습니다. 경주 선수들과 첫 번째 경주에서의 선수들의 성적을 추가해보도록 하겠습니다. 

~~~redis
> ZADD racer_scores 10 "Norem"
(integer) 1
> ZADD racer_scores 12 "Castilla"
(integer) 1
> ZADD racer_scores 8 "Sam-Bodden" 10 "Royce" 6 "Ford" 14 "Prickett"
(integer) 1
~~~

**ZADD** 명령어는 **SADD** 명령어와 비슷해보이지만, 인자를 하나 더 받는다는 차이점이 있습니다. (추가할 값 앞에 위치, 점수)
또한 **ZADD** 명령어는 가변 인자를 지원하므로, 위의 예시와 같이 여러 스코어-값 쌍들을 한 번에 지정할 수 있습니다.

Sorted Set을 사용하면 참가 선수들의 순위를 받는 것이 간단합니다. 이미 Sorted Set에서 정렬되어 있기 때문이죠.

참고: Sorted Set은 스킵 리스트와 해시 테이블을 동시에 사용하는 이중 구조로 구현되어 있습니다. 따라서 새로운 요소를 추가할 때 Redis는
O(log(N))의 연산을 수행합니다. 정렬된 요소들을 조회할 때 이미 정렬되어 있는 상태로 존재하기 때문에 Redis는 추가적으로 정렬 연산을
수행할 필요가 없습니다. **ZRANGE** 명령어는 오름차순으로 정렬되고, **ZREVRANGE**는 내림차순으로 정렬됩니다:

~~~redis
> ZRANGE racer_scores 0 -1
1) "Ford"
2) "Sam-Bodden"
3) "Norem"
4) "Royce"
5) "Castilla"
6) "Prickett"
> ZREVRANGE racer_scores 0 -1
1) "Prickett"
2) "Castilla"
3) "Royce"
4) "Norem"
5) "Sam-Bodden"
6) "Ford"
~~~

참고: 0하고 -1은 인덱스 0부터 마지막까지를 의미합니다. (-1은 **LRANGE**에서처럼 같은 의미를 가집니다.)

**WITHSCORES** 인자를 통해 요소들의 스코어를 함께 반환받을 수 있습니다.

~~~redis
> ZRANGE racer_scores 0 -1 withscores
 1) "Ford"
 2) "6"
 3) "Sam-Bodden"
 4) "8"
 5) "Norem"
 6) "10"
 7) "Royce"
 8) "10"
 9) "Castilla"
10) "12"
11) "Prickett"
12) "14"
~~~

## Operating on ranges

Sorted Set의 기능은 여기서 끝이 아닙니다. Sorted Set은 범위를 활용한 연산이 가능합니다.
**ZRANGEBYSCORE** 명령어를 통해 점수가 10이하인 선수들을 조회할 수 있습니다:

~~~redis
> ZRANGEBYSCORE racer_scores -inf 10
1) "Ford"
2) "Sam-Bodden"
3) "Norem"
4) "Royce"
~~~

위의 예시에서 Redis에서 점수가 10이하인 모든 선수들을 조회해 보았습니다.(양쪽 경계값 포함)

**ZREM** 명령어를 통해 선수의 이름으로 요소를 삭제할 수 있습니다. 범위를 가지고 범위 안에 해당하는 요소들을
삭제하는 것도 가능합니다. Castilla 선수와 점수가 10점 미만인 모든 선수들을 삭제해보겠습니다:

~~~redis
> ZREM racer_scores "Castilla"
(integer) 1
> ZREMRANGEBYSCORE racer_scores -inf 9
(integer) 2
> ZRANGE racer_scores 0 -1
1) "Norem"
2) "Royce"
3) "Prickett"
~~~

**ZREMRANGEBYSCORE** 명령어는 이름은 좀 그렇지만 매우 유용한 명령어입니다.**ZREMRANGEBYSCORE** 명령어는
제거된 요소의 개수도 반환합니다.

Sorted Set의 매우 유용한 명령어 중에는 순위를 조회하는 명령어도 있습니다. 정렬된 집합 내에서 몇 번째 위치에 있는지
조회할 수 있습니다. **ZREVRANK** 명령어를 통해 내림차순에서의 위치도 조회할 수 있습니다.

~~~redis
> ZRANK racer_scores "Norem"
(integer) 0
> ZREVRANK racer_scores "Norem"
(integer) 2
~~~

## Lexicographical scores

Redis 2.8 버전에서는 집합에 점수가 같은 모든 요소들이 들어있다고 가정했을때, 사전순으로 범위를 조회할 수 있는
기능이 추가되었습니다.(요소들은 C 언어의 memcmp로 비교되어, 지역화의 영향을 받지 않으며, 모든 Redis 인스턴스가 동일한
값을 반환하는 것을 보장합니다.)

사전순 범위를 다루는 주요 명령어에는 **ZRANGEBYLEX**, **ZEREVRANGEBYLEX**, **ZREMRANGEBYLEX**, **ZLEXCOUNT**가 있습니다.

예시로, 점수가 모두 0인 유명한 선수들을 추가해보겠습니다. Sorted Set의 정렬 규칙에 따라 사전순으로 정렬된 됩니다. **ZRANGEBYLEX**를
사용하여 사전순 범위 조회를 할 수 있습니다:

~~~redis
> ZADD racer_scores 0 "Norem" 0 "Sam-Bodden" 0 "Royce" 0 "Castilla" 0 "Prickett" 0 "Ford"
(integer) 3
> ZRANGE racer_scores 0 -1
1) "Castilla"
2) "Ford"
3) "Norem"
4) "Prickett"
5) "Royce"
6) "Sam-Bodden"
> ZRANGEBYLEX racer_scores [A [L
1) "Castilla"
2) "Ford"
~~~

범위는 포함 또는 제외로 지정할 수 있으며(첫 번째 문자에 따라), 또한 양과 음의 무한대는 "+", "-"로 표현됩니다.
더 자세한 내용은 공식 문서를 확인하세요.

이 기능은 Sorted Set을 일반적인 인덱스로 사용할 수 있게 해주기 때문에 중요합니다. 예를 들어, 128비트 부호 없는 정수로 요소를 인덱싱하고 싶다면, 
요소들을 동일한 점수로 정렬된 집합에 추가하되, 각 요소의 앞에 16바이트로 구성된 128비트 big-endian 숫자를 접두사로 붙여 저장하면 됩니다.
big-endian 방식으로 저장된 숫자는 바이트 순서대로 사전순 정렬하면 곧 숫자 정렬과 동일한 순서가 되므로, 정렬된 집합에서 128비트 숫자 범위로 조회할 수 있으며, 
이때 접두사를 제외한 실제 값을 활용하면 됩니다.

### Updating the score: leaderboards
***

다음 주제로 넘어가기전 Sorted Set에 대해 한 가지만 더 말씀드리자면, Sorted Set의 스코어는 언제든
수정이 될 수 있습니다. Sorted Set에 들어있는 요소에 대해 **ZADD** 명령어를 호출하는 것만으로도
O(log(N))으로 스코어를 수정할 수 있습니다. 따라서, Sorted Set은 대량의 수정에 적합합니다.

### Examples
***

- 리더보드를 Sorted Set으로 구현하는 방법에는 두 가지가 있습니다. 선수의 최신 점수를 알고 있으면,
**ZADD** 명령어를 통해 직접 수정할 수 있습니다. 하지만, 기존 점수에 점수를 더하고 싶으면 **ZINCRBY**
명령어를 사용할 수 있습니다.
~~~redis
> ZADD racer_scores 100 "Wood"
(integer) 1
> ZADD racer_scores 100 "Henshaw"
(integer) 1
> ZADD racer_scores 150 "Henshaw"
(integer) 0
> ZINCRBY racer_scores 50 "Wood"
"150"
> ZINCRBY racer_scores 50 "Henshaw"
"200"
~~~

**ZADD** 명령어는 기존에 요소가 이미 존재할 시 0을 반환하는 것을 볼 수 있고, **ZINCRBY** 명령어는
새로운 점수를 반환하는 것을 볼 수 있습니다. Henshaw 선수의 점수는 처음에 100이었고, 이전 점수를 고려하지 않고 150으로 변경된 뒤,
50이 증가하여 200이 되었습니다.

### Basic commands
***

- **ZADD**는 Sorted Set에 요소-스코어 쌍을 추가합니다. 이미 요소가 존재할 시 기존의 값을 수정합니다.
- **ZRANGE**는 Sorted Set의 요소들을 지정된 범위 내에서 정렬된 순서로 반환합니다. 
- **ZRANK**는 인자로 받은 요소의 순위를 반환하고 오름차순을 기준으로 합니다.
- **ZREVRANK**는 인자로 받은 요소의 순위를 반환하고 내림차순을 기준으로 합니다.

Sorted Set의 전체 명령어를 확인하세요.

### Performance
***

대부분의 Sorted Set 연산은 O(log(n))에서 이루어지며 n는 요소의 개수입니다.

**ZRANGE** 명령어를 사용할 때는 반환되는 값의 수가 많을 경우 주의해야 합니다.
O(log(n) + m)의 시간복잡도에서 이루어지며, m은 반환되는 값의 수입니다.

### Alternatives
***

Redis Sorted Set은 종종 다른 Redis 자료형들을 인덱싱하는 데 사용되기도 합니다. 데이터를
인덱싱하거나 쿼리해야할 일이 있다면, Json이나 Redis Query Engine 기능을 고려하세요.

### Learn more
***

- **Redis Sorted Sets Explained**는 Sorted Set에 관한 소개입니다.
- **Redis University's RU101**에서 Redis Sorted Set을 자세히 다룹니다.