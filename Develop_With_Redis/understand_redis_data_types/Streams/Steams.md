# Redis Streams
Redis Streams에 대한 소개

Redis Stream은 append-only 로그같은 방식으로 동작하는 자료 구조입니다. 하지만, 일반적인 append-only 로그의 한계를
극복할 수 있는 여러 연산을 제공합니다. 시간복잡도 O(1)의 랜덤 액세스와 소비자 그룹같은 복잡한 소비 전략등이 포함됩니다.
Streams를 통해 이벤트를 실시간으로 기록하고 동시에 배포할 수 있습니다. 다음은 Redis Stream의 사용사례입니다:

- 이벤트 소싱 (예: 사용자 행동, 클릭)
- 센서 모니터링 (예: 장비들 감시)
- 알림 (예: 별개의 스트림에 각 사용자의 알림 저장)

Redis는 각 스트림 항목에 대해 고유한 ID를 생성합니다. 이 ID를 가지고 관련 항목을 조회하거나, 스트림 내에서
이후의 항목들을 읽거나 처리할 수 있습니다. 이 ID들은 시간에 관계되어 있기 때문에, 이 문서에 표시된 ID는 실제 Redis 
인스턴스에서 보게 되는 ID와 다를 수 있습니다.

Redis 스트림은 여러 트리밍 전략(스트림이 무한으로 늘어나는 것을 방지하기 위함)과 여러가지 소비 전략을 지원합니다.(
`XREAD`, `XREADGROUP`, `XRANGE`를 확인해보세요.)

## Basic commands
***
- `XADD`는 새 항목을 스트림에 추가합니다.
- `XREAD`는 지정된 위치에서 시간 순으로 앞으로 이동하면서 하나 이상의 항목을 읽습니다.
- `XRANGE`는 두 개의 입력된 항목 ID 사이의 항목들의 범위를 반환합니다.
- `XLEN`은 스트림의 길이를 반환합니다.

다음 링크에서 스트림 명령어들의 전체 목록을 확인하세요.
- https://redis.io/docs/latest/commands/?group=stream

# Examples
***
- 선수들이 체크포인트를 지날 때마다, 각 선수에 대한 정보(이름, 속도, 위치, 장소 ID)를 기록한 스트림 항목을 추가합니다:
~~~redis
> XADD race:france * rider Castilla speed 30.2 position 1 location_id 1
"1692632086370-0"
> XADD race:france * rider Norem speed 28.8 position 3 location_id 1
"1692632094485-0"
> XADD race:france * rider Prickett speed 29.7 position 2 location_id 1
"1692632102976-0"
~~~

- ID 1692632086370-0에서 시작하는 두 스트림 항목들을 읽습니다:
~~~redis
> XRANGE race:france 1692632086370-0 + COUNT 2
1) 1) "1692632086370-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "30.2"
      5) "position"
      6) "1"
      7) "location_id"
      8) "1"
2) 1) "1692632094485-0"
   2) 1) "rider"
      2) "Norem"
      3) "speed"
      4) "28.8"
      5) "position"
      6) "3"
      7) "location_id"
      8) "1"
~~~

- 스트림 끝에서 시작하여 최대 100개의 새 스트림 항목을 읽고, 항목이 쓰여지지 않으면 최대 300밀리초 동안 대기하려면
다음과 같습니다:
~~~redis
> XREAD COUNT 100 BLOCK 300 STREAMS race:france $
(nil)
~~~

## Performance
***

스트림에 새로운 항목을 추가하는 작업은 시간복잡도 O(1)입니다. 단일 항목에 대한 접근 시간복잡도 O(n)이고, 여기서
n은 ID의 길이입니다. 스트림 ID는 보통 짧고 고정된 길이를 가지기 때문에, 보통 상수 시간에 처리됩니다. 그 이유에 대해
알고 싶다면 스트림은 **radix trees**로 구현되어 있다는 것을 참고하면 됩니다.
- https://en.wikipedia.org/wiki/Radix_tree
> Radix Tree
Radix 트리는 압축된 Trie 구조라고 볼 수 있다. Trie 구조에서는 한 글자씩 노드에 할당하지만, Radix 트리는 
여러 문자를 한 노드에 저장해 압축한다.
DNS 룩업, 라우팅 테이블등에 사용이 된다.

간단히 말해서, Redis 스트림은 매우 효율적인 삽입과 읽기 성능을 가졌습니다. 시간 복잡도에 대한 내용은
각 명령어 페이지에서 확인할 수 있습니다.

