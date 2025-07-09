# Redis Pub/Sub
Redis에서 Pub/Sub 채널을 사용하는 법

`SUBSCRIBE`, `UNSUBSCRIBE`, `PUBLISH`는 발행/구독 메시지 패러다임을 구현합니다. 발행자는 특정 수신자(구독자)에게 메시지를 보내도록 구현되지 않습니다.
대신, 발행된 메시지는 구독자에 대한 정보없이 채널을 통해 구분됩니다(느슨한 결합). 구독자들은 하나 이상의 채널에 대해 관심을 표현하고,
관심을 표현한 채널의 메시지만 수신합니다. 마찬가지로 구독자들은 발행자에 대한 그 어떤 정보도 들고 있지 않습니다. 이러한 발행자, 구독자 간의
디커플리은 높은 확장성과 보다 동적인 네트워크 토폴로지를 가능하게 합니다.

예를 들어, "channel11"과 "ch:00" 채널을 구독하려면, 클라이언트는 `SUBSCRIBE` 명령어와 채널 이름을 통해 구독을 합니다:

```redis
SUBSCRIBE channel11 ch:00
```

해당 채널로 다른 클라이언트들이 발신한 메시지들은 Redis에 의해 해당 채널을 구독하고 있는 모든 클라이언트들에게 발송됩니다. 구독자는
메시지가 발행된 순서대로 메시지를 수신합니다.

하나 이상의 채널을 구독 중인 클라이언트는 명령어를 실행해서는 안 되며, `SUBSCRIBE` 또는 `UNSUBSCRIBE`를 통해 다른 채널을 구독하거나 해지할
수는 있습니다. 구독 또는 해지에 대한 응답은 메시지 형태로 전송됩니다. 이때 전송되는 메시지는 클라이언트가 일관된 메시지 스트림을 읽을 수 있게 제공되며,
첫 번째 요소는 메시지의 유형을 가리킵니다. 구독된 RESP2 클라이언트의 문맥에서 허용되는 명령어들은 다음과 같습니다:
- `PING`
- `PSUBSCRIBE`
- `PUNSUBSCRIBE`
- `QUIT`
- `RESET`
- `SSUBSCRIBE`
- `SUBSCRIBE`
- `SUNSUNSCRIBE`
- `UNSUBSCRIBE`

하지만, 만약 RESP3가 사용되고 있으면 (`HELLO` 명령어 참조), 클라이언트는 구독 상태에서도 어떤 명령이든 실행할 수 있습니다.

주의점은, redis-cli를 사용할 때 구독 모드에 들어가면 `UNSUBSCRIBE`나 `PUNSUBSCRIBE`같은 명령을 사용할 수 없습니다. 이는 redis-cli가 
해당 모드에서는 어떤 명령도 입력받지 않기 때문이며, 구독 모드에서 나가려면 Ctrl-C를 통해 나갈 수 있습니다.

~~~TEXT
RESP는 Redis Serialization Protocol의 약자로, Redis 서버와 클라이언트 간 통신에 사용되는 직렬화 통신 프로토콜이다. 
주요 특징으로는 가볍고, 텍스트 기반으로 사람이 읽을 수 있는 형식이다. 이러한 특성으로 빠르게 파싱이 가능하다.

RESP2
- 기본값
- 데이터 타입 구분이 다소 불명확

RESP3
- HELLO 3 명령어를 통해 전환 가능
- 풍부한 데이터 표현 지원
- 구독 상태에서도 다른 명령 실행 가능
~~~

# Delivery semantics
***

Redis Pub/Sub은 at-most-once 메시지 시멘틱을 지원합니다. 말 그대로 메시지는 최대 한 번만 전달됩니다. Redis 서버가 메시지를 전송하고 나면,
해당 메시지가 다시 전송되는 일은 없습니다. 만약 구독자가 해당 메시지를 처리할 수 없으면 (예시로 오류 또는 네트워크 장애 상황), 메시지는 영원히
유실됩니다.

사용자의 애플리케이션이 강한 전송 보장을 요구한다면, **Redis Streams**에 대해 알아보는 것이 좋습니다. Streams에 있는 메시지들은 영속적이며,
at-most-once, at-least-once 모두 지원합니다.

# Format of pushed messages
***

메시지는 3개의 요소로 이루어진 배열-응답입니다.

첫 번째 요소는 메시지의 종류입니다:
- subscribe: 해당 응답의 두 번째 요소에 있는 채널에 성공적으로 구독했음을 나타냅니다. 세 번째 요소는 현재 구독하고 있는 채널의 개수입니다.
- unsubscribe: 해당 응답의 두 번째 요소에 있는 채널에 성공적으로 구독 해지했음을 나타냅니다. 세 번째 요소는 현재 구독하고 있는 채널의 개수입니다.
만약 마지막 요소가 0이면, 더 이상 어떤 채널도 구독하고 있지 않다는 뜻이며, 이때부터 클라이언트는 Pub/Sub 상태를 벗어나 Redis 모든 명령을 실행할 수 있습니다.
- message: 다른 클라이언트가 `PUBLISH` 명령어를 통해 발행한 메시지입니다. 두 번째 요소는 메시지를 보낸 채널 이름이고, 세 번째 요소는 실제 메시지 내용입니다(payload).

# Database & Scoping
***

Pub/Sub은 Redis의 키 공간과 관련이 없습니다. 데이터베이스 번호를 포함한 어떤 수준에서도 간섭하지 않게 구현되었습니다.

db 10에 발행해도 db 1에 있는 구독자가 청취할 수 있습니다.

어떤 형태로도 범위 지정이 필요하다면, 채널 이름 앞에 환경 이름(test, staging, production...)을 접두사로 붙이는 것이 좋습니다.

