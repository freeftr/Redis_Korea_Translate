# Bulk loading
Redis 프로토콜을 사용하여 데이터를 벌크로 연산하세요.

Bulk loading은 Redis에 기존 데이터들을 대량으로 적재하는 과정입니다. 이 작업을 빠르고 효율적으로 하는 것은 매우 중요합니다. 해당 문서에서는
Redis에서 bulk loading 하는 전략들을 소개합니다.

## Bulk loading using the Redis protocol
***
Bulk loading에 일반 Redis 클라이언트를 사용하는 것은 여러 이유로 좋지 않습니다: 하나의 명령을 순차적으로 보내는 것은 매 명령어마다 네트워크를
왕복으로 타는 시간을 소모하기 때문입니다. 파이프라이닝을 사용할 수도 있지만, 많은 레코드를 bulk loading하기 위해서는 응답을 읽음과 동시에 
새로운 명령어를 작성할 수 있어야 가능한 빠르게 데이터를 삽입할 수 있습니다.

소수의 클라이언트만이 논블로킹 I/O를 지원하며, 모든 클라이언트가 처리량을 극대화하기 위해 응답을 효율적으로 파싱할 수 있는 것도 아닙니다. 이러한 이유들로,
Redis에 대량의 데이터를 삽입할 때 권장되는 방법은 필요한 명령어들을 포함한 Redis 프로토콜의 텍스트 파일을 생성하는 것입니다.

예를 들어, 'keyN -> ValueN' 형태의 수십억개의 키들로 이루어진 대용량 데이터셋을 생성하고자 한다면, 다음의 명령어들을 포함하는 Redis 프로토콜 형식의
파일을 만들 것입니다:
~~~redis
SET Key0 Value0
SET Key1 Value1
...
SET KeyN ValueN
~~~

해당 파일을 만든 후에, 이 파일을 Redis에 가능한 빨리 전달하면 됩니다. 과거에는 이를 위해 netcat 명령어를 사용했으며, 다음과 같습니다:
~~~redis
(cat data.txt; sleep 10) | nc localhost 6379 > /dev/null
~~~

> netcat
데이터를 읽고 쓰는데 사용하는 범용 유틸리티로써 TCP/UDP 연결을 생성하거나 수신할 수 있고, 파이프처럼 사용할 수 있어서 다양한 네트워크 작업에 활용된다.

하지만 위 방법은 대용량 데이터에 별로 권장되는 방법은 아닙니다. 그 이유는 netcat은 모든 데이터가 전송된 시점을 알 수 없고, 에러를 확인할 수도 없기
때문입니다. Redis 2.6 이상 버전에서는 redis-cli에서 bulk loading을 위한 새로운 pipe mode를 지원합니다.

Pipe mode를 사용하기 위한 명령어는 다음과 같습니다:
~~~redis
cat data.txt | redis-cli --pipe
~~~

이에 대한 응답은 다음과 같습니다:
~~~redis
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 1000000
~~~

Redis-cli는 Redis 인스턴스로부터 받은 리다이렉트 에러만을 표준 출력으로 출력하기를 보장합니다.

### Generating Redis Protocol
Redis 프로토콜은 매우 파싱하기도 쉽고, 생성하기도 쉽습니다. 그리고 이 [문서](https://redis.io/docs/latest/develop/reference/protocol-spec/)에 잘 기술되어 있습니다.
Bulk loading을 위해서 프로토콜을 생성하고자 한다면, 모든 세부 사항을 알 필요는 없습니다. 단지, 모든 명령어는 다음과 같이 표현된다는 것만 인지하면 됩니다:
~~~redis
*<args><cr><lf>
$<len><cr><lf>
<arg0><cr><lf>
<arg1><cr><lf>
...
<argN><cr><lf>
~~~

여기서 `<cr>`은 "\r" (또는 ASCII 문자 13)을 의미하고, `<lf>`는 "\n" (또는 ASCII 문자 10)을 의미합니다.

예시로, `SET key value`는 프로토콜에서 다음과 같이 표현됩니다:
~~~redis
*3<cr><lf>
$3<cr><lf>
SET<cr><lf>
$3<cr><lf>
key<cr><lf>
$5<cr><lf>
value<cr><lf>
~~~

또는 따옴표로 묶인 문자열로 표현하면 다음과 같습니다:
~~~redis
"*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n"
~~~

Bulk loading을 위해 생성해야 하는 파일은 위의 방식으로 표현된 명령어들을 하나씩 순차적으로 나열한 것으로 구성됩니다.

다음의 Ruby 함수는 유효한 프로토콜을 생성합니다:
~~~Ruby
def gen_redis_proto(*cmd)
    proto = ""
    proto << "*"+cmd.length.to_s+"\r\n"
    cmd.each{|arg|
        proto << "$"+arg.to_s.bytesize.to_s+"\r\n"
        proto << arg.to_s+"\r\n"
    }
    proto
end

puts gen_redis_proto("SET","mykey","Hello World!").inspect
~~~

위 함수로, 다음과 같은 프로그램으로 앞서 예시로 든 키-값 쌍들을 쉽게 생성할 수 있습니다:
~~~redis
(0...1000).each{|n|
    STDOUT.write(gen_redis_proto("SET","Key#{n}","Value#{n}"))
}
~~~

Redis-cli에 위 프로그램을 파이프로 직접 연결함으로써 첫 대용량 데이터 삽입을 진행할 수 있습니다.

### How the pipe mode works under the hood
중요한 점은 redis-cli 파이프 모드에서 netcat만큼 빠르고 언제 서버로부터 마지막 응답을 받았는지 파악할 수 있냐입니다.

이는 다음과 같은 원리로 가능합니다:
- `redis-cli --pipe`는 서버에 가능한 빨리 데이터를 보냅니다.
- 동시에 파싱을 시도하면서 데이터를 가능한 시점에 바로 읽습니다.
- 표준 입력에서 더 읽을 데이터가 없으면, `ECHO`라는 특수한 명령어를 무작위 20바이트 문자열과 함께 보냅니다: 이게 마지막 명령이라는 것을 인지할 수 있게 해주며,
벌크 응답으로 20바이트의 문자열을 받아 비교해볼 수 있습니다.
- 이 특수한 명령어가 보내지면, 응답을 받는 코드가 위에서 언급한 비교를 시작합니다. 동일하면 성공적으로 마치게 됩니다.

이러한 원리를 통해 프로토콜에서 얼마나 많은 명령어들을 보냈는지 셀 필요가 없고, 단순 응답만 확인하면 됩니다.

하지만 응답을 파싱하는 동안, 모든 응답의 개수를 카운팅하는 작업도 함께 수행하여, 최종적으로 이번 대량 삽입 세션을 통해 서버로 전송된 명령어의 총 개수를 사용자에게 알려줄 수 있습니다.
