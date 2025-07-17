# Time series
Redis로 시계열 데이터를 수집하고 질의하세요.

Redis 시계열 자료형은 실제 값 데이터 포인트들을 데이터를 수집한 시간과 함께 저장할 수 있게 해줍니다. 시계열 데이터 값을 통해
값, 시간으로 범위 질의를 할 수 있습니다. 또한 일정 기간 동안의 데이터에 대해 집계 함수를 작성하고 도출된 결과로부터 새로운 시계열 데이터를
만들 수 있습니다. 시계열 데이터를 만들 때는 마지막으로 보고된 타임스탬프를 기준으로 최대 보존 기간을 설정할 수 있어 데이터가 무한히
증식하는 것을 제한할 수 있습니다.

시계열 데이터는 매우 빠른 읽기, 쓰기 작업을 지원합니다. 그래서 다음과 같은 애플리케이션에 적절합니다:
- 계측 데이터 로깅
- 시스템 성능 지표
- 금융 시장 데이터
- IoT 센서 데이터
- 스마트 미터링
- QoS(Quality of Service) 모니터링

Redis 시계열 데이터는 Redis 오픈 소스, Redis Software, Redis Cloud에서 사용 가능합니다.
[**Install Redis Open Source**](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/),
[**Install Redis Enterprise**](https://redis.io/docs/latest/operate/rs/installing-upgrading/install/)에서 전체
설치 가이드를 확인할 수 있습니다.

# Create a time series
***
`Ts.CREATE` 명령어를 통해 명시적으로 키 이름을 지정하고, 새 빈 시계열을 생성할 수 있습니다. `TS.ADD`를 통해
키가 존재하지 않는 시계열에 데이터를 추가하면 자동으로 생성됩니다(`TS.ADD`에 더 알고 싶으시다면 아래 Adding data points를
참고하세요).
~~~redis
> TS.CREATE thermometer:1
OK
> TYPE thermometer:1
TSDB-TYPE
> TS.INFO therometer:1
 1) totalSamples
 2) (integer) 9
    .
    .
~~~

각 데이터 포인트의 타임스탬프는 64비트의 정수 값입니다. 각 값은 Unix epoch 이후의 시간을 밀리초 단위로 측정한 Unix 타임스탬프 값입니다.
시계열을 생성할 때는, 각 데이터 별로 마지막으로 보고된 타임스탬프를 기준으로 최대 보존 시간을 지정할 수 있습니다. 0으로 지정했다면,
영구적으로 보관됩니다.
~~~redis
# Create a new time series with a first value of 10.8 (Celsius),
# recorded at time 1, with a retention period of 100ms.
> TS.ADD thermometer:2 1 10.8 RETENTION 100
(integer) 1
> TS.INFO thermometer:2
    .
    .
 9) retentionTime
10) (integer) 100
    .
    .

~~~

시계열을 생성할 때 하나 이상의 라벨을 부여할 수도 있습니다. 라벨은 이름-값 쌍으로 둘 다 문자열 형태입니다. 이름, 값을 통해 질의나 집계를
위한 시계열의 부분집합을 선택할 수도 있습니다.
~~~redis
> TS.ADD thermometer:3 1 10.4 LABELS location UK type Mercury
(integer) 1
> TS.INFO thermometer:3
 1) totalSamples
 2) (integer) 1
 3) memoryUsage
 4) (integer) 5000
    .
    .
19) labels
20) 1) 1) "location"
       2) "UK"
    2) 1) "type"
       2) "Mercury"
    .
    .
~~~

# Add data points
***
`TS.ADD`를 통해 개별적인 데이터 포인트들을 추가할 수 있습니다. `TS.MADD`를 통해 여러 데이터 포인트들을 하나 이상의 시계열에 한 번의 호출로
추가할 수도 있습니다(`TS.ADD`, `TS.MADD`는 존재하지 않는 키를 명시해서 시계열 생성하면 시계열이 생성되지 않는다는 점에 주의하세요). 반환 값은
연산 후 각 시계열에 들어있는 샘플들의 개수의 배열입니다. `*`를 타임스탬프로 사용하면, Redis가 서버 시계를 바탕으로 현재 Unix 시간을 기록합니다.
~~~redis
> TS.MADD thermometer:1 1 9.2 thermometer:1 2 9.9 thermometer:2 2 10.3
1) (integer) 1
2) (integer) 2
3) (integer) 2
~~~

