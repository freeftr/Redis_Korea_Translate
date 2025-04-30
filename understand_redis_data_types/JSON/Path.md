# Path

JSON 문서의 특정 요소에 접근하세요.
<br><br>
경로는 JSON 문서의 특정 요소에 당신이 접근할 수 있게 해줍니다. JSON 경로에 대한
표준 문법이 존재하지 않기 때문에, Redis JSON 독자적인 문법을 사용합니다. JSON 문법은
기존의 선례들을 참고하여 기반하였으며, JSONPath 문법과 의도적으로 비슷하게 만들어졌습니다.
<br><br>
JSON은 두 가지 쿼리 문법을 지원합니다: JSONPath 문법, 첫 번째 JSON 버전 문법
<br><br>
JSON은 경로 쿼리의 첫 번째 문자로 어느 문법을 사용하지 결정됩니다. **$**로 시작되면
JSONPath 문법을 사용합니다. 나머지 경우는 레거시 경로 문법을 사용합니다.
<br><br>
반환되는 값은 최상위에 JSON으로 직렬화된 문자열 배열을 포함한 JSON 문자열입니다. 
그리고 다중 경로(multi-paths)가 사용되는 경우, 반환 값은 값들이 직렬화된 JSON 값의 배열인 최상위 객체를 포함한 JSON 문자열입니다.

## JSONPath support

RedisJSON v2.0에서 **JSONPath**지원을 밝혔습니다. Goessner씨의 글에 설명되어
있는 문법을 따릅니다.
<br><br>
JSONPath 쿼리는 JSON 문서 내에서 여러 위치를 대상으로 결과를 반환할 수 있습니다.
이 경우, JSON 명령어는 가능한 모든 주소에 대해서 작동합니다. 이 점은 legacy 경로 쿼리가
가장 첫 번째 경로에만 작동하는 것에 비해 가장 큰 개선점입니다.
<br><br>
JSONPath를 사용하면 명령의 응답의 구조가 종종 달라진다는 것에 주의하세요.
**Commands** 페이지를 보시면 추가적인 정보를 보실 수 있습니다.
<br><br>
새로운 문법은 대괄호 표현법을 지원하며, 이를 통해 ":"같은 특수문자나 공백 문자도
키 이름에 포함을 시킬 수 있습니다.
<br><br>
CLI에서 큰 따옴표를 포함시키고 싶으면, JSONPath를 작은 따옴표로 감싸야 합니다.
다음은 그 예입니다.

```redis
JSON.GET store '$.inventory["mountain_bikes"]'
```

## JSONPath syntax

다음 JSONPath 문법 표는 Goessner's **path syntax compariso**을 참고하여 작성되었습니다.
<br><br>

| 문법 요소            | 설명                                                                                                                                                        |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| $                | 루트, 경로의 시작                                                                                                                                                |
| . or []          | 자식 요소 지정                                                                                                                                                  |
| ..               | 현재 위치를 기준으로 모든 하위 요소들을 재귀적으로 탐색                                                                                                                           |
| *                | 와일드카드, 모든 요소 반환                                                                                                                                           |
| []               | 배열의 요소에 접근하는 연산자                                                                                                                                          |
| [.]              | 객체 그 자체로 선택                                                                                                                                               |
| [start:end:step] | 인덱스인 start, end, 그리고 step을 기준으로 배열을 슬라이싱한다. 값을 생략할 수도 있으며, 기본값도 사용할 수 있다. start는 가장 첫번째 인덱스가 기본이고, end는 마지막이고, step은 1이다. [*] 또는 [:]를 통해 전체 요소를 선택할 수 있다. |
| ?()              | JSON 객체나 배열을 필터링한다. 비교연산자, 논리연산자, 괄호를 지원합니다.                                                                                                              |
| ()               | 스크립트 표현                                                                                                                                                   |
| @                | 현재 요소를 의미하며, 필터나 스크립트 표현식에서 사용된다.                                                                                                                         |

