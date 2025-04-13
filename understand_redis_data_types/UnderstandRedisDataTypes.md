# Redis의 자료형들

## Core Data Types
***
Redis Community Edition은 다음의 자료형들을 지원합니다.
- String
- Hash
- List
- Set
- Sorted Set
- Stream
- Bitmap
- Bitfield
- Geospatial

Redis Community Edition과 Redis Enterprise Edition에는 JSON과 같은 다른 유용한 자료형들을 지원하는
확장 모듈들도 포함하고 있습니다. 아래의 Extension data types에서 그 내용을 볼 수 있습니다.

### Strings
Redis strings은 가장 기본적인 Redis 자료형이며, 이는 연속적인 바이트들로 표현됩니다. 다음의 부분에서 추가적인 내용을 
확인할 수 있습니다.
- Overview of Redis strings
- Redis string command reference

### Lists
Redis lists는 삽입 순서대로 정렬된 string들의 배열입니다. 더 자세한 내용은 다음을 확인하세요.
- Overview of Redis lists
- Redis list command reference


### Sets
Redis sets는 Java의 HashSets, Python의 sets와 같이 정렬되지 않은 유일무이한 문자열들의 집합입니다. Redis set을 가지고 
시간복잡도 O(1)에 add, remove, 그리고 데이터의 존재 유무까지 확인할 수 있습니다. (다른 말로 집합의 데이터 수에 상관없이 가능합니다.)
더 자세한 내용은 다음을 확인하세요.

### Hashes
Redis Hashes는 field-value 쌍으로 구성된 집합입니다. 비슷한 예로, Python의 dictionaries, Java의 HashMaps, 그리고 Ruby의 
hashes가 있습니다. 더 자세한 내용은 다음을 확인하세요.
- Overview of Redis hashes
- Redis hashes command reference

### Sorted sets
Redis sorted sets는 유일무이한 string들로 이루어진 집합이며, 이는 각 string에 연결된 점수(associated score)를 기준으로 정렬된 상태를 유지합니다.
(만약 점수가 같을 경우 사전순으로 정렬됩니다.) 더 자세한 내용은 다음을 확인하세요.
- Overview of Redis sorted sets
- Redis sorted set command reference

### Streams
Redis stream은 append-only 로그처럼 작동하는 자료구조입니다. Stream은 이벤트들을 발생한 순서대로 기록하고, 모아서 처리하는데 
사용됩니다. 더 자세한 내용은 다음을 확인하세요.
- Overview of Redis Streams
- Redis Streams command reference

*Append-only Log
말 그대로 데이터의 오로지 추가만 가능한 자료구조입니다. 선형 구조로 끝에 새로운 데이터를 추가하는 방식으로 작동됩니다. 이러한 특성 때문에
불변성을 지니며, insert만 하는 쓰기 작업만 존재하기 때문에 효율성을 가집니다. 또한 모든 데이터가 순서대로 기록되기 때문에 복구가 용이합니다.
이벤트 기록용 큐로서 Kafka 등에 사용되고 있습니다. 하지만 데이터의 추가만 존재하기 때문에 필요한 storage가 크다는 단점이 있습니다.

### Geospatial Indexes
Redis Geospatial Index는 주어진 반경이나 bounding box를 이용해 위치를 찾아내는데 유용한 자료구조입니다. 더 자세한 내용은 다음을 확인하세요.
- Overview of Redis Streams
- Redis Streams command reference

### Bitmaps
Redis Bitmap은 비트 연산을 가능하게 해줍니다. (문자열로 비트연산이 가능함.)
더 자세한 내용은 다음을 확인하세요.
- Overview of Redis bitmaps
- Redis bitmap command reference

### Bitfields
Redis bitfields는 하나의 문자열 값에 여러 개의 카운터를 효율적으로 인코딩할 수 있게 해줍니다.
Bitfield는 원자적인(get, set, increment) 연산을 지원하며, 다양한 오버플로우(overflow) 정책을 지원합니다.
더 자세한 내용은 다음을 확인하세요.
- Overview of Redis bitfields
- The BITFIELD command

