# Configuration Parameters
Redis 시계열은 여러 설정 파라미터를 지원합니다.

# Redis Open Source - set configuration parameters
***
Redis 오픈 소스의 Redis 8버전 이전에는, 모든 시계열 구성 파라미터가 load-time 파라미터였습니다. load-time 설정
파라미터 값을 사용하려면 다음 중 하나를 선택하면 됩니다:
- Redis 서버 시작 시 loadmodule 인자 다음에 다음 명령어를 실행하세요:
    `redis-server --loadmodule ./{modulename}.so [OPT VAL]...`
- 설정 파일의 loadmodule 지시어에 다음을 인자로 추가하세요 (ex: redis.conf):
    `loadmodule ./{modulename}.so [OPT VAL]...`
- `MODULE LOAD path [arg [arg ...]]` 명령어 사용
- `MODULE LOADEX path [CONFIG name value [CONFIG name value ...]] [ARGS args [args ....]]` 명령어 사용

Redis 8.0 부터 대부분의 시계열 설정 파라미터들은 runtime 파라미터입니다. Runtime 매개면수는 load time에도
설정할 수 있지만, Redis `CONFIG` 명령어를 활용하는 것이 훨씬 쉽고 Redis runtime 설정 파라미터와 같이 작동합니다.

이것은 다음을 의미합니다:
- `CONFIG SET parameter value [parameter value ...]`
  - 하나 이상의 설정 파라미터를 설정합니다.
- `CONFIG SET parameter [parameter ...]`
  - 하나 이상의 파라미터에 대한 현재 값을 조회합니다.
- `CONFIG REWRITE`
  - 설정 변경을 반영하기 위해 Redis 설정 파일을 다시 작성합니다 (ex: redis.conf).

Redis 8.0부터는 Redis 구성 파라미터와 동일한 방식으로, 시계열 설정 파라미터를 Redis 설정 파일에 직접적으로 명시할 수 있습니다. 

`CONFIG SET`을 통해 값을 설정하거나, 구성 파일을 통해 수동으로 추가되면, 해당 값은 `--loadmodule`, `module`,
`MODULE LOAD`, 또는 `MODULE LOADEX`를 통해 덮어 씌워집니다.

Redis 8.0에서는 Redis 설정 파라미터에 맞추기 위해 시계열 설정 파라미터에 새로운 이름이 추가되었습니다. `CONFIG`를 사용할 
경우 반드시 새로운 이름을 사용해야 합니다.

# Time series configuration paramters
***
| 파라미터 이름 Name (< 8.0) | 파라미터 이름 (≥ 8.0)         | Run-time | Redis Software | Redis Cloud         |
|----------------------|-------------------------|----------|----------------|----------------------|
| CHUNK_SIZE_BYTES     | ts-chunk-size-bytes     | ✅       | ✅ Supported   | ✅ Flexible & Annual <br> ❌ Free & Fixed |
| COMPACTION_POLICY    | ts-compaction-policy    | ✅       | ✅ Supported   | ✅ Flexible & Annual <br> ❌ Free & Fixed |
| DUPLICATE_POLICY     | ts-duplicate-policy     | ✅       | ✅ Supported   | ✅ Flexible & Annual <br> ❌ Free & Fixed |
| RETENTION_POLICY     | ts-retention-policy     | ✅       | ✅ Supported   | ✅ Flexible & Annual <br> ❌ Free & Fixed |
| ENCODING             | ts-encoding             | ✅       | ✅ Supported   | ✅ Flexible & Annual <br> ❌ Free & Fixed |
| IGNORE_MAX_TIME_DIFF | ts-ignore-max-time-diff | ✅       |                |                      |
| IGNORE_MAX_VAL_DIFF  | ts-ignore-max-val-diff  | ✅       |                |                      |
| NUM_THREADS          | ts-num-threads          | ⬜       | ✅ Supported   | ❌ Flexible & Annual <br> ❌ Free & Fixed |
| OSS_GLOBAL_PASSWORD  | Deprecated in v8.0      | ✅       |                |                      |

> Redis 시계열에서의 Chunk
- 시계열 데이터가 메모리에 저장되는 단위 블록
- Redis 시계열 데이터는 데이터를 한 덩어리에 저장하지 않고 일정 크기의 청크로 쪼개서 저장한다.

## CHUNK_SIZE_BYTES / ts-chunk-size-bytes
새로운 청크의 데이터 부분에 초기 할당되는 크기는 바이트 단위입니다. 실제 청크들은 더 많은 메모리가 소비될 수 있습니다.
이 값을 변경해도 기존 청크들에는 영향을 주지 않습니다.

