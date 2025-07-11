# Redis sets

Redis Set은 유니크한 문자열들의 순서가 없는 집합입니다. 다음과 Redis Set을 효율적으로
사용하실 수 있습니다:
- 유니크한 항목 추적 (e.g., 블로그 게시물에 접근하는 모든 고유한 IP 주소들을 추적하기)
- 관계 표현 (e.g., 특정 역할을 가진 모든 사용자 집합)
- 합집합, 교집합, 차집합같은 집합 연산 수행

## Basic commands

- **SADD**는 집합에 새로운 멤버를 추가시킵니다.
- **SREM**은 특정한 멤버를 집합에서 제거합니다.
- **SISMEMBER**는 특정 문자열이 집합에 포함되어 있는지 검사합니다.
- **SINTER** 두 집합의 교집합을 반환합니다.
- **SCARD**는 집합의 크기(기수성)을 반환합니다.

## Examples

- 미국과 프랑스에서 경주하는 자전거들을 저장하는 예시입니다. 이미 집합에 존재하는 멤버를 추가할 경우, 새로 추가되지 않는 것에 주의하세요.
~~~redis
> SADD bikes:racing:france bike:1
(integer) 1
> SADD bikes:racing:france bike:1
(integer) 0
> SADD bikes:racing:france bike:2 bike:3
(integer) 2
> SADD bikes:racing:usa bike:1 bike:4
(integer) 2
~~~

- bike:1과 bike:2가 미국에서 경주하고 있는지 확인해보세요.
~~~redis
> SISMEMBER bikes:racing:usa bike:1
(integer) 1
> SISMEMBER bikes:racing:usa bike:2
(integer) 0
~~~

- 어느 자전거들이 두 경주 모두 참여하고 있을까요?
~~~redis
> SINTER bikes:racing:france bikes:racing:usa
1) "bike:1"
~~~

- 얼마나 많은 자전거들이 프랑스에서 경주중일까요?
~~~redis
> SCARD bikes:racing:france
(integer) 3
~~~

## Tutorial

**SADD** 명령어는 집합에 새로운 요소를 추가합니다. 또한, 이미 값이 존재하는지, 교집합, 차집합, 합집합같은 연산도 수행합니다.

~~~redis
> SADD bikes:racing:france bike:1 bike:2 bike:3
(integer) 3
> SMEMBERS bikes:racing:france
1) bike:3
2) bike:1
3) bike:2
~~~

아래의 예시에서 제가 3개의 요소를 제 집합에 추가하였고, 모든 멤버들을 반환하라고 명령하였습니다.
집합은 순서를 보장하지 않기 때문에, Redis는 매 호출마다 순서없이 요소를 반환합니다.

Redis는 집합에 대한 소속여부를 테스트할 수 있는 여러 명령어들이 있습니다. 이 명령어들은 한 개의 요소
또는 여러 개의 요소에 대해 수행가능합니다.

~~~redis
> SISMEMBER bikes:racing:france bike:1
(integer) 1
> SMISMEMBER bikes:racing:france bike:2 bike:3 bike:4
1) (integer) 1
2) (integer) 1
3) (integer) 0
~~~

두 집합의 차이점도 확인할 수 있습니다. 예를 들어 프랑스에서는 경주중이지만, 미국에서는 경주중이 아닌
자전거들을 확인할 수 있습니다.

~~~redis
> SADD bikes:racing:usa bike:1 bike:4
(integer) 2
> SDIFF bikes:racing:france bikes:racing:usa
1) "bike:3"
2) "bike:2"
~~~

다른 중요한 연산들도 Redis의 적절한 명령어들을 통해 쉽게 구현할 수 있습니다. 예를 들어
프랑스, 미국, 그리고 다른 나라에서 경주중인 모든 자전거들의 목록을 얻고 싶을 때, 여러 집합간의 교집합을
구하는 명령어인 **SINTER** 명령어를 통해 쉽게 구할 수 있습니다. 교집합뿐만 아니라 합집합, 차집합 그리고 
다른 연산들도 수행할 수 있습니다. 예시로, 다른 경주를 추가해 이를 확인해 볼 수 있습니다:

~~~redis
> SADD bikes:racing:france bike:1 bike:2 bike:3
(integer) 3
> SADD bikes:racing:usa bike:1 bike:4
(integer) 2
> SADD bikes:racing:italy bike:1 bike:2 bike:3 bike:4
(integer) 4
> SINTER bikes:racing:france bikes:racing:usa bikes:racing:italy
1) "bike:1"
> SUNION bikes:racing:france bikes:racing:usa bikes:racing:italy
1) "bike:2"
2) "bike:1"
3) "bike:4"
4) "bike:3"
> SDIFF bikes:racing:france bikes:racing:usa bikes:racing:italy
(empty array)
> SDIFF bikes:racing:france bikes:racing:usa
1) "bike:3"
2) "bike:2"
> SDIFF bikes:racing:usa bikes:racing:france
1) "bike:4"
~~~

**SDIFF**는 모든 공집합들의 차집합에 대해 빈 배열을 반환합니다. 또한 **SDIFF**에 입력되는
집합들의 순서도 **SDIFF**는 교환법칙을 따르지 않기 때문에 주의해야 합니다.

집합에서 항목을 제거하고 싶을 때에는 **SREM** 명령어를 통해 한 개 또는 여러 항목들을 제거할 수 있고,
**SPOP** 명령어를 통해 집합에서 임의의 항목을 제거할 수 있습니다. **SRANDMEMBER** 명령어를 통해 
임의의 값을 집합에서 제거하지 않고 반환받을 수 있습니다:

~~~redis
> SADD bikes:racing:france bike:1 bike:2 bike:3 bike:4 bike:4
(integer) 5
> SREM bikes:racing:france bike:1
(integer) 1
> SPOP bikes:racing:france
"bike:3"
> SMEMEBERS bikes:racing:france
1) "bike:2"
2) "bike:4"
3) "bike:5"
> SRANDMEMBER bikes:racing:france
"bike:2"
~~~

## Limits

Redis 집합의 최대 항목들 개수는 2^32 - 1 (4,294,967,295)입니다.

## Performance

확인, 제거, 멤버 확인같은 대부분의 집합 연산들은 O(1)에서 이루어집니다. 이것은 매우 효율적입니다.
하지만, 수백 수천, 또는 그 이상의 크기를 가지는 집합들에서는 **SMEMBERS** 명령어는 주의해야 합니다.
이 명령어는 한 집합에 대한 연산에 있어서 O(n)의 시간복잡도를 가집니다. 대체제로, **SSCAN**을
사용하여 집합의 모든 멤버를 반복적으로 조회할 수 있습니다.

## Alternatives

대규모 데이터셋 또는 스트리밍 데이터에 대해 집합의 멤버십을 검사할 경우, 많은 메모리를 사용할 수 있습니다.
정확성보다 메모리를 더 고려하신다면, **Bloom filter**를 사용할 수 있습니다.

Redis 집합은 일종의 인덱스로 자주 사용됩니다. 만약 데이터를 인덱싱하거나 쿼리할 필요가 있다면, 
**JSON** 데이터 타입과 **Redis Query Engine** 기능을 활용하는 것을 고려해보세요.

## Learn more

- **Redis Sets Explained**하고 **Redis Sets Elaborated**는 Redis 집합에 관한 짧은 비디오들입니다.
- **Redis University's RU101**에서는 Redis 집합에 대해 자세하게 다룹니다.