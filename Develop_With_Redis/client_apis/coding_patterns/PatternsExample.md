# Redis patterns example
트위터 예제를 만들면서 여러 Redis 코딩 패턴을 배워보세요.

이 글은 PHP로 작성되고, Redis만을 데이터베이스로 하는 매우 간단한 [트위터 예제](https://github.com/antirez/retwis)의
설계 및 구현에 대해 설명합니다. 개발 커뮤니티에서는 키-값 저장소를 전통적으로 웹 애플리케이션을 위한 관계형 데이터베이스를 
대체할 수 없는 특수한 목적의 데이터베이스라고 여겨 왔습니다. 이 글은 키-값 계층 위에 설계된 Redis 자료 구조가 어떻게 효과적으로
많은 애플리케이션을 위한 데이터모델을 구현하는지 보여줍니다.

참고: 이 글의 원본은 2009년, Redis가 출시된더 때에 작성되었습니다. 그때는 Redis 데이터모델이 전체 애플리케이션을 작성하는데 효과적인지
명확하지 않았습니다. 5년이 지난 지금, Redis를 주 저장소로 사용하는 많은 애플리케이션이 있고, 이 글의 목적은 Redis를 처음 접하는 사람들을
위한 튜토리얼입니다. Redis를 활용해 간단한 데이터 계층을 설계하고, 여러 자료 구조를 활용하는 법을 배울 수 있습니다.

[Retwis(트위터 예제)](https://github.com/antirez/retwis)는 구조적으로 간단하고, 좋은 성능을 가지고 있습니다. 또한, 적은 노력으로
웹과 Redis 서버에 분산시킬 수 있습니다. [Retwis 소스코드를 참고하세요.](https://github.com/antirez/retwis)

보편적인 가독성을 위해 PHP로 작성했습니다. Ruby, Python, Erlang으로 작성해도 성능은 동일합니다. 몇 가지 클론들이 존재하지만,
모든 클론이 현재 튜토리얼 버전과 같은 데이터 계층을 쓰는 것은 아니니, 이 글을 참고하세요.
- [Retwis-RB](https://github.com/danlucraft/retwis-rb)는 Daniel Lucraft가 Sinatra랑 Ruby로 작성한 Retwis의 포트입니다.
- [Retwis-J](https://docs.spring.io/spring-data/data-keyvalue/examples/retwisj/current/)는 Retwis의 자바 포트이며, 
Costin Leau에 의해 Spring Data Framework를 활용하여 작성되었습니다. [깃헙](https://github.com/SpringSource/spring-data-keyvalue-examples)
에서 소스 코드를 확인할 수 있으며, [springsource.org](http://j.mp/eo6z6I) 이 곳에서 이해를 위한 문서를 참고할 수 있습니다.
> 포트는 어떤 프로그램이나 시스템을 다른 플랫폼이나 언어에서도 동작하도록 수정하거나 재작성하는 것이다.

## What is a key-value store?
***
키-값 저장소의 핵심은 값이라고 부르는 데이터를 키에 저장하는 것입니다. 값은 어느 키에 저장되었는지만 알면 조회가 가능합니다.
값을 가지고 키를 찾는 것은 직접접인 방법은 없습니다. 어느 의미에서는, 매우 큰 해시나 딕셔너리로 볼 수 있지만, 애플리케이션이 종료되도,
데이터가 영속성을 가집니다. 예를 들어, `SET`을 통해 키 `foo`에 값 `bar`를 저장합니다:
~~~redis
SET foo bar
~~~

Redis는 데이터를 영속적으로 저장하며, "foo에는 어느 값이 저장되어 있나?"라고 물으면 Redis는 `bar`로 응답할 것입니다:
~~~redis
GET foo => bar
~~~

다른 보편적인 연산으로는 `DEL`가 있습니다. `DEL`를 통해 특정 키와 그 키의 값들을 삭제할 수 있습니다. `SETNX`는 키가 존재하지
않는 경우에만 값을 설정하고, `INCR`은 키에 저장된 값을 원자적으로 증가시킵니다:
~~~redis
SET foo 10
INCR foo => 11
INCR foo => 12
INCR foo => 13
~~~

## Atomic operations
***
`INCR`에는 특별한 점이 있습니다. Redis가 왜 이런 연산을 굳이 구현해도 되는데 제공하는지 아시나요? 다음과 같이 간단한데 말이죠:
~~~redis
x = GET foo
x = x + 1
SET foo x
~~~

문제는 이러한 방식으로 값을 증가시키면 하나의 클라이언트에서는 잘 작동하지만, 여러 클라이언트가 동시에 키에 접근하면 문제가 생깁니다:
~~~redis
x = GET foo (yields 10)
y = GET foo (yields 10)
x = x + 1 (x is now 11)
y = y + 1 (y is now 11)
SET foo x (foo is now 11)
SET foo y (foo is now 11)
~~~

문제가 생기죠? 값을 두 번 증가시켰는데, 10에서 12로 증가하는 것이 아닌 11이 됩니다. 이러한 이유는 `GET/increment/SET` 조합의
증가 방식은 원자적이지 못하기 때문입니다. Redis나 Memcached의 `INCR`은 반면 원자적으로 동작합니다. 서버에서 연산이 끝날 때까지
키에 대한 보호를 함으로써 동시에 접근이 발생해도 안전하게 증가합니다.

Redis가 다른 키-값 저장소와 다른 점은 `INCR`같은 연산을 제공해 복잡한 문제들을 해결할 수 있게 해줍니다. 이것이 Redis를 통해 
다른 SQL같은 데이터베이스 없이도 전체 애플리케이션을 구현할 수 있는 이유입니다. 일일이 하나씩 다 신경안쓰고 말이죠.

## Beyond key-value stores: lists
***
이 섹션에서는 트위터 예제를 만들기 위한 Redis 기능들을 살펴볼 것입니다. 첫 번째는 Redis 값이 문자열만이 아니라는 것입니다.
Redis는 리스트, Set, 해시, Sorted set, Bitmap, HyperLogLog를 값으로 사용할 수 있게 지원하고, 이들에 대한 원자적 연산을
제공합니다. 그래서 같은 키에 대한 동시적인 접근에도 안전함을 보장합니다. 리스트부터 살펴보시죠:
~~~redis
LPUSH mylist a (now mylist holds 'a')
LPUSH mylist b (now mylist holds 'b','a')
LPUSH mylist c (now mylist holds 'c','b','a')
~~~

`LPUSH`는 Left Push라는 뜻입니다. 요소를 리스트의 맨 왼쪽에 추가합니다. 만약 mylist라는 키가 없다고 가정하면, 자동으로 빈 리스트를
만들고 `PUSH` 연산을 합니다. 당연하듯이 `RPUSH`도 존재하면, 요소를 리스트의 맨 오른쪽에 추가합니다. 이 기능은 트위터 예제에 매우 유용합니다.
사용자 업데이트는 `username:updates`라는 키에 저장된 리스트에 계속해서 추가할 수 있습니다.

리스트로부터 데이터를 조회하는 명령어에는 `LRANGE`가 있습니다. `LRANGE`는 리스트에서 주어진 범위의 요소들을 가져오거나, 전체 요소를 가져옵니다.
~~~redis
LRANGE mylist 0 1 => c,b
~~~

`LRANGE`는 0-based 인덱스를 사용하며, 첫 번째 요소의 인덱스가 0부터 시작한다는 뜻입니다. 인자는 `LRANGE key first-index last-index`와
같습니다. last-index 인자는 음수가 될 수 있으며, 특별한 의미를 가집니다: -1은 리스트의 마지막 요소를 가리킵니다. -2는 두 번째, 그 다음 순차적으로
가리킵니다. 전체 리스트를 가져오기 위해서는 다음과 같이 할 수 있습니다:
~~~redis
LRANGE mylist 0 -1 => c,b,a
~~~

다른 중요한 연산에는
- `LLEN`: 리스트의 요소 개수 조회.
- `LTRIM`: `LRANGE`와 다르게 주어진 범위를 트리밍. 이 연산은 주어진 범위를 가져오고 새로운 값으로 원자적으로 설정합니다.

## The Set data type
***
현재 이 튜토리얼에서는 set을 쓰지 않고 좀 더 유용한 set인 sorted set을 사용하고 있습니다. 그래도 set부터 설명을 하고 sorted set에
대해 설명하겠습니다.

리스트뿐만 아니라 여러 자료 구조가 있습니다. Redis는 set도 지원합니다. Set은 정렬되지 않은 요소들의 집합입니다. 요소를 추가, 삭제, 
존재유무 확인, 다른 set과의 교집합을 구할 수 있습니다. 마찬가지로 조회도 가능합니다. 예제를 통해 알아봅시다. `SADD`는 add to set이라는
뜻이고, `SREM`은 remove from set, `SISMEMBER`는 test if member, 마지막으로 `SINTER`는 perform intersection이라는 뜻입니다.
다른 연산으로는 `SCARD`: set의 기수성 조회, `SMEMBERS`: set의 모든 요소 반환이 있습니다.
~~~redis
SADD myset a
SADD myset b
SADD myset foo
SADD myset bar
SCARD myset => 4
SMEMBERS myset => bar,a,foo,b
~~~

`SMEMBER`는 set에 추가한 순서대로 요소를 반환하지 않는다는 점에 주의하세요. 순서를 유지하고 싶으면 리스트에 저장하세요.
~~~redis
SADD mynewset b
SADD mynewset foo
SADD mynewset hello
SINTER myset mynewset => foo,b
~~~

`SINTER`는 두 개이상의 set간의 교집합을 반환합니다. 4개,5개 또는 10000개도 가능합니다. 마지막으로 `SISMEMBER`가 어떻게 동작하는지
봅시다:
~~~redis
SISMEMBER myset foo => 1
SISMEMBER myset notamember => 0
~~~

## The Sorted Set data type
***
Sorted set은 set과 비슷합니다. 마찬가지로 요소들의 집합이죠. 하지만, Sorted set에서는 각 요소가 element score라는 값에 연결됩니다.
이 스코어때문에 sorted set안의 요소들은 정렬되어, 항상 두 요소를 스코어로 비교할 수 있습니다(점수가 같으면 문자열로 비교합니다).

Set처럼 sorted set에서는 중복된 값이 없으며, 모든 값이 고유합니다. 하지만, 요소의 스코어를 수정하는 것은 가능합니다.

Sorted set 명령들은 Z가 접두사로 붙습니다. 다음은 sorted set 예제입니다:
~~~redis
ZADD zset 10 a
ZADD zset 5 b
ZADD zset 12.55 c
ZRANGE zset 0 -1 => b,a,c
~~~

위의 예시에서는 `ZADD`를 통해 여러 요소를 추가했습니다. 그 후, `ZRANGE`를 가지고 조회했죠. 보시다시피, 요소들이 연결된 스코어에 의해
정렬된 채로 반환됩니다. 어떤 요소와 스코어가 존재하는지 확인하고 싶으면, `ZSCORE`를 사용할 수 있습니다:
~~~redis
ZSCORE zset a => 10
ZSCORE zset non_existing_element => NULL
~~~

Sorted set은 매우 유용한 자료 구조입니다. 요소들을 스코어를 가지고 범위, 사전순, 역순등으로 질의할 수 있습니다. 더 자세한 정보는
[여기](https://redis.io/docs/latest/commands/#sorted_set)를 확인하세요.

## The Hash data type
***
마지막 자료 구조는 거의 모든 언어에서 동일한 해시입니다. Redis 해시는 기본적으로 Ruby 또는 Python 해시와 같습니다. 해시는
값들에 연결된 필드의 집합입니다:
~~~redis
HMSET myuser name Salvatore surname Sanfilippo country Italy
HGET myuser surname => Sanfilippo
~~~

`HMSET`은 해시에서 필드를 설정하는데 사용합니다. `HGET`을 통해 이를 조회할 수 있습니다. `HEXISTS`를 통해 필드의 존재 유무를 확인할 수 있고,
`HINCRBY`로 해시 필드를 증가시킬 수 있습니다.

해시는 객체를 표현하는데 이상적인 자료 구조입니다. 예를 들어, 트위터 예제에서는 해시를 통해 사용자와 업데이트를 표현했습니다.

이제 주 자료 구조들의 기본을 알아보았으니, 코딩을 시작해보겠습니다.

## Prerequisites 
***
만약 Retwis 소스코드를 받지 않은 상태라면, 빨리 받으세요. PHP 파일들과 Predis의 복사본이 들어있습니다. Predis는 PHP 클라이언트
라이브러리입니다.

다른 것으로는 Redis 서버가 필요합니다. 소스를 받고 빌드라고 `./redis-server`로 실행시키세요. 다른 설정은 추가적으로 필요없습니다.

## Data layout
***
관계형 데이터베이스를 가지고 작업할 때, 스키마가 반드시 설계되어야 합니다. 하지만, Redis에는 테이블이 없으니 어떤 것을 설계해야 할까요?
객체를 표현하는데 필요한 키들을 식별하고, 키들이 어떤 값을 가지고 있어야 하는지 정해야 합니다.

사용자부터 시작해봅시다. 사용자를 표현하려면 이름, ID, 비밀번호, 팔로워, 팔로잉, 기타 관련 정보들이 필요합니다. 그럼 사용자를 어떻게
식별할까요? 관계형 데이터베이스처럼 고유한 숫자 ID로 식별하는 것이 좋습니다. 사용자에 대한 참조는 이 ID를 가지고 이루어지게 됩니다. 고유한
ID를 생성하는 것은 `INCR`로 원자적으로 간단하게 해결할 수 있습니다. `antirez`라는 새로운 사용자를 만들면 다음과 같이 할 수 있습니다:
~~~redis
INCR next_user_id => 1000
HMSET user:1000 username antirez password p1pp0
~~~

참고: 실제 애플리케이션에서는 해싱된 비밀번호를 사용해야 합니다. 여기서는 간단하게 하기 위해 이렇게 진행했습니다.

위에서 `next_user_id` 키를 가지고 매번 새로운 유저에게 고유 ID를 생성해줍니다. 그 후, 이 고유 ID를 이용해 해당 사용자의 데이터를 담는 
해시의 키 이름으로 사용합니다. 이런 방식은 키-값 저장소에서 흔히 사용되는 디자인 패턴입니다. 이미 정의한 필드 외에도, 사용자를 완전히 정의하기 
위해 필요한 요소들이 더 있습니다. 예를 들어, 종종 사용자 이름으로부터 사용자 ID를 조회해야 할 경우가 있습니다. 따라서 사용자를 추가할 때마다
다음과 같은 작업도 함께 수행합니다: User라는 해시에 필드는 사용자 이름, 값에는 해당 ID로 설정합니다. 
~~~redis
HSET users antirez 1000
~~~

처음에는 다소 낯설게 느껴질 수 있지만, 중요한 점은 Redis에서는 직접적인 방식으로만 데이터에 접근할 수 있다는 점입니다. Redis한테
특정 값을 가지고 있는 키들을 달라는 것은 불가능한 일입니다. 하지만 이것이 바로 Redis의 강점이기도 합니다. 이러한 새로운 패러다임은 데이터를
항상 기본 키를 통해 접근 가능하도록 구조화하도록 유도합니다.