## JSONPath examples

다음의 JSONPath 예시는 아래 가게 물품에 대한 정보를 담은 JSON 문서를 사용하는 예시입니다.

```json
{
    "inventory": {
        "mountain_bikes": [
            {
                "id": "bike:1",
                "model": "Phoebe",
                "description": "This is a mid-travel trail slayer that is a fantastic daily driver or one bike quiver. The Shimano Claris 8-speed groupset gives plenty of gear range to tackle hills and there\u2019s room for mudguards and a rack too.  This is the bike for the rider who wants trail manners with low fuss ownership.",
                "price": 1920,
                "specs": {"material": "carbon", "weight": 13.1},
                "colors": ["black", "silver"],
            },
            {
                "id": "bike:2",
                "model": "Quaoar",
                "description": "Redesigned for the 2020 model year, this bike impressed our testers and is the best all-around trail bike we've ever tested. The Shimano gear system effectively does away with an external cassette, so is super low maintenance in terms of wear and tear. All in all it's an impressive package for the price, making it very competitive.",
                "price": 2072,
                "specs": {"material": "aluminium", "weight": 7.9},
                "colors": ["black", "white"],
            },
            {
                "id": "bike:3",
                "model": "Weywot",
                "description": "This bike gives kids aged six years and older a durable and uberlight mountain bike for their first experience on tracks and easy cruising through forests and fields. A set of powerful Shimano hydraulic disc brakes provide ample stopping ability. If you're after a budget option, this is one of the best bikes you could get.",
                "price": 3264,
                "specs": {"material": "alloy", "weight": 13.8},
            },
        ],
        "commuter_bikes": [
            {
                "id": "bike:4",
                "model": "Salacia",
                "description": "This bike is a great option for anyone who just wants a bike to get about on With a slick-shifting Claris gears from Shimano\u2019s, this is a bike which doesn\u2019t break the bank and delivers craved performance.  It\u2019s for the rider who wants both efficiency and capability.",
                "price": 1475,
                "specs": {"material": "aluminium", "weight": 16.6},
                "colors": ["black", "silver"],
            },
            {
                "id": "bike:5",
                "model": "Mimas",
                "description": "A real joy to ride, this bike got very high scores in last years Bike of the year report. The carefully crafted 50-34 tooth chainset and 11-32 tooth cassette give an easy-on-the-legs bottom gear for climbing, and the high-quality Vittoria Zaffiro tires give balance and grip.It includes a low-step frame , our memory foam seat, bump-resistant shocks and conveniently placed thumb throttle. Put it all together and you get a bike that helps redefine what can be done for this price.",
                "price": 3941,
                "specs": {"material": "alloy", "weight": 11.6},
            },
        ],
    }
}
```

가장 먼저 데이터베이스에 JSON 문서를 만드세요.

