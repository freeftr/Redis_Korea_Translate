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

