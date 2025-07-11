# Redis Vector sets
*Vector set은 최신 자료형이며 언제든 수정될 수 있습니다.

Vector sets는 sorted set과 유사한 자료형이지만, sorted set의 스코어대신 벡터를 문자열 형태로 갖는 요소들을 저장합니다.
Vector set을 통해 set에 항목을 추가할 수 있으며, 다음과 같은 일들을 수행할 수 있습니다:
- 특정 벡터와 유사한 항목들의 부분집합을 조회하거나
- 이미 Vector set에 존재하는 요소의 벡터와 가장 유사한 항목들의 부분집합을 조회할 수 있습니다.

Vector set은 필터 기반 검색도 지원합니다.(선택적) Vector set의 요소들에 속성을 연결할 수 있으며, **VSIM** 명령어의 FILTER 옵션을
통해 속성에 적용하는 간단한 수학적 필터를 통해 주어진 벡터와 유사한 항목들을 조회할 수 있습니다. 간단한 필터 예시입니다:".year > 1950".

Vector set에는 다음과 같은 명령어들이 존재합니다.
- **VADD** - Vector set에 요소를 추가하고, 만약 set이 존재하지 않을 시 새로운 set을 생성합니다.
- **VCARD** - Vector set의 요소들의 개수를 조회합니다.
- **VDIM** - Vector set에 포함된 벡터들의 차원 수를 조회합니다.
- **VEMB** - Vector set 요소와 연관된 근사 벡터를 조회합니다.
- **VGETATTR** - Vector set 요소들의 속성을 조회합니다.
- **VINFO** - Vector set의 메타데이터들과 내부 정보를 조회합니다.(크기, 차원, 양자화 방식, 그래프 구조)
- **VRANDMEMBER** - Vector set의 요소를 무작위하게 조회합니다.
- **VREM** - Vector set에서 요소를 제거합니다.
- **VSETATTR** - Vector set 요소를 수정하거나 설정합니다.
- **VISM** - 선택적 필터링과 함께 주어진 벡터에 유사한 요소들을 조회합니다.

## Examples
***
다음의 예시는 Vector set의 사용법을 보여주기 위한 예시입니다. 이해를 돕기 위해 2차원 벡터를 사용할 것입니다.
하지만 실제 사용 사레에서는 벡터가 보통 텍스트 임베딩을 나타내며, 수백 차원의 벡터를 사용합니다.
텍스트 임베딩 활용에 대한 자세한 내용은 Redis for AI 문서를 참조하세요.
(해당 예시는 아래에 공식문서 링크를 남기도록 하겠습니다.)
- https://redis.io/docs/latest/develop/data-types/vector-sets/

### Basic operations
`VADD` 명령어를 통해 포인트 벡터를 추가하는 것부터 시작해봅시다. Vector set 객체는 자동으로 생성됩니다. 객체가 생성된
후에는 `TYPE` 명령을 사용하여 해당 객체의 타입의 vector set인지 확인할 수 있습니다.

```redis
> VADD points VALUES 2 1.0 1.0 pt:A
(integer) 1
> VADD points VALUES 2 -1.0 -1.0 pt:B
(integer) 1
> VADD points VALUES 2 -1.0 1.0 pt:C
(integer) 1
> VADD points VALUES 2 1.0 -1.0 pt:D
(integer) 1
> VADD points VALUES 2 1.0 0 pt:E
(integer) 1
> TYPE points
vectorset
```

`VCARD` 명령어를 통해 set에 들어있는 요소의 개수를 알아보고(==set의 기수성), `VDIM` 명령어를 통해 차원의 수를 조회해보세요:

```redis
> VCARD points
(integer) 5
> VDIM points
(integer) 2
```

`VEMD` 명령을 사용하여 vector set에 있는 요소들의 좌표 값을 조회하세요. 다만, 벡터를 추가할 때 성능을 위해 양자화가 적용되므로, 
일반적으로 정확한 값이 아닐 수 있음에 주의하세요.

