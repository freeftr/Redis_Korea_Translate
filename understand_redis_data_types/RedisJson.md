
# JSON

JSON 데이터 타입은 Redis에서 JavaScript 객체 표기법을 지원합니다.  
JSON 데이터 타입을 통해 JSON 문서들을 저장하고, 조회하고, 수정할 수 있습니다.  
JSON 문서는 JSONPath 표현식을 통해 질의하거나, 조작할 수 있습니다.  
또한, JSON은 Redis Query 엔진과도 원활하게 연동되어 JSON 문서들을 조회하고, 인덱싱할 수 있습니다.

## Primary features

- JSON 표준에 대한 완전한 지원
- 문서 내 요소를 조회하거나, 수정할 수 있는 JSONPath 문법을 지원합니다. (JSONPath syntax 참고)
- 하위 요소들에 빠르게 접근할 수 있게 트리 구조의 이진 데이터로 저장된 문서들
- JSON 값 타입들에 대한 타입이 지정된 원자적 연산을 제공합니다.

## Use Redis with JSON

첫 번째로 해볼 JSON 명령어는 **JSON.SET**입니다. **JSON.SET**을 통해 Redis 키에 JSON 값을 설정할 수 있습니다.  
**JSON.SET**은 모든 JSON 값 타입을 허용합니다. 아래 예시는 JSON 문자열을 생성하는 예시입니다.

```redis
JSON.SET bike $ '"Hyperion"'
OK
JSON.GET bike $
"["Hyperion"]"
JSON.TYPE bike $
1) "string"
```

위의 명령을 보시면 **$** 기호가 있는 것을 확인하실 수 있습니다.  
이것은 JSON 문서 내 값의 경로를 나타내며, 위의 예시에서는 루트를 가리킵니다.

<br><br>

아래는 추가적인 문자열 관련 명령어들입니다.  
**JSON.STRLEN**은 문자열의 길이를 알려주며, **JSON.STRAPPEND**을 통해 다른 문자열에 이어 붙일 수 있습니다.

```redis
JSON.STRLEN bike $
1) (integer) 8
JSON.STRAPPEND bike $ '" (Eduro bikes)"'
1) (integer) 23
JSON.GET bike $
"["Hyperion (Enduro bikes)"]"
```

숫자들은 증가시키거나 곱할 수 있습니다:

```redis
JSON.SET crashes $ 0
OK
JSON.NUMINCRBY crashes $ 1
"[1]"
JSON.NUMINCRBY crashes $ 1.5
"[2.5]"
JSON.NUMINCRBY crashes $ -0.75
"[1.75]"
JSON.NUMMULTBY crashes $ 24
"[42]"
```

다음은 JSON 배열과 객체를 포함한 좀 더 흥미로운 예시입니다:

```redis
JSON.SET newbike $ '["Deimos", {"crashes": 0}, null]'
OK
JSON.GET newbike $
"[["Deimos",{"crashes":0},null]]"
JSON.GET newbike $[1].crashes
"[0]"
JSON.DEL newbike $[-1]
(integer) 1
JSON.GET newbike $
"[["Deimos",{"crashes":0}]]"
```

```text
정리 [박종하]
- JSON.SET: 키에 JSON 배열을 설정한다.
- JSON.GET: 키에 저장된 전체 JSON을 가져온다.
- JSON.DEL JSON 배열 $[-1]: 배열에서 마지막 요소를 삭제한다.
```

**JSON.DEL** 명령어는 경로 파라미터로 지정한 JSON 값을 삭제합니다.  
JSON 명령어 중 배열 전용 하위 명령어들을 사용하여 조작할 수 있습니다:

```redis
JSON.SET riders $[]
OK
JSON.ARRAPPEND riders $ '"Norem"'
1) (integer) 1
JSON.GET riders $
"[["Norem"]]"
JSON.ARRINSERT riders $ 1 '"Prickett"' '"Royce"' '"Castilla"'
1) (integer) 4
JSON.GET riders $
"[["Norem", "Prickett", "Royce", "Castilla"]]"
JSON.ARRTRIM riders $ 1 1
1) (integer) 1
JSON.GET riders $
1) "[["Prickett"]]"
JSON.ARRPOP riders $
1) ""Prickett""
JSON.ARRPOP riders $
1) (nil)
```

JSON 객체 또한 자체 명령어를 가지고 있습니다.

```redis
JSON.SET bike:1 $ '{"model": "Deimos", "brand": "Ergonom", "price": 4972}'
OK
JSON.OBJLEN bike:1 $
1) (integer) 3
JSON.OBJKEYS bike:1 $
1) 1) "model"
   2) "brand"
   3) "price"
```

## Format CLI output

CLI에는 raw 모드가 있어 **JSON.GET**의 결과를 보다 읽기 쉽게 포맷팅할 수 있습니다.  
Redis-cli에서 **--raw** 옵션을 통해 **INDENT**, **NEWLINE**, 그리고 **SPACE** 등의 키워드를 **JSON.GET** 명령어에 포함시키면 됩니다:

```redis
redis-cli --raw
JSON.GET obj INDENT "	" NEWLINE "
" SPACE " " $
[
    {
        "name": "Leonard Cohen",
        "lastSeen": "1478476800",
        "loggedOut": true
    }
]
```

## Enable Redis JSON

Redis JSON 타입은 Redis Community Edition에 포함되어 있으며, Redis Software 및 Redis Cloud에서도 사용이 가능합니다.  
**Install Redis Community Edition**이나 **Install Redis Enterprise**를 문서를 통해 전체 설치 방법을 확인할 수 있습니다.

## Limitation

JSON 값은 최대 depth 128까지 명령에 입력될 수 있으며, 이를 초과해서 입력하게 되면 해당 명령어는 오류를 반환합니다.

## Further information

Redis JSON에 대해 더 알고 싶으시면 아래 섹션의 페이지들에서 확인할 수 있습니다.  
<br><br>
**Index/Search JSON documents**
- Redis JSON과 Redis Query 엔진을 같이 활용해 JSON 문서를 인덱싱하고 조회할 수 있습니다.
  <br>
  **Path**
- JSON 문서의 특정 요소들에 접근할 수 있습니다.
  <br>
  **Use cases**
- JSON 사용 예시들
  <br>
  **Performance**
- 성능 벤치마크
  <br>
  **Redis JSON RAM Usage**
- 메모리 소모량 디버깅
  <br>
  **Guide for migrating from RESP2 to RESP3 replies**
- 클라이언트 개발자를 위한 RESP2에서 RESP3로의 마이그레이션 가이드
  <br>
  **Developer notes**
- JSON 디버깅, 테스트, 문서화에 관한 개발자 노트
