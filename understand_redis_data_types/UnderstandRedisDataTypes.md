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
Redis lists는 삽입 순서대로 정렬된 string들의 배열입니다. 다음의 부분에서 추가적인 내용을 확인할 수 있습니다.
- Overview of Redis lists
- Redis list command reference


### Sets
Redis sets는 Java의 HashSets, Python의 sets와 같이 정렬되지 않은 유일무이한 문자열들의 집합입니다. Redis set을 가지고 
시간복잡도 O(1)에 add, remove, 그리고 데이터의 존재 유무까지 확인할 수 있습니다. (다른 말로 집합의 데이터 수에 상관없이 가능합니다.)
다음의 부분에서 추가적인 내용을 확인할 수 있습니다.

### Hashes
Redis Hashes는 field-value 쌍으로 구성된 집합입니다. 비슷한 예로, Python의 dictionaries, Java의 HashMaps, 그리고 Ruby의 
hashes가 있습니다. 다음의 부분에서 추가적인 내용을 확인할 수 있습니다.
- Overview of Redis hashes
- Redis hashes command reference

### Sorted sets
Redis sorted sets는 유일무이한 string들로 이루어진 집합이며, 이는 각 string에 연결된 점수(associated score)를 기준으로 정렬된 상태를 유지합니다.
(만약 점수가 같을 경우 사전순으로 정렬됩니다.) 다음의 부분에서 추가적인 내용을 확인할 수 있습니다.
- Overview of Redis sorted sets
- Redis sorted set command reference

### Streams
Redis stream은 append-only 로그처럼 작동하는 자료구조입니다. Stream은 이벤트들을 발생한 순서대로 기록하고, 모아서 처리하는데 
사용됩니다. 다음의 부분에서 추가적인 내용을 확인할 수 있습니다.
- Overview of Redis Streams
- Redis Streams command reference

*Append-only Log
말 그대로 데이터의 오로지 추가만 가능한 자료구조입니다. 선형 구조로 끝에 새로운 데이터를 추가하는 방식으로 작동됩니다. 이러한 특성 때문에
불변성을 지니며, insert만 하는 쓰기 작업만 존재하기 때문에 효율성을 가집니다. 또한 모든 데이터가 순서대로 기록되기 때문에 복구가 용이합니다.
이벤트 기록용 큐로서 Kafka 등에 사용되고 있습니다. 하지만 데이터의 추가만 존재하기 때문에 필요한 storage가 크다는 단점이 있습니다.

### Geospatial Indexes
Redis Geospatial Index는 주어진 반경이나 bounding box를 이용해 위치를 찾아내는데 유용한 자료구조입니다. 다음의 부분에서 추가적인 내용을 확인할 수 있습니다.
- Overview of Redis Streams
- Redis Streams command reference