Type: integer

Valid range: [48 .. 1048576]; 8의 배수이어야 합니다.

### Precedence order
청크 크기가 다른 수준으로 설정될 수 있기 때문에, 실제 청크 크기의 우선순위는 다음과 같습니다:
1. Key-level 정책: `TS.CREATE` 또는 `TS.ALTER`의 CHUNK_SIZE 인자로 설정된 값
2. `ts-chunk-size-bytes` 설정 파라미터
3. 하드코딩된 기본값: 4096

### Example
기본 청크 크기를 1024바이트로 설정하세요:

8.0 버전 이하:
~~~redis
$ redis-server --loadmodule ./redistimeseries.so CHUNK_SIZE_BYTES 1024
~~~

8.0 버전 이상:
~~~redis
redis> CONFIG SET ts-chunk-size-bytes 1024
~~~

## COMPACTION_POLICY / ts-compaction-policy
`TS.ADD`, `TS.INCRBY`, `TS.DECRBY`를 통해 생성된 새로운 키들의 기본 compaction rules

Type: string

이 설정 파라미터는 `TS.CREATE`를 통해 생성한 키들에 영향을 주지 않는 점을 명시하세요. 이해하기 위해서 다음 시나리오를
생각해보세요: 기본 compaction rule을 설정한 후 수동으로 추가 compaction rule (`TS.CREATERULE`을 사용해서)을
만들고 싶다고 가정합니다. 이때 먼저 비어 있는 대상 키를 생성해야 합니다. 하지만 이러한 접근 방식에는 문제점이 있습니다:
기본 compaction 정책은 Redis가 대상 키에 대해 원치 않는 자동 압축을 생성할 수 있습니다.

각 규칙은 세미콜론을 통해 구분됩니다 (;). 각 규칙은 콜론 (:)을 통해 구분되는 여러 필드를 포함하고 있습니다:
- 집계 형식: 다음과 같음:

| Aggregator | Description                 |
|------------|-----------------------------|
| avg        | 모든 값의 산술 평균                 |
| sum        | 모든 값의 합계                    |
| min        | 최소값                         |
| max        | 최대값                         |
| range      | 최대값과 최소값의 차이                |
| count      | 값의 개수                       |
| first      | 해당 버킷 내에서 가장 이른 타임스탬프를 가진 값 |
| last       | 해당 버킷 내에서 가장 늦은 타임스탬프를 가진 값 |
| std.p      | 모집단 표준편차                    |
| std.s      | 표본 표준편차                     |
| var.p      | 모집단 분산                      |
| var.s      | 표본 분산                       |
| twa        | 모든 값의 시간 가중 평균 (v1.8부터 지원)  |

- 각 시간 버킷의 지속 시간: 숫자 + 시간 단위 형식 (1분에 대한 예시: 1M, 60s 또는 60000m)
  - m - 밀리초
  - s - 초
  - M - 분
  - h - 시간
  - d - 일
- 보존 시간: 숫자 + 시간 단위 형식 (1분에 대한 예시: 1M, 60s, 또는 60000m)
  - m - 밀리초
  - s - 초
  - M - 분
  - h - 시간
  - d - 일
  - 0m, 0s, 0M, 0h, 0d는 만료가 없다는 뜻입니다.
- v1.8부터:
  - 선택 사항: 타임 버킷 정렬 기준 - 숫자 + 시간 단위 형식 (1분에 대한 예시: 1M, 60s 또는 60000m)
    - m - 밀리초
    - s - 초
    - M - 분
    - h - 시간
    - d - 일

`alignTimestamp`를 설정하면 Epoch 이후 해당 시점에서 시작되는 첫 번째 버킷이 생성되며, 모든 다른 버킷도 그에 맞춰
정렬됩니다. 기본값: 0. 예시: 만약 `bucketDuration`이 24시간이면, `alignTimestamp`를 6시간으로 설정하면,
각 버킷의 시간 구간은 [06:00 .. 다음 날 06:00) 형태가 됩니다.

> 주의사항:
클러스터 환경에서는, 이 설정 파라미터를 적용하면, 모든 시계열 키 이름에 해시 태그를 사용해야 합니다. 이것을 통해 Redis가
키를 통해 각 해시 슬롯에 압축을 생성하게 됩니다. 만약 사용하지 않으면 그 어떤 에러도 출력하지 않은 채 압축에 실패할 것입니다.