# Query data points
***
`TS.GET`을 사용해 시계열 데이터 중 가장 높은 타임스탬프를 가지고 있는 데이터 포인트를 조회하세요. 타임스탬프와 값 모두 반환합니다.
~~~redis
# The last recorded temperature for thermometer:2
# was 10.3 at time 2ms.
> TS.GET thermometer:2
1) (integer) 2
2) 10.3
~~~

`TS.RANGE`를 통해 주어진 타임스탬프 범위에 해당하는 시계열 데이터를 조회하세요. 포함 범위이며, 즉 범위의 시작과 끝이 모두 포함됩니다.
`-`, `+`를 통해 시계열 데이터들 내 최소부터 최대까지 조회가 가능합니다. 응답은 타임스탬프-값의 쌍들이며 타임스탬프순으로 오름차순 정렬됩니다.
내림차순으로 받기를 원하신다면, `TS.REVRANGE`를 조회하시면 됩니다.
~~~redis
# Add 5 data points to a time series named "rg:1".
> TS.CREATE rg:1
OK
> TS.MADD rg:1 0 18 rg:1 1 14 rg:1 2 22 rg:1 3 18 rg:1 4 24
1) (integer) 0
2) (integer) 1
3) (integer) 2
4) (integer) 3
5) (integer) 4

# Retrieve all the data points in ascending order.
> TS.RANGE rg:1 - +
1) 1) (integer) 0
   2) 18
2) 1) (integer) 1
   2) 14
3) 1) (integer) 2
   2) 22
4) 1) (integer) 3
   2) 18
5) 1) (integer) 4
   2) 24

# Retrieve data points up to time 1 (inclusive).
> TS.RANGE rg:1 - 1
1) 1) (integer) 0
   2) 18
2) 1) (integer) 1
   2) 14

# Retrieve data points from time 3 onwards.
> TS.RANGE rg:1 3 +
1) 1) (integer) 3
   2) 18
2) 1) (integer) 4
   2) 24

# Retrieve all the data points in descending order.
> TS.REVRANGE rg:1 - +
1) 1) (integer) 4
   2) 24
2) 1) (integer) 3
   2) 18
3) 1) (integer) 2
   2) 22
4) 1) (integer) 1
   2) 14
5) 1) (integer) 0
   2) 18

# Retrieve data points up to time 1 (inclusive), but
# return them in descending order.
> TS.REVRANGE rg:1 - 1
1) 1) (integer) 1
   2) 14
2) 1) (integer) 0
   2) 18
~~~

`TS.RANGE`, `TS.REVERANGE` 모두 결과를 필터링할 수 있게 지원합니다. 특정 타임스탬프 목록을 지정하면, 해당 정확한 타임스탬프를 가진 샘플만
결과에 포함됩니다(이 옵션을 사용해도 타임스탬프 범위는 반드시 지정해야 합니다).  최소, 최대 값을 지정해 그 사이 값들에 해당하는 조회도 가능합니다.
값을 범위로 잡아도 포함 범위로 작동하며, 단일 값을 조회하기 위해 최소, 최대를 같은 값으로도 설정할 수 있습니다.
~~~redis
> TS.RANGE rg:1 - + FILTER_BY_TS 0 2 4
1) 1) (integer) 0
   2) 18
2) 1) (integer) 2
   2) 22
3) 1) (integer) 4
   2) 24
> TS.REVRANGE rg:1 - + FILTER_BY_TS 0 2 4 FILTER_BY_VALUE 20 25
1) 1) (integer) 4
   2) 24
2) 1) (integer) 2
   2) 22
> TS.REVRANGE rg:1 - + FILTER_BY_TS 0 2 4 FILTER_BY_VALUE 22 22
1) 1) (integer) 2
   2) 22
~~~

## Query multiple time series
`TS.GET`, `TS.RANGE`, `TS.REVRANGE`는 여러 시계열 데이터에 연산을 할 수 있게 하는 `TS.MGET`, `TS.MRANGE`, `TS.MREVRANGE`
버전도 존재합니다. `TS.MGET`은 각 시계열별로 가장 높은 타임스탬프를 가진 데이터 포인트들을 반환하고, `TS.MRANGE`, `TS.MREVRANGE`는
각 시계열 별로 범위에 해당하는 데이터 포인트들을 가져옵니다.