```redis
> JSON.SET bikes:inventory $ '{ "inventory": { "mountain_bikes": [ { "id": "bike:1", "model": "Phoebe", "description": "This is a mid-travel trail slayer that is a fantastic daily driver or one bike quiver. The Shimano Claris 8-speed groupset gives plenty of gear range to tackle hills and there\'s room for mudguards and a rack too. This is the bike for the rider who wants trail manners with low fuss ownership.", "price": 1920, "specs": {"material": "carbon", "weight": 13.1}, "colors": ["black", "silver"] }, { "id": "bike:2", "model": "Quaoar", "description": "Redesigned for the 2020 model year, this bike impressed our testers and is the best all-around trail bike we\'ve ever tested. The Shimano gear system effectively does away with an external cassette, so is super low maintenance in terms of wear and tear. All in all it\'s an impressive package for the price, making it very competitive.", "price": 2072, "specs": {"material": "aluminium", "weight": 7.9}, "colors": ["black", "white"] }, { "id": "bike:3", "model": "Weywot", "description": "This bike gives kids aged six years and older a durable and uberlight mountain bike for their first experience on tracks and easy cruising through forests and fields. A set of powerful Shimano hydraulic disc brakes provide ample stopping ability. If you\'re after a budget option, this is one of the best bikes you could get.", "price": 3264, "specs": {"material": "alloy", "weight": 13.8} } ], "commuter_bikes": [ { "id": "bike:4", "model": "Salacia", "description": "This bike is a great option for anyone who just wants a bike to get about on With a slick-shifting Claris gears from Shimano\'s, this is a bike which doesn\'t break the bank and delivers craved performance. It\'s for the rider who wants both efficiency and capability.", "price": 1475, "specs": {"material": "aluminium", "weight": 16.6}, "colors": ["black", "silver"] }, { "id": "bike:5", "model": "Mimas", "description": "A real joy to ride, this bike got very high scores in last years Bike of the year report. The carefully crafted 50-34 tooth chainset and 11-32 tooth cassette give an easy-on-the-legs bottom gear for climbing, and the high-quality Vittoria Zaffiro tires give balance and grip.It includes a low-step frame , our memory foam seat, bump-resistant shocks and conveniently placed thumb throttle. Put it all together and you get a bike that helps redefine what can be done for this price.", "price": 3941, "specs": {"material": "alloy", "weight": 11.6} } ] }}'
```

## Access examples

다음의 예시는 JSON 문서의 다양한 경로에서 **JSON.GET** 명령어를 이용하여 데이터를 조회하는 예시입니다.
<br><br>
와일드 카드 연산자 *를 이용하여 인벤토리 안의 모든 아이템들을 반환받을 수 있습니다.

```redis
> JSON.GET bikes:inventory $.inventory.*
"[[{\"id\":\"bike:1\",\"model\":\"Phoebe\",\"description\":\"This is a mid-travel trail slayer...
```

어떤 쿼리들에 대해서는 다중 경로는 같은 결과를 만들 수 있습니다. 그 예시로 
다음의 경로들은 mountain bikes의 모든 이름을 반환합니다.

```redis
> JSON.GET bikes:inventory $.inventory.mountain_bikes[*].model
"[\"Phoebe\",\"Quaoar\",\"Weywot\"]"
> JSON.GET bikes:inventory '$.inventory["mountain_bikes"][*].model'
"[\"Phoebe\",\"Quaoar\",\"Weywot\"]"
> JSON.GET bikes:inventory '$..mountain_bikes[*].model'
"[\"Phoebe\",\"Quaoar\",\"Weywot\"]"
```

.. 연산자를 통해 JSON 문서의 여러 섹션에서 필드를 가져올 수 있습니다.
다음 예시는 인벤토리 물품의 모든 이름을 가져옵니다.

```redis
> JSON.GET bikes:inventory $..model
"[\"Phoebe\",\"Quaoar\",\"Weywot\",\"Salacia\",\"Mimas\"]"
```

배열 슬라이싱을 통해 범위 안의 요소를 선택할 수 있습니다.
다음 예시는 첫 두 mountain bikes의 이름들을 반환합니다.

```redis
> JSON.GET bikes:inventory $..mountain_bikes[0:2].model
"[\"Phoebe\",\"Quaoar\"]"
```

