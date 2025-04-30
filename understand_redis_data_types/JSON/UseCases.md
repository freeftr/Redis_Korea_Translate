# Use Cases

JSON의 사용 예시들

Redis의 기본 자료 구조형으로 JSON 객체들을 저장할 수 있고, 그것이 보통의 방빕입니다. 그 예시로
JSON을 직렬화한 후 Redis 문자열로 저장할 수 있습니다.

하지만, Redis JSON은 이러한 방식에 비해 여러 장점을 지닙니다.

## Access and retrieval of subvalues

JSON을 가지고, 네트워크를 통해 전체 객체를 전송하지 않고도 중첩된 값들을 가져올 수 있습니다.
하위 객체에 바로 접근할 수 있는 방식은 Redis에 큰 JSON 객체들을 저장하는 것보다 많은
효율을 가지고 있습니다. 

## Atomic partial updates

JSON은 배열의 값을 증가시키거나, 추가하거나, 삭제하거나, 문자열을 더하는 연산 등등을
원자적으로 수행할 수 있게 해줍니다. 같은 작업을 직렬화된 객체로 하기 위해서는 전체 객체를
조회하고, 재직렬화해야합니다. 이 작업은 비용이 크고, 원자성이 떨어집니다.

## Indexing and querying

Redis 문자열에 JSON 객체를 저장할 경우, 해당 객체들을 질의할 수 있는 좋은 방법은
없습니다. 반면에, JSON 객체들을 Redis Community Edition을 통해 JSON으로 
저장하는 방식을 통해 해당 객체들에 인덱스를 부여하고, 질의를 할 수 있습니다.
이러한 기능은 **Redis Query Engine**에 의해 제공됩니다.