사용되는 매개변수는 거의 비슷하며, 첫 번째 매개변수로 키 이름을 사용하지 않는다는 점이 다릅니다. 대신, 필터 표현식을 통해 특정 라벨을
가진 시계열만 조회하도록 합니다(Creating a time series에서 어떻게 시계열에 라벨을 추가하는지 확인하세요). 필터 표현식은 간단한 문법을 사용하며,
라벨의 존재 여부나 값을 기준으로 필터링할 수 있게 해줍니다. `TS.MGET` 페이지에서 필터 문법에 대한 자세한 사항을 확인하세요. 또한, 데이터 포인트를
반환할 때 모든 라벨을 포함하거나, 선택한 일부 라벨만 포함하도록 요청할 수도 있습니다.
~~~redis
# Create three new "rg:" time series (two in the US
# and one in the UK, with different units) and add some
# data points.
> TS.CREATE rg:2 LABELS location us unit cm
OK
> TS.CREATE rg:3 LABELS location us unit in
OK
> TS.CREATE rg:4 LABELS location uk unit mm
OK
> TS.MADD rg:2 0 1.8 rg:3 0 0.9 rg:4 0 25
1) (integer) 0
2) (integer) 0
3) (integer) 0
> TS.MADD rg:2 1 2.1 rg:3 1 0.77 rg:4 1 18
1) (integer) 1
2) (integer) 1
3) (integer) 1
> TS.MADD rg:2 2 2.3 rg:3 2 1.1 rg:4 2 21
1) (integer) 2
2) (integer) 2
3) (integer) 2
> TS.MADD rg:2 3 1.9 rg:3 3 0.81 rg:4 3 19
1) (integer) 3
2) (integer) 3
3) (integer) 3
> TS.MADD rg:2 4 1.78 rg:3 4 0.74 rg:4 4 23
1) (integer) 4
2) (integer) 4
3) (integer) 4

# Retrieve the last data point from each US time series. If
# you don't specify any labels, an empty array is returned
# for the labels.
> TS.MGET FILTER location=us
1) 1) "rg:2"
   2) (empty array)
   3) 1) (integer) 4
      2) 1.78
2) 1) "rg:3"
   2) (empty array)
   3) 1) (integer) 4
      2) 7.4E-1

# Retrieve the same data points, but include the `unit`
# label in the results.
> TS.MGET SELECTED_LABELS unit FILTER location=us
1) 1) "rg:2"
   2) 1) 1) "unit"
         2) "cm"
   3) 1) (integer) 4
      2) 1.78
2) 1) "rg:3"
   2) 1) 1) "unit"
         2) "in"
   3) 1) (integer) 4
      2) 7.4E-1

# Retrieve data points up to time 2 (inclusive) from all
# time series that use millimeters as the unit. Include all
# labels in the results.
> TS.MRANGE - 2 WITHLABELS FILTER unit=mm
1) 1) "rg:4"
   2) 1) 1) "location"
         2) "uk"
      2) 1) "unit"
         2) "mm"
   3) 1) 1) (integer) 0
         2) 25
      2) 1) (integer) 1
         2) 18
      3) 1) (integer) 2
         2) 21

# Retrieve data points from time 1 to time 3 (inclusive) from
# all time series that use centimeters or millimeters as the unit,
# but only return the `location` label. Return the results
# in descending order of timestamp.
> TS.MREVRANGE 1 3 SELECTED_LABELS location FILTER unit=(cm,mm)
1) 1) "rg:2"
   2) 1) 1) "location"
         2) "us"
   3) 1) 1) (integer) 3
         2) 1.9
      2) 1) (integer) 2
         2) 2.3
      3) 1) (integer) 1
         2) 2.1
2) 1) "rg:4"
   2) 1) 1) "location"
         2) "uk"
   3) 1) 1) (integer) 3
         2) 19
      2) 1) (integer) 2
         2) 21
      3) 1) (integer) 1
         2) 18
~~~

# Aggregation
***

시계열에 계속 데이터가 추가된다면 그 크기는 매우 커질 것입니다. 개별 샘플을 처리하는 것 대신, 전체 시간 범위를 동일한 크기의 버킷으로 나눈 후
각 버킷을 평균값이나 최대값 같은 집계 값으로 대표하는 것이 유용할 수 있습니다.

예를 들어, 하루에 10억 개가 넘는 데이터를 수집한다고 할때, 데이터를 1분 단위의 버킷으로 집계할 수 있습니다. 각 버킷이 하나의 값으로 표현되므로,
데이터셋의 크기는 1,440개의 데이터 포인트로 줄어들게 됩니다. (24시간 * 60분 = 1,440분)

