# Hashes

Redis 해시는 필드-값 쌍으로 이루어진 컬렉션 형태의 레코드 타입입니다. 해시를 통해
기본 객체들을 나타내거나 여러 개의 카운터를 그룹으로 지정할 수 있습니다.

~~~redis
> HSET bike:1 model Deimos brand Ergonom type 'Enduro bikes' price 4972
(integer) 4
> HGET bike:1 model
"Deimos"
> HGET bike:1 price
"4972"
> HGETALL bike:1
1) "model"
2) "Deimos"
3) "brand"
4) "Ergonom"
5) "type"
6) "Enduro bikes"
7) "price"
8) "4972"
~~~

해시는 객체를 표현하는데 유용하지만, 실제로 해시에 넣을 수 있는 필드 수에는 (사용 가능한 메모리를 제외하면)
실질적인 제한이 없습니다. 따라서 애플리케이션 내에서 해시는 다양한 방법으로 응용될 수 있습니다.

**HSET** 명령어를 통해 해시의 여러 필드들을 설정할 수 있으며, **HGET** 명령어를 통해 단일 필드를 조회할 수 있습니다.
**HMGET** 명령어는 **HGET**과 비슷하지만, 값들의 배열을 반환합니다.

~~~redis
> HMGET bike:1 model price no-such-field
1) "Deimos"
2) "4972"
3) (nil)
~~~

**HINCRBY**처럼 개별 필드에 대해 연산을 할 수 있는 명령어들도 있습니다:

~~~redis
> HINCRBY bike:1 price 100
(integer) 5072
> HINCRBY bike:1 price -100
(integer) 4972
~~~

해시 명령어들의 전체 목록을 문서에서 확인할 수 있습니다.

소규모 해시는 메모리에서 특별한 방식으로 인코딩되어 매우 효율적으로 메모리를 사용할 수 있습니다.


## Basic commands

- **HSET**: 해시의 단일 또는 여러 필드의 값을 설정할 수 있습니다.
- **HGET**: 주어진 필드의 값을 반환합니다.
- **HMGET**: 주어진 단일 필드 또는 여러 필드의 값을 반환합니다
- **HINCRBY**: 주어진 필드의 값을 주어진 정수만큼 중가시킵니다.

해시 명령어들의 전체 목록을 확인하세요.

## Examples

- bike:1이 탑승된 횟수, 사고 난 횟수, 소유자가 변경된 횟수 등의 카운터를 저장하세요.

~~~redis
> HINCRBY bike:1:stats rides 1
(integer) 1
> HINCRBY bike:1:stats rides 1
(integer) 2
> HINCRBY bike:1:stats rides 1
(integer) 3
> HINCRBY bike:1:stats crashes 1
(integer) 1
> HGET bike:1:stats rides
"3"
> HMGET bike:1:stats owners crashes
1) "1"
2) "1"
~~~

## Field expiration

Redis 오픈 소스 7.4에서 새롭게 추가된 기능은 개별 해시 필드에 대해 만료 시간 또는 TTL(Time-To-Live)를 지정할 수 있다는 점입니다.
이 기능은 키 만료와 유사하며, 이에 해당하는 여러 명령어들도 함께 제공됩니다.

다음 명령어들을 사용하여 특정 필드에 대해 정확한 만료 시간 또는 TTL 값을 설정할 수 있습니다:
- **HEXPIRE**: 남은 TTL을 초 단위로 설정합니다.
- **HPEXPIRE**: 남은 TTL을 밀리초 단위로 설정합니다.
- **HEXPIREAT**: 만료 시간을 초 단위로 지정된 timestamp1으로 설정합니다.
- **HEPXPIREAT**: 만료 시간을 밀리초 단위로 지정된 timestamp1으로 설정합니다.

다음 명령어들을 사용하여 특정 필드가 언제 만료되는지 또는 TTL이 얼마나 남았는지 확인할 수 있습니다:
- **HEXPIRETIME**: 만료 시간을 초 단위의 timestamp로 가져옵니다.
- **HPEXPIRETIME**: 만료 시간을 밀리초 단위의  timestamp로 가져옵니다.
- **HTTL**: 남은 TTL을 초 단위로 가져옵니다.
- **HPTTL**: 남은 TTL을 밀리초 단위로 가져옵니다.