필터 표현식 ?()은 특정 조건에 대한 JSON 요소를 검색하게 해줍니다. 
비교 연산자(==, !=, <, <=, >, >=, =~는 v2.42부터 지원)를 논리 연산자(&&, ㅣㅣ),
그리고 괄호((, ))와 같은 표현식들과 같이 사용할 수 있습니다. 필터 표현식은
배열이나 객체에 적용될 수 있으며, 배열이나 객체의 모든 값들을 순회하면서 조건에 
맞는 값들만 조회합니다.
<br><br>
필터 조건내에서 경로는 점 표기법을 사용합니다. @은 현재 배열 요소나 객체 값을 나타내며, **$**은 최상위 요소를 나타냅니다. 예시로,
@.key이름을 통해 중첩된 값을 나타내고, $.최상위_요소_이름을 통해 최상위 요소를 나타냅니다.
<br><br>
버전 v2.4.2 부터는 비교연산자 =~을 사용할 수 있으며, 이는 왼쪽에 있는 문자열 값의 경로를 오른쪽에 있는 정규 표현식과 비교해줍니다.
더 많은 정보를 위해서는 **supported regular expression syntax docs**를 참조하세요.
<br><br>
문자열이 아닌 값들은 비교가 안됩니다. 오로지 왼쪽 값이 문자열 경로이고, 오른쪽 값이 하드코딩된 문자열 값이거나, 문자열 경로일때만 적용됩니다.
아래의 예시를 보세요.
<br><br>
정규 표현식 매칭은 부분적입니다. 이 말은 "foo"와 같은 패턴은 "barefoots"와 같은 패턴에도 일치한다는 뜻입니다. 정확한 일치 여부를 확인하고
싶으면, "^foo$" 정규표현식 패턴을 사용하세요.
<br><br>
다른 JSONPath 엔진들은 정규 표현식 패턴을 /로 감싸서 사용할 수 있으며, 정확히 일치하는지 판별합니다. 부분일치를 적용시키고 싶으면, 
/.*foo.*/같은 정규 표현식을 사용하시면 됩니다.

## Filter examples

다음의 예시에서는 가격이 3000 이하이고, 무게가 10보다 적은 산악자전거들을 반환합니다.