압축 정책이 정해진 후, 새로 생성된 시계열에 대해 compaction rule이 생성되게 됩니다. 그리고 compaction 키 이름은
다음과 같습니다:
- 타임 버킷 정렬이 0이면:
  - `key_agg_our`  형식에서 `key`는 소스 시계열의 키, `agg`는 집계 함수(대문자), `dur`는 버킷 지속 시간(밀리초)입니다.
  - 예시: `key_SUM_60000`
- 타임 버킷 정렬이 0이 아니면:
  - `key_agg_dur_aln` 형식에서 `key`는 소스 시계열의 키, `agg`는 집계 함수(대문자), `dur`는 버킷 지속 시간(밀리초), `aln`은
  밀리초 단위의 타임 버킷 정렬입니다.
  - 예시: `key_SUM_60000_1000`

### Precedence order
1. `ts-compaction-policy` 설정 파라미터
2. Compaction rule이 없는 경우

### Example rules
- `max:1M:1h` - 1분 간격으로 최댓값을 집계하고, 최근 1시간 동안의 데이터만 유지합니다.
- `twa:1d:0m:360M` - 하루 단위로 시간 가중 평균을 사용해 집계하며 정렬 기준은 자정이 아닌 오전 6시로 맞춥니다.
  - 만료 기간 없음

### Example
5개의 compaction rule로 설정된 압축 정책 예시입니다:

8.0 버전 미만:
~~~redis
$ redis-server --loadmodule ./redistimeseries.so COMPACTION_POLICY max:1m:1h;min:10s:5d:10d;last:5M:10m;avg:2h:10d;avg:3d:100d
~~~

8.0 버전 이상:
~~~redis
redis> CONFIG SET ts-compaction-policy max:1m:1h;min:10s:5d:10d;last:5M:10m;avg:2h:10d;avg:3d:100d
~~~

## DUPLICATE_POLICY / ts-duplicate-policy
중복된 타임스탬프를 가진 여러 샘플을 삽입할 때 (`TS.ADD`, `TS.MADD`)의 기본 처리 정책은 다음을 따릅니다:

| policy | description                                           |
|--------|-------------------------------------------------------|
| BLOCK  | 새로 보고된 값을 무시하고 오류를 반환합니다.                             |
| FIRST  | 새로 보고된 값을 무시합니다. (처음 값 유지)                            |
| LAST   | 새로 보고된 값으로 기존 값을 덮어씁니다.                               |
| MIN    | 새 값이 기존 값보다 작을 때만 덮어씁니다.                              |
| MAX    | 새 값이 기존 값보다 클 때만 덮어씁니다.                               |
| SUM    | 기존 샘플이 있으면 `(기존 값 + 새 값)`으로 덮어쓰고, 없으면 새 값을 그대로 설정합니다. |

새로운 시계열이 생성될 때 마다 기본 값이 추가됩니다.

Type: string

### Precedence order
복제 정책이 다른 수준에서 제공될 수 있기 때문에, 복제 정책의 실제 우선순위는 다음과 같습니다:
1. `TS.ADD`의 `ON_DUPLICATE_POLICY` 옵션
2. `TS.CREATE` 또는 `TS.ALTER`에서 설정한 키 수준의 `DUPLICATE_POLICY`
3. `ts-duplicate-policy` 설정 파라미터
4. 하드코딩된 기본 값: BLOCK

## RETENTION_POLICY / ts-retention-policy
새로 생성된 키에 대한 기본 보존 기간을 밀리초 단위로 설정합니다.

보존 기간은 각 키별로 가장 최근에 보고된 타임스탬프와 비교했을 때 샘플의 가질 수 있는 최대 수명입니다.
샘플들은 오직 자신의 타임스탬프와 이후에 호출되는 `TS.ADD`, `TS.MADD`, `TS.INCRBY`, `TS.DECRBY`를 통해 전달되는
타임스탬프간의 차이에 따라 만료됩니다.

값이 0이면 만료가 없다는 뜻입니다.

`COMPACTION_POLICY / ts-compaction-policy`, `RETENTION_POLICY / ts-retention-policy` 모두 명시되어 있으면,
새로 생성된 압축에 대한 보존 기간은 `COMPACTION_POLICY / ts-compaction-policy`에 명시된 보존 시간과 동일합니다.

### Precedence order
보존 시간이 다른 수준에서 제공될 수 있기 때문에, 실제 보존 시간의 우선순위는 다음과 같습니다:
1. `TS.CREATE`, `TS.ALTER` 명령어의 `RETENTION` 옵션으로 설정된 키 수준의 보존 기간
2. `ts-rentention-policy`의 설정 파라미터
3. 보존 시간 없음

### Example
기본 보존 기간을 300일로 설정한 예시입니다:

