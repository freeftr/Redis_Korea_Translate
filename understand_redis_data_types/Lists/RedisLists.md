# Redis lists

Redis 리스트는 문자열 값들로 연결된 리스트입니다. Redis 리스트는 다음과 같은 상황에서 주로 쓰입니다:
- 스택, 큐 구현
- 백그라운드 워커 시스템에서 작업 대기열을 관리할 때

## Basic Commands

***

- **LPUSH**로 리스트의 첫 번째에 요소를 넣을 수 있습니다; **RPUSH**를 통해 리스트의 마지막에 요소를 넣을 수 있습니다.
- **LPOPS**로 리스트의 첫 번째 요소를 꺼내고 제거합니다; **RPOPS**는 리스트의 마지막 요소를 꺼내고 제거합니다,.
- **LLEN**으로 리스트의 길이 값을 받을 수 있습니다.
- **LMOVE**로 요소들을 한 리스트에서 다른 리스트로 원자적으로 옮길 수 있습니다.
- **LRANGE**로 리스트에서 요소들의 범위를 받을 수 있습니다.
- **LTRIM**으로 리스트를 지정된 범위의 요소들로 잘라서 줄여줍니다.

### Blocking Commands

리스트는 여러 차단 명령어들을 제공합니다. 예를 들어:

- **BLPOP**은 리스트의 첫 번째 요소를 꺼내어 반환합니다. 리스트가 비어 있을 경우, 요소가 추가될 때까지 대기하거나 정해진 Timeout까지 대기하게 됩니다.
- **BLMOVE**는 소스 리스트의 요소를 타겟 리스트로 원자적으로 옮깁니다. 만약 소스 리스트가 비어있다면, 새로운 요소가 추가될 때까지 대기하게 됩니다.

**complete series of list commands**를 통해 전체 명령어를 확인하세요.

## Examples

***

- 리스트를 큐처럼 사용하기 (FIFO)

```redis
> LPUSH bikes:repairs bike:1
(integer) 1
> LPUSH bikes:repairs bike:2
(integer) 2
> RPOP bikes:repairs
"bike:1"
> RPOP bikes:repairs
"bike:2"
```

- 리스트를 스택처럼 사용하기 (FILO)

```redis
> LPUSH bikes:repairs bike:1
(integer) 1
> LPUSH bikes:repairs bike:2
(integer) 2
> LPOP bikes:repairs
"bike 2"
> LPOP bikes:repairs
"bike 1"
```

- 리스트의 길이 확인하기

```redis
> LLEN bikes:repairs
(integer) 0
```

- 원자적으로 요소를 꺼내서 다른 리스트에 집어넣기

```redis
> LPUSH bikes:repairs bike:1
(integer) 1
> LPUSH bikes:repairs bike:2
(integer) 2
> LMOVE bikes:reparis bikes:finished LEFT LEFT
"bike 2"
> LRANGE bikes:repairs 0 -1
1) "bike:1"
> LRANGE bikes:finished 0 -1
1) "bike:2"
```

- **LTRIM**을 이용하여 리스트의 길이를 제한하기

```redis
> RPUSH bikes:repairs bike:1 bike:2 bike:3 bike:4 bike:5
(integer) 5
> LTRIM bikes:repairs 0 2
OK
> LRANGE bikes:repairs 0 -1
1) "bike:1"
2) "bike:2"
3) "bike:3"
```

### What are lists?

IT 업계에서 "List"라는 용어가 종종 부적절하게 사용되는 경우가 많기 때문에, List 자료형에 대해 말하기 전에 먼저 이론적인
부분을 살펴봐야 합니다. 예를 들어, Python Lists는 흔히 말하는 연결 리스트가 아닌 실제로는 배열(Array)을 가리킵니다.

일반적인 관점에서 리스트는 순서가 있는 요소들의 집합입니다: 10, 20, 1, 2, 3는 리스트입니다. 하지만 배열로 구현된 리스트와
연결 리스트로 구현된 리스트는 매우 다릅니다. 

Redis 리스트는 연결 리스트로 구현되었습니다. 이는 다른 말로, 리스트 안에 수백만 개의 요소가 있더라도, 리스트의 앞이나 뒤에 요소를
추가하는 작업은 항상 상수 시간(O(1))안에 처리됩니다. 리스트의 앞에 LPUSH로 요소를 넣는 처리 시간은 리스트에 10개의 요소가 있든 1000만개
요소가 있든 상관없이 동일합니다.

단점에는 어떤 것이 있을까요? Array 기반의 리스트에서는 인덱스를 통해 요소에 접근하는 속도(O(1))가 빠르고, 연결 리스트에서는 그렇지
못합니다. 연결 리스트에서는 리스트의 처음부터 순차적으로 탐색해야 하므로 작업량이 접근하는 요소의 인덱스에 비례합니다.

Redis List는 연결 리스트로 구현되어 있습니다. 그 이유는 데이터베이스 시스템에서는 매우 긴
리스트에 데이터를 빠르게 추가하는 것이 중요하기 때문입니다. 또다른 강점으로는, Redis 리스트는
일종한 길이를 일정한 시간만큼으로 가져올 수 있습니다.