범위 질의 명령어들은 집계 함수와 버킷 크기를 지정할 수 있습니다. 가능한 집계 함수는 다음과 같습니다:
- avg: 모든 값의 산술 평균
- sum: 모든 값의 합
- min: 최솟값
- max: 최댓값
- range: 가장 큰 값과 작은 값의 차이
- count: 값의 개수
- first: 버킷에서 가장 낮은 타임스탬프를 가진 값
- last: 버킷에서 가장 높은 타임스탬프를 가진 값
- std.p: 값들의 모표준편차
- std.s: 값들의 표본표준편차
- var.p: 값들의 모분산
- var.s: 값들의 표본분산
- twa: 버킷의 시간 구간에 대한 시간 가중 평균
  - RedisTimesSeries v1.8부터 지원

예시로, 아래의 예시는 `rg:2` 시계열의 다섯 개 데이터 포인트 전체에 대해 avg 함수를 사용한 집게를 보여줍니다. 버킷 크기는 2밀리초이므로,
3개의 집계된 값이 존재합니다. 마지막 버킷에서는 하나의 값만 사용되어 평균이 계산됩니다.
~~~redis
> TS.RANGE rg:2 - + AGGREGATION avg 2
1) 1) (integer) 0
   2) 1.9500000000000002
2) 1) (integer) 2
   2) 2.0999999999999996
3) 1) (integer) 4
   2) 1.78
~~~

## Bucket alignment
버킷들의 시퀸스는 참조 타임스탬프를 갖는데, 이는 시퀸스에서 첫 번째 버킷이 시작되는 시점을 나타냅니다. 기본적으로는 참조 타임스탬프는 0입니다.
예를 들어, 다음의 명령어들은 시계열을 만들고 25 밀리초의 버킷 크기로 min 집계를 기본 0 정렬로 적용합니다.
~~~redis
> TS.CREATE sensor3
OK
> TS.MADD sensor3 10 1000 sensor3 20 2000 sensor3 30 3000 sensor3 40 4000 sensor3 50 5000 sensor3 60 6000 sensor3 70 7000
1) (integer) 10
2) (integer) 20
3) (integer) 30
4) (integer) 40
5) (integer) 50
6) (integer) 60
7) (integer) 70
> TS.RANGE sensor3 10 70 AGGREGATION min 25
1) 1) (integer) 0
   2) 1000
2) 1) (integer) 25
   2) 3000
3) 1) (integer) 50
   2) 5000
~~~

아래의 다이어그램은 집계 버킷과 참조 타임스탬프 0 시점에 어떻게 정렬되는지를 보여줍니다.
~~~redis
Value:        |      (1000)     (2000)     (3000)     (4000)     (5000)     (6000)     (7000)
Timestamp:    |-------|10|-------|20|-------|30|-------|40|-------|50|-------|60|-------|70|--->

Bucket(25ms): |_________________________||_________________________||___________________________|
                           V                          V                           V
                  min(1000, 2000)=1000      min(3000, 4000)=3000     min(5000, 6000, 7000)=5000
~~~

질의 범위의 시작과 끝에 버킷을 정렬할 수도 있습니다. 예를 들어, 다음 명령어는 버킷을 쿼리 범위의 시작 지점인 10에 정렬합니다.
~~~redis
> TS.RANGE sensor3 10 70 AGGREGATION min 25 ALIGN start
1) 1) (integer) 10
   2) 1000
2) 1) (integer) 35
   2) 4000
3) 1) (integer) 60
   2) 6000
~~~

아래의 다이어그램은 버킷의 정렬을 보여줍니다.
~~~redis
Value:        |      (1000)     (2000)     (3000)     (4000)     (5000)     (6000)     (7000)
Timestamp:    |-------|10|-------|20|-------|30|-------|40|-------|50|-------|60|-------|70|--->

Bucket(25ms):          |__________________________||_________________________||___________________________|
                                    V                          V                           V
                        min(1000, 2000, 3000)=1000      min(4000, 5000)=4000     min(6000, 7000)=6000
~~~

## Aggregation across timeseries
기본적으로 `TS.MRANGE`와 `TS.MREVRANGE`의 결과는 시계열로 그룹화되어 있습니다. 하지만, **GROUPBY**, **REDUCE** 옵션을 통해 라벨을 기준으로
그룹화할 수 있습니다. 또한 같은 타임스탬프와 같은 라벨 값을 가진 요소들에 대해 집계를 적용할 수 있습니다 (이 기능은 RedisTimeSeries v1.6 이상에서
사용할 수 있습니다.).