8.0 버전 미만:
~~~redis
$ redis-server --loadmodule ./redistimeseries.so RETENTION_POLICY 25920000000
~~~

8.0 버전 이상:
~~~redis
redis> CONFIG SET ts-retention-policy 25920000000
~~~

## ENCODING / ts-encoding 
참고: v1.6 이전에는 설정 파라미터의 이름이 `CHUNK_TYPE`이였습니다.

`ts-compaction-policy`가 설정되어 있을 때, 자동으로 생성되는 압축(compaction) 시계열에 대해 사용되는 
기본 청크 인코딩 방식입니다.

Type: string

Valid values: COMPRESSED, UNCOMPRESSED

### Precedence order 
1. `ts-encoding` 설정 파라미터
2. 하드코딩된 기본 값: COMPRESSED

### Example
기본 인코딩을 UNCOMPRESSED로 설정한 예시입니다:

8.0 버전 미만:
~~~redis
$ redis-server --loadmodule ./redistimeseries.so ENCODING UNCOMPRESSED
~~~

8.0 버전 이상:
~~~redis
redis> CONFIG SET ts-encoding UNCOMPRESSED
~~~

## IGNORE_MAX_TIME_DIFF / ts-ignore-max-time-diff and IGNORE_MAX_VAL_DIFF / ts-ignore-max-val-diff 
새로 생성된 키에 기본 값을 설정합니다.

Types:
- `ts-ignore-max-time-diff`: integer
- `ts-ignore-max-val-diff`: double

Valid ranges:
- `ts-ignore-max-time-diff`: [0 .. 9,223,372,036,854,775,807]
- `ts-ignore-max-val-diff`: [0 .. 1.7976931348623157e+308]

많은 센서들이 데이터를 주기적으로 보고합니다. 종종 측정된 값과 이전 측정 값 간의 차이는 무시할 수 있을 정도로 작으며,
이는 랜덤 노이즈나 측정 정확도의 한계와 관련이 있습니다. 이러한 상황에서는 새로운 측정값을 시계열에 추가하지 않는 것이 바람직할 수 있습니다.

다음 조건을 모두 만족할 경우, 새로운 샘플은 중복으로 간주되어 무시됩니다:
1. 압축 시계열이 아닐 것
2. `ts-duplicate-policy`가 LAST로 설정되어 있을 것
3. 샘플이 순차적으로 추가될 것 (timestamp >= max_timestamp)
4. 현재 타임스탬프와 이전 타임스탬프 간의 차이(timestamp - max_timestamp)가
ts-ignore-max-time-diff 이하일 것 
5. 현재 값과 이전 값 간의 절댓값 차이(abs(value - value_at_max_timestamp))가 
ts-ignore-max-val-diff 이하일 것

여기서 max_timestamp는 해당 시계열에서 가장 큰 타임스탬프를 가진 샘플의 타임스탬프이고,
value_at_max_timestamp는 그 시점에 기록된 값을 의미합니다.

### Precedence order
1. `ts-ignore-max-time-diff`와 `ts-ignore-max-val-diff`의 설정 파라미터
2. 하드코딩된 기본값: 0, 0.0

## NUM_THREADS / ts-num-threads 
클러스터 모드에서 TS.MRANGE, `TS.MREVRANGE`, `TS.MGET`, `TS.QUERYINDEX`와 같은 교차 키 쿼리를 수행할 때, 
샤드당 최대 사용 가능한 스레드 수를 지정합니다. 이 값은 1 이상이어야 합니다. 이 값을 설정하는 것은 성능을 저하시킬 수도 있고,
향상시킬 수도 있습니다.

Type: integer

Valid range: [1..16]

Redis Open Source default: 3

Redis Software default: 플랜에 따라 설정되며, 플랜을 변경하면 자동으로 업데이트됩니다.

Redis Cloud defaults:
- Flexible & Annual: 플랜에 따라
- Free & Fixed: 1

### Example

8.0 버전 미만:
~~~redis
$ redis-server --loadmodule ./redistimeseries.so NUM_THREADS 3
~~~
8.0 버전 이상:
~~~redis
redis> redis-server --loadmodule ./redistimeseries.so ts-num-threads 3
~~~

## OSS_GLOBAL_PASSWORD
8.0 버전 이전에는 클러스터에서 시계열을 사용할 경우, 모든 클러스터 노드에 `OSS_GLOBAL_PASSWORD` 설정 파라미터를
설정했어야 했습니다. Redis는 더 이상 이 파라미터를 사용하지 않거나 무시합니다. Redis는 이제 노드들 간 내부 명령어를
전송하기 위해 새로운 shared secret 메커니즘을 사용합니다.