## Stream basics
***
스트림은 append-only 자료구조입니다. 기본적인 쓰기 명령어인 `XADD`는 특정한 스트림에 새로운 항목을 추가합니다.

각 스트림 항목은 하나 이상의 키-값 쌍들을 가지며, 이는 dictionary 또는 Redis 해시와 비슷합니다:
~~~redis
> XADD race:france * rider Castilla speed 29.9
position 1 location_id 2
"1692632147973-0"
~~~

위의 `XADD` 호출은 ` rider: Castilla, speed: 29.9, position: 1, location_id: 2`을 `race:france` 스트림에
추가됩니다. 이때 1692632147973-0라는 자동 생성된 항목 ID를 반환합니다. 첫 번째 인자는 키의 이름이고, 두 번째 인자는 
스트림 내의 각 항목을 식별하는 ID입니다. 하지만, 이 사례에서는 *를 사용했기 때문에 Redis 서버가 자동으로 새로운 ID를
생성해줍니다. 모든 새로운 ID는 단조적으로 증가합니다. 쉽게 바꿔 말하면, 모든 새로운 항목은 기존의 항목들과 비교해서 항상
큰 ID 값을 가질 것입니다. ID를 서버가 자동으로 생성하는 방식이 거의 항상 바람직하며, 명시적으로 ID를 지정해야 하는 경우는 
매우 드뭅니다. 이에 대해서는 이후에 더 설명합니다. 각 스트림 항목이 ID를 가지고 있다는 점은 로그와 비슷하며, 로그에서는
행 번호나 바이트 오프셋을 통해 항목을 식별할 수 있는 것처럼 말입니다. `XADD` 예시로 돌아와서, 키 이름과 ID 뒤에는
스트림 항목을 구성하는 키-값들이 인자로 들어옵니다.

`XLEN`을 통해 스트림 안의 항목들의 개수를 조회할 수 있습니다:
~~~redis
> XLEN race:france
(integer) 4
~~~

### Entry IDs
`XADD`로 반환받은 항목 ID는 스트림 내에서 각 항목을 고유하게 식별하며, 두 부분으로 구성됩니다:
~~~redis
<millisecondsTime>-<sequenceNumber>
~~~

밀리초 부분은 스트림 ID를 생성하는 Redis 노드의 로컬 시간입니다. 하지만 만약 현재 밀리초 시간이 이전의 항목 시간보다
작으면 이전 항목 시간을 사용이 대신 사용됩니다. 이는 시스템 시계가 되돌아가는 일이 발생해도 ID를 단조적으로 증가시키는 것을
유지시키기 위해서 입니다. 시퀸스 번호는 같은 밀리초 내에서 생성된 항목들에 대해 사용됩니다. 이 시퀸스 번호는 64 비트
정수이기 때문에, 현실적으로는 같은 밀리초 안에서 생성할 수 있는 항목 수에 제한이 없습니다.

이러한 형식의 ID는 처음에는 이상해 보일 수 있지만, ID에 시간이 들어간 이유가 있습니다. 그 이유는 Redis 스트림은
ID로 범위 질의를 지원하기 때문입니다. ID가 항목이 생성된 시간에 관계되어 있기 때문에, 시간을 가지고 질의를 할 수 있습니다.
이것에 대해 추후 `XRANGE` 명령어를 살펴보면서 더 살펴보겠습니다.

여러 이유로 시간과 관련되지 않고, 실제로는 외부 시스템의 다른 ID와 연관된 증가하는 ID가 필요하다면, 앞서 언급했듯이
`XADD` 명령은 자동 생성을 트리거하는 * 와일드카드 ID 대신 명시적으로 ID를 생성할 수 있습니다. 다음의 예시에서
보겠습니다:
~~~redis
> XADD race:usa 0-1 racer Castilla
0-1
> XADD race:usa 0-2 racer Norem
0-2
~~~

이 사례에서는 최소 ID 0-1임을 주의하세요. 이전에 사용된 ID보다 작거나 동일한 ID는 허용되지 않습니다:
~~~redis
> XADD race:usa 0-1 racer Prickett
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
~~~

만약 Redis 7 버전 이후를 사용하신다면, 밀리초 시간 부분만 포함된 명시적 ID를 제공할 수도 있습니다. 이 경우
시퀸스 번호 부분은 자동으로 생성됩니다. 다음과 같은 문법으로 사용할 수 있습니다:
~~~redis
> XADD race:usa 0-* racer Prickett
0-3
~~~