```redis
> VEMB points pt:A
1) "0.9999999403953552"
2) "0.9999999403953552"
9> VEMB points pt:B
1) "-0.9999999403953552"
2) "-0.9999999403953552"
> VEMB points pt:C
1) "-0.9999999403953552"
2) "0.9999999403953552"
> VEMB points pt:D
1) "0.9999999403953552"
2) "-0.9999999403953552"
> VEMB points pt:E
1) "1"
2) "0"
```

`VSETATTR` 명령어와 `VGETATTR` 명령어를 통해 JSON 형태의 속성 데이터를 설정하고 조회할 수 있습니다. 또한, `VSETATTR`에 빈 문자열을 전달하면
해당 요소의 속성 데이터를 삭제할 수 있습니다.

~~~redis
> VSETATTR points pt:A "{\"name\": \"Point A\", \"description\": \"First point added\"}" 
(integer) 1
> VGETATTR points pt:A
"{\"name\": \"Point A\", \"description\": \"First point added\"}"
> VSETATTR points pt:A "" 
(integer) 1
> VGETATTR points pt:A
(nil)
~~~

`VREM`을 통해 원하지 않는 요소를 삭제하세요.

~~~redis
> VADD points VALUES 2 0 0 pt:F
(integer) 1
127.0.0.1:6379> VCARD points
(integer) 6
127.0.0.1:6379> VREM points pt:F
(integer) 1
127.0.0.1:6379> VCARD points
(integer) 5
~~~

### Vector similarity search

샘플 점으로부터의 벡터 거리 순서대로 점들을 정렬하기 위해 `VSIM`을 사용하세요.

~~~redis
> VSIM points values 2 0.9 0.1
1) "pt:E"
2) "pt:A"
3) "pt:D"
4) "pt:C"
5) "pt:B"
~~~

점 A에 가장 가까운 4 요소들을 찾고 해당 요소들의 거리 점수를 찾으세요:

~~~redis
> VSIM points ELE pt:A WITHSCORES COUNT 4
1) "pt:A"
2) "1"
3) "pt:E"
4) "0.8535534143447876"
5) "pt:C"
6) "0.5"
7) "pt:D"
8) "0.5"
~~~

JSON 속성을 추가하고 필터 표현식을 통해 검색에 활용하세요:

~~~redis
> VSETATTR points pt:A "{\"size\":\"large\",\"price\": 18.99}"
(integer) 1
> VSETATTR points pt:B "{\"size\":\"large\",\"price\": 35.99}"
(integer) 1
> VSETATTR points pt:C "{\"size\":\"large\",\"price\": 25.99}"
(integer) 1
> VSETATTR points pt:D "{\"size\":\"small\",\"price\": 21.00}"
(integer) 1
> VSETATTR points pt:E "{\"size\":\"small\",\"price\": 17.75}"
(integer) 1

# Return elements in order of distance from point A whose
# `size` attribute is `large`.
> VSIM points ELE pt:A FILTER '.size == "large"'
1) "pt:A"
2) "pt:C"
3) "pt:B"

# Return elements in order of distance from point A whose size is
# `large` and whose price is greater than 20.00.
> VSIM points ELE pt:A FILTER '.size == "large" && .price > 20.00'
1) "pt:C"
2) "pt:B"
~~~

## More information
***
이 섹션의 다른 페이지들을 참고하여 vector set의 기능과 성능 관련 파라미터에 대해 더 자세히 알아보세요.
- Filter expressions: Redis vector set에서 유사도 검색 결과를 필터링하기 위해 필터 표현식을 사용하는 방법을 알아봅니다.
- Performance: Redis vector set이 부하 상황에서 어떻게 동작하는지, 속도와 검색 정확도를 최적화하는 방법을 알아봅니다.
- Scalability: 더 큰 데이터셋과 워크로드를 처리할 수 있도록 Redis vector set을 확장하는 방법을 배웁니다.
- Memory Optimization: Redis vector set에서 메모리 사용량을 최적화하는 방법을 학습합니다.
- Troubleshooting: Redis vector set을 사용할 때 발생할 수 있는 문제를 진단하고 디버깅하는 방법을 배웁니다.