```redis
> JSON.GET bikes:inventory '$..mountain_bikes[?(@.price < 3000 && @.specs.weight < 10)]'
"[{\"id\":\"bike:2\",\"model\":\"Quaoar\",\"description\":\"Redesigned for the 2020 model year...```
```

다음 예시는 합금으로 만들어진 자전거의 이름들을 반환합니다.

```redis
> JSON.GET bikes:inventory '$..[?(@.specs.material == "alloy")].model'
"[\"Weywot\", \"Mimas\"]"
```

다음의 예시는 버전 v2.4.2부터 유효한 예시이며, "al-"로 시작하는 재료로 만들어진
자전거들만 정규식을 사용하여 필터링합니다. 정규 표현식 패턴에 포함된 접두사
(?i) 덕분에 대소문자를 구분하지 않을 수 있습니다.

```redis
JSON.GET bikes:inventory '$..[?(@.specs.material =~ "(?i)al")].model'
"[\Quaoar\",\"Weywot\",\"Salacia\",\"Mimas\"]"
```

JSON 객체의 속성을 활용하여 정규식 패턴을 지정할 수도 있습니다. 예시로, 각 산악
자전거에 regex_pat 문자열 속성을 추가하고, 값으로 "(?i)al"을 설정하여 앞선 예시처럼
재질과 매칭할 수 있습니다. 그 다음, regex_pat을 자전거의 재료와 비교하여 매칭할 수 있습니다.

```redis
> JSON.SET bikes:inventory $.inventory.mountain_bikes[0].regex_pat '"(?i)al"'
OK
> JSON.SET bikes:inventory $.inventory.mountain_bikes[1].regex_pat '"(?i)al"'
OK
> JSON.SET bikes:inventory $.inventory.mountain_bikes[2].regex_pat '"(?i)al"'
OK
> JSON.GET bikes:inventory '$.inventory.mountain_bikes[?(@.specs.material =~ @.regex_pat)].model'
"[\"Quaoar\",\"Weywot\"]"
```

## Update examples

JSONPath 쿼리를 이용하여 JSON 문서의 특정 부분을 수정할 수 있습니다.

예시로, **JSON.SET** 명령어를 통해 특정 필드를 수정하기 위해 JSONPath를 설정할
수 있습니다. 다음의 예시는 헤드폰 목록에 있는 첫 번째 항목의 가격을 수정하는 예시입니다.

```redis
> JSON.GET bikes:inventory $..price
"[1920,2072,3264,3941]"
> JSON.NUMINCRBY bikes:inventory $..price -100
"[1820,1972,3164,1375,3841]"
> JSON.NUMINCRBY bikes:inventory $..price 100
"[1920,2072,3264,1475,3941]"
```

filter 표현식을 통해 조건을 만족하는 특정 JSON 요소만 수정할 수 있습니다.
다음의 예시는 가격이 2000이하인 자전거의 가격을 1500으로 설정하는 예시입니다.


```redis
> JSON.SET bikes:inventory '$.invetory.*[?(@.price<2000)].price' 1500
OK
> JSON.GET bikes:inventroy $..price
"[1500,2072,3264,1500,3941]"
```

JSONPath 쿼리는 경로를 매개변수로 받는 다른 JSON 명령어들과 함께 사용할 수 있습니다.
예시로, **JSON.ARRAPPEND** 명령어를 사용하여 헤드폰들에 새로운 색 옵션을 추가할 수 있습니다.

```redis
> JSON.ARRAPPEND bikes:inventory '$.inventory.*[?(@.price<2000)].colors' '"pink"'
1) (integer) 3
2) (integer) 3
127.0.0.1.6379> JSON GET bikes:inventory $..[*].colors
"[[\"black\",\"silver\",\"pink\"],[\"black\",\"white\"],[\"black\",\"silver\",\"pink\"]]"
```

## Legacy path syntax

***

RedisJSON v1에서는 다음과 같은 경로 방식을 사용합니다. JSON v2에서는
JSONPath 방식에 더해 legacy 방식도 지원합니다.

경로는 항상 Redis JSON 값의 루트에서 시작됩니다. 루트는 마침표 문자(.)로 표시됩니다.
루트의 하위 요소를 참조하는 경로의 경우, 이는 생략이 가능합니다.

Redis JSON은 객체 키 접근에 대해 점 표기법과 대괄호 표기법 모두 지원합니다. 
다음의 경로는 headphones를 가리키며, headphones는 루트 아래 inventory의 하위 요소입니다.

- .inventory.headphones
- inventory["headphones"]
- ['inventory']["headphones"]

배열의 요소에 접근하기 위해서는 요소의 인덱스를 대괄호로 감싸면 됩니다.
인덱스는 0부터 시작하며, 그 다음은 1, 그 다음은 순차적으로 증가합니다.
음수 인덱스를 사용하여 배열의 끝으로부터 요소를 접근할 수도 있습니다. 예시로,
-1은 배열의 마지막 요소를 가리키고, -2은 마지막으로부터 2번째 요소를 가리킵니다.

## JSON key names and path compatibility

JSON 키는 JSON 문자열이면, 그 어떤 값이든 가능합니다. 반면, 경로는 JavaScript(와 Java)
의 변수명 컨벤션을 따릅니다.


JSON은 임의의 키 이름을 가진 객체를 저장할 수 있지만, 다음과 같은 이름 규칙을
따르는 경우에만 legacy 경로를 사용할 수 있습니다.

1. 이름은 항상 문자, 달러 표시($), 또는 언더스코어(_) 문자로 시작해야 합니다.
2. 이름은 문자들, 숫자들, 달러 표시들, 언더스코어 문자들을 포함할 수 있습니다.
3. 이름은 대소문자를 구분합니다.

## Times complexity of path evaluation

***

경로를 따라 요소를 검색하는 시간복잡도는 다음과 같이 계산됩니다:

1. 하위 단계 - 경로상의 각 단계마다 추가적인 검색 발생
2. 키 검색 - 부모 객체의 키 개수를 N이라 할 때, 검색 시간은 O(N)†입니다.
3. 배열 검색 - O(1)

따라서 경로를 검색하는 전체 시간복잡도는 **O(N*M)**이 되며,
여기서 N은 경로의 깊이, M은 부모 객체의 키 수입니다.

N이 작은 객체에서는 이 정도 복잡도가 허용 가능하지만, N이 큰 객체에서는 접근 성능을 최적화할 수 있습니다.