엄청 많은 요소들의 컬렉션의 중간 요소에 접근할 때는 Sorted Set 자료형을 사용할 수 있습니다.
Sorted Set에 대해서는 **Sorted sets** 튜토리얼 페이지에서 확인하실 수 있습니다.

### First steps with Redis Lists

**LPUSH** 명령어는 새로운 요소를 리스트의 왼쪽에(또는 머리)에 추가합니다. 그에 반면
**RPUSH** 명령어는 새로운 요소를 리스트의 오른쪽에(또는 꼬리)에 추가합니다.
**LRANGE** 명령어는 범위 안의 요소들을 리스트에서 꺼내옵니다.:

~~~redis
> RPUSH bikes:repairs bike:1
(integer) 1
> RPUSH bikes:repairs bike:2
(integer) 2
> LPUSH bikes:repairs bike:important_bike
(integer) 3
> LRANGE bikes:repairs 0 -1
1) "bike:important_bike"
2) "bike:1"
3) "bike:2"
~~~

**LRANGE**는 범위의 시작점과 끝점, 두 개의 인덱스를 필요로 합니다. 두 인덱스 모두 음수일 수 있으며,
이것은 Redis에게 마지막부터 카운팅하라는 뜻입니다: -1은 마지막 요소, -2는 끝에서 두번째 요소,
그리고 계속 이어집니다.

**RPUSH**는 위와 같이 리스트이 오른쪽에 요소들을 추가하고 **LPUSH**는 리스트의 왼쪽에
요소들을 추가합니다.

두 명령어 모두 **가변 인자 명령어**이고, 이것은 한번의 호출로 여러 요소들을 집어 넣을 수 있다는 뜻입니다.

~~~redis
> RPUSH bikes:repairs bike:1 bike:2 bike:3
(integer) 3
> LPUSH bikes:repairs bike:important_bike bike:very_important_bike
> LRANGE bikes:repairs 0 -1
1) "bike:very_important_bike"
2) "bike:important_bike"
3) "bike:1"
4) "bike:2"
5) "bike:3"
~~~

Redis는 리스트에 요소가 없으면 NULL을 반환합니다.

### Common use cases for lists

리스트는 여러 작업에서 매우 유용합니다. 다음은 예시는 대표적인 두 가지 예시입니다:
- 사용자들이 소셜 네트워크에 포스팅한 최신의 게시물들 기억하기
- 생산자-소비자 패턴을 사용하는 프로세스간 통신에서 생산자는 항목들을 리스트에 푸쉬하고 소비자는 그 항목들을
리스트에서 꺼내서 소비하고, 추가적인 행동을 취합니다. Redis는 이와 같은 예시에서 보다 효율적이고, 신뢰할 수 있게
여러 특별한 명령어들을 제공합니다. 

예시로 유명한 Ruby 라이브러리인 **resque**와 **sidekiq**는 백그라운드 작업을 구현하기 위해
내부적으로 Redis 리스트를 사용합니다.

유명한 소셜 네트워크 서비스인 트위터는 Redis 리스트를 사용해 사용자들의 최신 트윗들을 가져옵니다.

이와 같은 사용 사레들을 단계별로 설명하기 위해, 당신의 홈페이지가 사진 공유 소셜 네트워크의 올려진
최신의 사진들을 보여주고, 이것을 빠르게 접근할 수 있게 하고 싶다고 가정합시다.

- 사용자가 매번 사진을 올릴 때마다, 사진의 ID를 LPUSH를 통해 Redis 리스트에 넣습니다.
- 사용자들이 홈페이지를 방문할 때, LRANGE 0 9 명령어를 사용해 가장 최신의 10개 게시물을 가져옵니다.

### Capped lists

대부분의 경우 최신 항목들을 저장하기 위해 그 어떤 서비스든 리스트를 사용합니다: SNS 서비스, 로그, 그
외 어떤 것들

Redis는 리스트를 고정된 크기의 컬렉션처럼 사용할 수 있게 해줍니다. **LTRIM** 명령어를 통해
최신의 N개 항목들만 기억하고 오래된 데이터들을 모두 버립니다.

**LTRIM** 명령어는 **LRANGE**와 비슷하지만, 범위 안의 요소들을 보여주는 것이 아닌,
범위 안의 요소들을 새로운 리스트 값들로 바꿉니다. 이때 범위 밖의 요소들을 모두 제거됩니다.

예시로, 당신이 수리 목록의 끝에 자전거들을 추가하는데, 가장 최신의 3개 데이터만 보고싶다고 
가정하면 다음과 같이 사용할 수 있습니다:
~~~redis
> RPUSH bikes:repairs bike:1 bike:2 bike:3 bike:4 bike:5
(integer) 5
> LTRIM bikes:repairs 0 2
OK
> LRANGE bikes:repairs 0 -1
1) "bike:1"
2) "bike:2"
3) "bike:3"
~~~