# Wire protocol example
***

~~~redis
SUBSCRIBE first second
*3
$9
subscribe
$5
first
:1
*3
$9
subscribe
$6
second
:2
~~~

이 시점에서, 다른 클라이언트로부터 second라는 이름의 채널에 대해 `PUBLISH` 연산을 수행합니다:

~~~redis
> PUBLISH second Hello
~~~

다음은 첫 번째 클라이언트가 수신하는 내용입니다:

~~~redis
*3
$7
message
$6
second
$5
Hello
~~~

이제 클라이언트는 구독중인 모든 채널에서 `UNSUBSCRIBE` 명령을 추가 인자 없이 사용하여 구독을 해지합니다:

~~~redis
UNSUBSCRIBE
*3
$11
unsubscribe
$6
second
:1
*3
$11
unsubscribe
$5
first
:0
~~~

# Pattern-matching subscriptions
***

Redis의 Pub/Sub 구현은 패턴 매칭을 지원합니다. 클라이언트는 glob 스타일 패턴에 구독할 수 있으며, 지정한 패턴과 일치하는 채널 이름으로
전송된 모든 메시지를 수신할 수 있습니다.

예를 들어:

~~~redis
PSUBSCRIBE news.*
~~~

위의 명령을 통해 news.art.figurative, news.music.jaz 등에 보내진 모든 메시지를 수신할 수 있습니다. 모든 glob 스타일 패턴이 유효하므로,
여러 개의 와일드 카드도 사용할 수 있습니다.

~~~redis
PUNSUBSCRIBE news.*
~~~

위의 명령을 통해 해당 패턴에 대해 패턴 구독을 해지하게 됩니다. 이 호출은 다른 구독에는 영향을 주지 않습니다.

패턴 매칭에 의해 수신된 메시지는 다른 형태로 전송됩니다:
- 메시지의 종류는 pmessage입니다: 다른 클라이언트의 `PUBLISH` 명령을 통해 보낸 메시지중 구독 중인 패턴과 일치한 채널에서 발생한 메시지를 
의미합니다. 두 번째 요소는 매칭된 원래의 패턴이고, 세 번째 요소는 매시지가 발행된 채널 이름, 마지막 요소는 실제 메시지 내용입니다.

`SUBSCRIBE`, `UNSUBSCRUBE`와 비슷하게 `PSUBSCRIBE`하고 `PUNSUNSCRIBE` 명령도 각각 psubscribe, punsubscribe 유형의 메시지를 통해
서버가 응답을 보냅니다. 이 응답은 일반 구독 메시지와 형식은 동일합니다.

# Messages matching both a pattern and a channel subscription
***

클라이언트가 패턴에 일치하는 여러 개의 패턴에 구독되어 있거나, 패턴과 채널 모두 구독되어 있으면 동일한 메시지를 여러 번 수신할 수 있습니다. 다음
예시는 이를 보여줍니다:

~~~redis
SUBSCRIBE foo
PSUBSCRIBE f*
~~~

위의 예시에서 만약 foo 채널에 메시지가 발행되면, 클라이언트는 두 개의 메시지를 수신할 것입니다: 타입 메시지 한 개와 pmessage를 수신할 것입니다.

# The meaning of the subscription count with pattern matching
***

Subscribe, unsubscribe, psubscribe, punsubscribe의 메시지 유형에서는 마지막 요소는 활성화된 구독의 개수입니다. 이 숫자는 구독중인 채널과 패턴 모두를
포함하는 숫자입니다. 따라서 클라이언트는 Pub/Sub 상태를 벗어나기 위해서는 이 숫자가 0으로 되기 위해 패턴과 채널 모두 구독 해지해야 합니다.

# Sharded Pub/Sub
***

Redis 7.0부터는 Sharded Pub/Sub이 도입되었습니다. 여기서 샤드 채널은 키를 슬롯에 할당하는 것과 동일한 알고리즘을 사용해 슬롯에 매핑됩니다.
샤드 메시지는 해당 채널이 해싱된 슬롯을 소유한 노드로 전송되어야 합니다. 클러스터는 해당 샤드 내의 모든 노드에 메시지가 전달되도록 보장합니다.
따라서 클라이언트는 슬롯을 담당하는 마스터 노드 또는 그 레플리카에 접속해 샤드 채널에 구독할 수 있습니다.
`SSUBSCRIBE`, `SUNSUBSCRIBE`, `SPUBLISH` 명령어는 Sharded Pub/Sub을 구현하기 위해 사용됩니다.

Sharded Pub/Sub은 클러스터 모드에서 Pub/Sub 사용을 확장하는 데 도움을 줍니다. 메시지 전파가 클러스터 내 특정 샤드로 제한되므로, 모든 메시지가
클러스터 내 모든 노드로 전파되는 전역 Pub/Sub 방식에 비해 클러스터 버스를 통해 전달되는 데이터 양이 줄어듭니다. 이를 통해 사용자는 샤드를 추가함으로써
Pub/Sub 사용량을 수평적으로 확장할 수 있게 됩니다.

# Programming example
***

Pieter Noordhuis께서 EventMachine과 Redis를 사용하여 다중 사용자 고성능 웹 채팅을 만드는 훌륭한 예제를 제공했습니다.

# Client library implementation hints
***

모든 수신된 메시지에는 해당 메시지를 수신하게 된 원래의 구독 정보가 포함되어 있기 때문에 클라이언트 라이브러리는 원래의 구독 정보를 해시 테이블을 사용해 콜백에 바인딩할 수 있습니다.

이렇게 하면 메시지를 수신했을 때 O(1) 시간 복잡도로 조회하여 해당 메시지를 등록된 콜백 함수에 즉시 전달할 수 있습니다. 