## Extension data types
***
Redis Community Edition과 Redis Enterprise 버전에서는 다음과 같은 자료형들을 지원하는 확장형 모듈을 
포합합니다 :
- JSON
- Probabilistic 자료형 (확률에 기반한 자료형으로 빠른 계산과 적은 메모리를 위해 근삿값을 사용하는 자료형이라고 한다.)
- 시계열
이 자료형들은 Redis Community Edition에 기본으로 포함되지는 않는 항목들입니다. Redis Community Edition에서 지원하는
자료형들을 보고 싶으시면 Core data types를 확인하세요.

### JSON
Redis JSON은 익히 알고 있는 JSON 형식과 같게 구조적이고, 계층적인 배열들과 키-값 객체들을 제공합니다. 마찬가지로 JSON을 Redis로
가져올 수 있습니다. 또한, 개별 데이터들을 접근하고, 수정하고, 조회할 수 있습니다.
더 자세한 내용은 다음을 확인하세요.
- Overview of Redis JSON
- JSON command reference

### Probabilistic data types
이 자료형들은 근삿값이지만 매우 효율적으로 통계를 수집하고 계산할 수 있게 해줍니다. 다음과 같은 자료형들이 가능합니다.
- HyperLogLog
- Bloom filter
- Cuckoo filter
- t-digest
- Top-K
- Count-min sketch

### HyperLogLog
Redis HyperLogLog는 큰 집합들의 기수성을 추정치로 계산할 수 있게 해줍니다.
더 자세한 내용은 다음을 확인하세요.
- Overview of Redis HyperLogLog
- Redis HyperLogLog command reference

### Bloom filter
Redis Bloom filter는 집합에서 요소의 존재 여부를 확인할 수 있게 해줍니다.
더 자세한 내용은 다음을 확인하세요.
- Overview of Redis Bloom filters
- Bloom filter command reference

### Cuckoo filter
Redis Cuckoo filters는 집합에서 요소의 존재 여부를 확인할 수 있게 해줍니다. Bloom filter와 비슷하지만
기능과 성능적인 측면에서 사소한 trade-off 차이를 보입니다.
더 자세한 내용은 다음을 확인하세요.
- Overview of Redis Cuckoo filters
- Cuckoo filter command reference

### t-digest
Redis t-digest는 데이터 스트림에서 백분율 추정치를 계산할 수 있습니다.
더 자세한 내용은 다음을 확인하세요.
- Redis t-digest overview
- t-digest command reference

### Top-K
Redis Top-K 구조는 데이터 값 스트림 내에서 데이터 포인트의 순위를 측정할 수 있게 해줍니다.
더 자세한 내용은 다음을 확인하세요.
- Redis Top-K overview
- Top-K command reference

### Count-min sketch
Redis Count-min sketch는 데이터 값 스트림 내에서 데이터 포인트의 빈도를 추정해 줍니다.
더 자세한 내용은 다음을 확인하세요.
- Redis Count-min sketch overview
- Count-min sketch command reference

## Time series
***
Redis Times series 구조는 타임스탬프과 포함된 데이터 포인트를 저장하고 조회할 수 있게 해줍니다.
더 자세한 내용은 다음을 확인하세요.
- Redis time series overview
- Count-min sketch command reference

## Adding extensions
***
기본 제공되는 데이터 타입의 기능을 확장하려면 다음 옵션 중 하나를 사용할 수 있습니다:
- Lua로 직접 커스텀 server-side 함수들을 작성하기.
- Modules API를 이용해 직접 Redis 모듈을 개발하거나, 커뮤니티에서 제공되는 모듈을 사용하기.
- Redis Community Edition에서 제공하는 JSON, 검색, 시계열 등 다양한 기능을 활용하기.