예를 들어, 다음의 명령어들은 두 개는 UK 라벨, 두 개는 US 라벨로 4개의 시계열을 생성하고, 데이터 포인트들을 추가합니다. 첫 `TS.MRANGE` 명령어는 결과들을
나라 별로 그룹화하고, max 집계를 통해 각각의 나라, 타임스탬프에서 가장 큰 샘플 값을 찾아옵니다. 다음 `TS.MRANGE`는 같은 방식으로 그룹화하지만,
avg 집계를 적용합니다.
~~~redis
> TS.CREATE wind:1 LABELS country uk
OK
> TS.CREATE wind:2 LABELS country uk
OK
> TS.CREATE wind:3 LABELS country us
OK
> TS.CREATE wind:4 LABELS country us
OK
> TS.MADD wind:1 1 12 wind:2 1 18 wind:3 1 5 wind:4 1 20
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 1
> TS.MADD wind:1 2 14 wind:2 2 21 wind:3 2 4 wind:4 2 25
1) (integer) 2
2) (integer) 2
3) (integer) 2
4) (integer) 2
> TS.MADD wind:1 3 10 wind:2 3 24 wind:3 3 8 wind:4 3 18
1) (integer) 3
2) (integer) 3
3) (integer) 3
4) (integer) 3

# The result pairs contain the timestamp and the maximum sample value
# for the country at that timestamp. 
> TS.MRANGE - + FILTER country=(us,uk) GROUPBY country REDUCE max
1) 1) "country=uk"
   2) (empty array)
   3) 1) 1) (integer) 1
         2) 18
      2) 1) (integer) 2
         2) 21
      3) 1) (integer) 3
         2) 24
2) 1) "country=us"
   2) (empty array)
   3) 1) 1) (integer) 1
         2) 20
      2) 1) (integer) 2
         2) 25
      3) 1) (integer) 3
         2) 18

# The result pairs contain the timestamp and the average sample value
# for the country at that timestamp.
> TS.MRANGE - + FILTER country=(us,uk) GROUPBY country REDUCE avg
1) 1) "country=uk"
   2) (empty array)
   3) 1) 1) (integer) 1
         2) 15
      2) 1) (integer) 2
         2) 17.5
      3) 1) (integer) 3
         2) 17
2) 1) "country=us"
   2) (empty array)
   3) 1) 1) (integer) 1
         2) 12.5
      2) 1) (integer) 2
         2) 14.5
      3) 1) (integer) 3
         2) 13
~~~

# Compaction
***
집계 쿼리는 방대한 데이터 셋에서 작고 관리하기 쉬운 데이터셋으로 중요한 데이터들을 추출할 수 있게 해줍니다. 계속해서 시게열에 데이터를 추가한다면,
최신의 데이터에 대해서도 같은 집계를 반복해서 실행해야 할 수도 있습니다. 이때 매번 쿼리를 실행하는 것 대신, compaction rule을 시계열에 추가해
데이터가 도착할 때마다 집계를 점진적으로 계산할 수 있습니다. 집계된 값은 별도의 시계열에 저장되며, 원본은 유지됩니다.

`TS.CREATERULE`을 통해 compaction rule을 생성하고, 원본 시계열 키, 대상 시계열 키, 집계 함수, 버킷 지속 시간을 지정해야 합니다.
이때 대상 시계열을 compaction rule을 생성하기 전에 존재해야 합니다. 그 이유는 compaction은 규칙 생성 이후에 원본 시계열에 추가되는 데이터만
처리합니다.

예를 들어, 아래의 명령어들을 사용해 3밀리초마다 최솟값을 계산하는 compaction rule을 가진 시계열을 생성할 수 있습니다.
~~~redis
# The source time series.
> TS.CREATE hyg:1
OK
# The destination time series for the compacted data.
> TS.CREATE hyg:compacted
OK
# The compaction rule.
> TS.CREATERULE hyg:1 hyg:compacted AGGREGATION min 3
OK
> TS.INFO hyg:1
    .
    .
23) rules
24) 1) 1) "hyg:compacted"
       2) (integer) 3
       3) MIN
       4) (integer) 0
    .
    .
> TS.INFO hyg:compacted
    .
    .
21) sourceKey
22) "hyg:1"
    .
    .