다음의 명령어로 특정 필드의 만료 시간을 제거하세요:
- **HPERSIST**: 만료 시간 제거

### Common field expiration use cases

1. Event Tracking: 해시 키를 사용하여 지난 한 시간 동안의 이벤트들을 저장하세요. 각 이벤트들의 TTL을 한 시간으로 설정하세요.
**HLEN** 명령어로 지난 한 시간동안의 이벤트들의 개수를 확인할 수 있습니다.
2. Fraud Detection: 이벤트에 대해 시간대별 카운터를 저장하는 해시를 생성하세요. 각 필드의 TTL을 48시간으로 설정하세요.
해시를 조회하여 지난 48시간 동안 시간대별 이벤트를 확인하세요.
3. Customer Session Management: 사용자의 데이터를 해시 키에 저장하세요. 각 세션 별로 새로운 해시 키를
생성하고 세션 필드를 각 사용자의 해시 키에 추가하세요. 세션이 만료되면 자동으로 사용자의 세션 키 와 세션 필드 모두 만료시키세요.
4. Active Session Tracking: 모든 활성 세션들을 세션 키에 저장하세요. 세션이 비활성화된 후 만료시키기 위해 각 세션들의 TTL을 설정하세요. 
**HLEN** 명령어를 통해 활성 세션들의 개수를 확인하세요.


### Field expiration examples

공식 클라이언트 라이브러리에서는 해시 필드 만료 기능을 아직 지원하지 않지만, Python (redis-py), Java (Jedis)
클라이언트 라이브러리의 베타 버전에서 해시 필드 만료 기능을 테스트 해볼 수 있습니다.

다음의 Python 예시들을 통해 필드 만료를 사용하는 방법을 보여줍니다.

다음과 같은 구조를 가진 센서 데이터를 저장하기 위한 해시 데이터셋을 가정해보세요:

```python
event = {
    'air_quality': 256,
    'battery_level': 89
}

r.hset('sensor', mapping=event)
```

아래 예시들에서는 sensor:sensor1 키의 필드들이 만료된 후, 해당 키를 새로 갱신해야 할 될것입니다.

해시의 여러 필드에 TTL을 설정하고 조회하세요:

~~~python
r.hexpire('sensor:sensor1', 60, 'air_quality', 'battery-level')
ttl = r.httl('sensor:sensor1', 'air_quality', 'battery_level')
print(ttl)
~~~

해시 필드를 밀리초 단위로 설정하고 조회하세요:

~~~python
r.hpexpire('sensor:sensor1', 60000, 'air_quality')
pttl = r.hpttl('sensor:sensor1', 'air_quality')
print(pttl)
~~~

해시 필드의 만료 시간을 timestamp로 조회하고 설정하세요:

~~~python
r.hexpireat('sensor:sensor1', 
    datetime.now() + timedelta(hours=24), 
    'air_quality')
expire_time = r.hexpiretime('sensor:sensor1', 'air_quality')
print(expire_time)
~~~

## Performance

대부분의 Redis 해시는 시간복잡도 O(1)을 가집니다.

**HKEYS**, **HVALS** **HGETALL**, 그리고 만료와 관련된 명령어들은 O(n)의 시간복잡도를 가지고
여기서 n은 필드-값 쌍의 개수입니다.

## Limits

모든 해시는 4,294,967,295 (2^32 - 1) 필드-값 쌍들을 저장할 수 있습니다.
실제로는, 해시의 크기는 Redis가 배포된 가상 머신(VM)의 전체 메모리 용량에 의해서만 제한됩니다.


## Learn more

- **Redis Hashes Explained**는 Redis 해시에 대한 이해를 돕기 위한 짧은 설명 영상입니다.
- **Redis University's RU101**은 Redis 해시에 대해 자세하게 다룹니다.

1. 모든 타임스탬프는 Unix 에폭 이후의 초 또는 밀리초 단위로 지정됩니다.