~~~

처음 3밀리초 구간에서는 데이터 포인트들을 추가하더라도 compaction에는 아무 데이터도 생성되지 않습니다. 하지만 시간 4에 해당하는 데이터(즉, 두 번째 버킷)
를 추가하면, 컴팩션 규칙이 첫 번째 버킷에 대한 최솟값을 계산하고, 그 결과를 compaction에 추가합니다.
~~~redis
> TS.MADD hyg:1 0 75 hyg:1 1 77 hyg:1 2 78
1) (integer) 0
2) (integer) 1
3) (integer) 2
> ts.range hyg:compacted - +
(empty array)
> TS.ADD hyg:1 3 79
(integer) 3
> ts.range hyg:compacted - +
1) 1) (integer) 0
   2) 75
~~~

일반적인 방법은, compaction rule이 원본 시계열의 최신 버킷에 대해서는 데이터를 추가하지 않지만, 이전 버킷들에 대해서는 압축된 데이터를 추가하거나
갱신합니다. 이 방법은 데이터 샘플이 실시간으로 순차적으로 추가되는 사용 패턴을 반영한 것입니다 (버킷 기간이 끝나기 전까지는 집계가 정확하지 않기 때문).
주의할 점은, 뒤의 버킷에 데이터를 추가한다고 해서 이전 버킷이 닫히지는 않습니다. 최신의 버킷에 데이터를 추가하거나 삭제하더라도, compaction rule은
해당 버킷의 압축 데이터를 여전히 업데이트합니다.

# Delete data points
***
`TS.DEL` 명령어를 통해 주어진 타임스탬프 범위에 맞는 데이터 포인트들을 삭제할 수 있습니다. 포함 범위로 지원됩니다. 만약 단일 타임스탬프를 
삭제하고 싶으면, 범위의 시작과 끝을 해당 값으로 사용하세요.
~~~redis
> TS.INFO thermometer:1
 1) totalSamples
 2) (integer) 2
 3) memoryUsage
 4) (integer) 4856
 5) firstTimestamp
 6) (integer) 1
 7) lastTimestamp
 8) (integer) 2
    .
    .
> TS.ADD thermometer:1 3 9.7
(integer) 3
> TS.INFO thermometer:1
 1) totalSamples
 2) (integer) 3
 3) memoryUsage
 4) (integer) 4856
 5) firstTimestamp
 6) (integer) 1
 7) lastTimestamp
 8) (integer) 3
    .
    .
> TS.DEL thermometer:1 1 2
(integer) 2
> TS.INFO thermometer:1
 1) totalSamples
 2) (integer) 1
 3) memoryUsage
 4) (integer) 4856
 5) firstTimestamp
 6) (integer) 3
 7) lastTimestamp
 8) (integer) 3
    .
    .
> TS.DEL thermometer:1 3 3
(integer) 1
> TS.INFO thermometer:1
 1) totalSamples
 2) (integer) 0
    .
    .
~~~

# Use time series with other metrics tools
***
RedisTimeSeries Github organization에서는 RedisTimeSeries를 다른 도구들과 통합하는 데 도움이 되는 프로젝트들이 있습니다:
1. [Prometheus](https://github.com/RedisTimeSeries/prometheus-redistimeseries-adapter): RedisTimesSeries를 백엔드 데이터베이스로 사용하기 위한 읽기/쓰기 어댑터
2. [Grafana 7.1+](https://github.com/RedisTimeSeries/grafana-redis-datasource): Redis Data Source 사용
3. [Telegraf](https://github.com/influxdata/telegraf): InfluxData에서 플러그인을 다운로드하세요.
4. StatsD, Graphite: Graphite 프로토콜을 사용하는 내보내기 기능

# More information
***
이 섹션의 다른 페이지는 RedisTimeSeries의 개념을 자세하게 설명합니다. 또한 시계열 명령어 레퍼런스도 참고하세요.

[Configuration Parameters](https://redis.io/docs/latest/develop/data-types/timeseries/configuration/)
- Redis 시계열은 여러 설정 매개변수를 지원합니다.

[Out-of-order / backfilled ingestion performance considerations] (https://redis.io/docs/latest/develop/data-types/timeseries/out-of-order_performance_considerations/)
- 순서가 뒤섞인 데이터 / 과거 데이터 삽입 성능 고려사항

[Use cases](https://redis.io/docs/latest/develop/data-types/timeseries/use_cases/)
- 시계열 사용 사례