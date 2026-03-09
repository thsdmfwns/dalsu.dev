---
title: DB와 JSON
description: meta description
image: images/post/DB&JSON.png
date: 2026-03-08T11:36:48+09:00
draft: false
author: SonDev
tags:
  - DB
  - NoSql
  - Json
  - SQL
categories:
  - DB
---
RDBMS(관계형 데이터베이스)의 중요한 목적 중 하나는 **데이터의 무결성을 보장하는 것**이다. 이를 위해 RDBMS는 컬럼 기반의 정형 스키마와 정규화된 구조를 바탕으로 데이터를 저장한다. 이러한 방식은 데이터 중복을 최소화하고, 제약조건과 관계 모델링을 통해 일관된 상태를 유지하는 데 효과적이다.

그러나 현실 세계의 데이터가 항상 고정된 컬럼 구조에 적합한 것은 아니다. 특히 파일 메타데이터와 같이 데이터의 속성이 자주 변경되거나, 객체마다 서로 다른 필드를 가지는 경우에는 전통적인 정규화 방식만으로 이를 표현하는 데 한계가 있다. 모든 속성을 컬럼으로 정의할 경우 테이블 구조가 과도하게 비대해지고, 사용되지 않는 컬럼과 `NULL` 값이 증가하며, 이에 따른 처리 복잡성도 함께 커지게 된다.

이러한 문제를 완화하기 위해, 실제 시스템에서는 공통적으로 관리해야 하는 핵심 속성만 관계형 스키마로 정의하고, 가변적인 속성은 JSON 형태로 저장하는 방식을 자주 사용한다. 이는 관계형 데이터베이스의 무결성과 구조적 장점을 유지하면서도, 반정형 데이터에 대한 유연한 표현을 가능하게 한다.

또한 RDBMS는 버전이 발전함에 따라 JSON 데이터를 단순 저장하는 수준을 넘어, JSON 내부 필드에 대한 조회, 함수 기반 처리, 인덱싱과 같은 기능을 점차 강화해 왔다. 그 결과 현대의 RDBMS는 정형 데이터와 반정형 데이터를 하나의 저장소 내에서 함께 다룰 수 있는 방향으로 확장되고 있다.

### MYSQL과 JSON
MySQL은 **5.7 버전부터 네이티브 JSON 타입**을 지원한다.  
이는 JSON 데이터를 단순히 `TEXT`로 저장하는 방식과 비교했을 때 몇 가지 뚜렷한 차이가 있다.

--- 
##### 1. 자동 유효성 검사
MySQL의 JSON 타입은 **저장 시점에 JSON 문서의 유효성을 자동으로 검사**한다.  
따라서 형식이 잘못된 JSON 데이터는 저장되지 않고 에러가 발생한다.
##### 2. 내부 바이너리 형식으로 저장
MySQL은 JSON 데이터를 단순 문자열 그대로 저장하지 않고,  
**내부 바이너리 형식으로 변환하여 저장**한다.

이 방식의 장점은 다음과 같다.
- 매번 문자열 전체를 파싱하지 않아도 된다.
- 특정 키나 배열 인덱스에 더 빠르게 접근할 수 있다.
- 중첩된 값이나 하위 객체를 보다 효율적으로 조회할 수 있다.
##### 3. 다양한 JSON 함수 지원
MySQL은 JSON 타입 자체뿐 아니라 JSON 데이터를 다루기 위한  
다양한 **JSON 함수**도 함께 제공한다.

--- 

하지만 인덱스 측면에서는 주의할 점도 있다.
MySQL의 JSON 컬럼은 일반 스칼라 컬럼처럼 **직접 인덱스를 걸 수 없다**.
대신 JSON 내부의 특정 값을 꺼내는 **생성 칼럼(generated column)** 을 만든 뒤, 그 칼럼에 인덱스를 생성하는 방식으로 최적화한다.

#### PostgreSQL과 JSON
PostgreSQL은 JSON 데이터를 저장하기 위해 `json`과 `jsonb` 두 가지 타입을 제공한다.  
두 타입 모두 JSON 데이터를 저장할 수 있지만, 가장 큰 차이는 **저장 방식**과 **조회 효율**에 있다.
- `json`
    - **원문 텍스트를 그대로 저장**.
- `jsonb`
    - **분해된 바이너리 형식으로 저장**.
    - 저장 시 변환 비용이 들지만, 이후 **조회와 연산 성능이 더 우수**.
    - **인덱스 지원**.

대부분의 경우 `jsonb`를 사용하는 편이 더 적합하다.
##### 인덱싱
MySQL과 PostgreSQL의 가장 큰 차이 중 하나는 **JSON 데이터에 대한 인덱싱 지원 방식**이다.
##### B-tree / Hash 인덱스

PostgreSQL의 `jsonb`는  **B-tree** 와 **Hash** 인덱스를 지원한다.  
B-tree와 Hash는 보통 **JSON 문서 전체를 하나의 값처럼 비교**할 때 유용하다. 공식 문서도 `jsonb`에 대한 B-tree/Hash 인덱스는 대체로 **완전한 JSON 문서의 동등성 확인**에 적합하다고 설명한다.

인덱스 생성 예시는 다음과 같다.

```SQL
CREATE INDEX idx_btree_jdoc ON api USING BTREE (jdoc);  
CREATE INDEX idx_hash_jdoc  ON api USING HASH (jdoc);
```

하지만 이런 인덱스는 **JSON 내부 구조를 검색하는 용도**에는 적합하지 않다.  
그런 경우에는 `GIN` 인덱스를 사용하는 것이 맞다.
##### GIN 인덱스
PostgreSQL의 `jsonb` 컬럼에는 **GIN 인덱스**를 생성할 수 있다.  
이를 통해 JSON 내부의 키, 값, 포함 관계, 그리고 `jsonpath` 조건 검색까지 효율적으로 처리할 수 있다.

또한 GIN 인덱스는 하나만 있는 것이 아니라, 목적에 따라 두 가지 연산자 클래스를 선택할 수 있다.
- **`jsonb_ops`**
    - 기본값.
    - 많은 연산자를 지원.
- **`jsonb_path_ops`**
    - 비기본.
    - 지원하는 연산자는 더 적다.
    - 대신 인덱스가 더 작고, 특정 검색에서는 더 빠를 수 있음.
##### `jsonb_ops` (기본값)
지원하는 주요 연산자는 다음과 같다.

- **`?`**  
    지정한 **키 또는 문자열 요소가 존재하는지** 확인한다.  
    예시:
    ```SQL
        data ? 'name'
    ```
    → JSON 객체에 `name` 키가 있으면 참이다.
- **`?|`**  
    여러 키(또는 문자열 요소) 중 **하나라도 존재하는지** 확인한다.  
    예시:
	```SQL
	data ?| array['name', 'email']
	```
    → `name` 또는 `email` 중 하나라도 있으면 참이다.
- **`?&`**  
    여러 키(또는 문자열 요소)가 **모두 존재하는지** 확인한다.  
    예시:
    ```SQL
    data ?& array['name', 'email']
    ```
    → `name`과 `email`이 모두 있어야 참이다.
- **`@>`**  
    왼쪽 JSON이 오른쪽 JSON을 **포함(containment)** 하는지 확인한다.  
    예시:
    ```SQL
    data @> '{"status": "active"}'
    ```
    → JSON 내부에 `"status": "active"` 구조가 포함되어 있으면 참이다.
- **`@?`**  
    **JSONPath 조건식에 일치하는 항목이 존재하는지** 확인한다.  
    예시:
    ```SQL
    data @? '$.items[*] ? (@.price > 100)'
    ```
    → `items` 배열 안에 `price > 100` 인 요소가 하나라도 있으면 참이다.
- **`@@`**  
    **JSONPath 조건식 자체를 평가**하여 boolean 결과를 반환한다.  
    예시:
    ```SQL
    data @@ '$.age > 20'
    ```
    → JSON의 `age` 값이 20보다 크면 참이다.

인덱스 생성 예시는 다음과 같다.
```SQL
CREATE INDEX idxgin ON api USING GIN (jdoc);
```
##### `jsonb_path_ops`
`jsonb_path_ops`는 더 제한적인 대신, 특정 상황에서 더 효율적인 선택지다.
지원하는 주요 연산자는 다음과 같다.
- `@>`
- `@?`
- `@@`

**키 존재 여부를 확인하는 `?`, `?|`, `?&` 연산자는 지원하지 않는다.**

인덱스 생성 예시는 다음과 같다.
```SQL
CREATE INDEX idxginp ON api USING GIN (jdoc jsonb_path_ops);
```
##### 인덱스를 통한 검색
`api` 테이블에 `jdoc jsonb` 컬럼이 있고, 
여기서 `company` 값이 `"Magnafone"`인 문서를 찾는다고 하자.
```SQL
SELECT jdoc->'guid', jdoc->'name'  
FROM api  
WHERE jdoc @> '{"company": "Magnafone"}';
```
이 쿼리는 `jdoc` 컬럼 자체에 `@>` 연산을 적용하고 있으니
`jdoc`에 생성된 **GIN 인덱스를 활용할 수 있다.**
### 벤치마킹
##### 벤치마킹 환경
이번 벤치마크는 다음 환경에서 수행했다.
```text
BenchmarkDotNet v0.15.8, Windows 11 (10.0.26200.7840/25H2/2025Update/HudsonValley2)
AMD Ryzen 5 5600X 3.70GHz, 1 CPU, 12 logical and 6 physical cores
.NET SDK 10.0.102
  [Host]     : .NET 9.0.12 (9.0.12, 9.0.1225.60609), X64 RyuJIT x86-64-v3
  Job-QIUXEI : .NET 9.0.12 (9.0.12, 9.0.1225.60609), X64 RyuJIT x86-64-v3
```
##### 벤치마킹 시나리오
**1. 비교 대상**
- MySQL `JSON`
- PostgreSQL `jsonb` 인덱스 없음
- PostgreSQL `jsonb` + GIN 인덱스

**2. 데이터**
- `Bogus`로 동일한 구조의 JSON 문서를 생성
- 테스트 건수는 `10 / 1,000 / 10,000 / 100,000`

**3. 측정 작업**
- **Insert**: 생성한 JSON 문서를 각 DB 테이블에 대량 삽입
- **Update**: 특정 조건에 맞는 JSON 문서를 찾아 값을 수정
- **Query**: 같은 조건으로 JSON 내부 값을 조회
##### 테스트에 사용한 JSON 구조
```C#
public class BenchmarkDocument  
{  
    public required string DocumentId { get; init; }  
    public required string Category { get; init; }  
    public required string Name { get; init; }  
    public required string Description { get; init; }  
    public required Profile Profile { get; init; }  
    public required Metrics Metrics { get; init; }  
    public required Flags Flags { get; init; }  
    public required string[] Tags { get; init; }  
    public required DateTime UpdatedAt { get; init; }  
}  
  
public sealed class Profile  
{  
    public required string Tier { get; init; }  
    public required string Region { get; init; }  
    public required string OwnerEmail { get; init; }  
}  
  
public sealed class Metrics  
{  
    public required int Score { get; init; }  
    public required int Stock { get; init; }  
    public required decimal Amount { get; init; }  
}  
  
public sealed class Flags  
{  
    public required bool Reviewed { get; init; }  
    public required bool Featured { get; init; }  
}
```

```JSON
{
  "documentId": "doc-0001",
  "category": "electronics",
  "name": "Wireless Headphones X100",
  "description": "Noise-cancelling wireless headphones with 30 hours battery life.",
  "profile": {
    "tier": "premium",
    "region": "ap-northeast-2",
    "ownerEmail": "owner@example.com"
  },
  "metrics": {
    "score": 87,
    "stock": 142,
    "amount": 199.99
  },
  "flags": {
    "reviewed": true,
    "featured": false
  },
  "tags": ["audio", "wireless", "noise-cancelling", "bluetooth"],
  "updatedAt": "2026-03-09T20:15:00Z"
}
```

#### INSERT

##### SQL
```SQL
MySqlInsert 
INSERT INTO benchmark_mysql (payload) VALUES (CAST(@Payload AS JSON));

PostgreSqlNoIndexInsert
INSERT INTO benchmark_pg_noindex (payload) VALUES (CAST(@Payload AS jsonb));

PostgreSqlGinInsert
INSERT INTO benchmark_pg_gin (payload) VALUES (CAST(@Payload AS jsonb));
```

##### 결과
{{< details summary="결과값 열기 / 닫기" >}}
> | Method                                       | RowCount |         Mean |         Error |       StdDev |       Median |          Min |          Max |       Gen0 |    Allocated |
| -------------------------------------------- | -------- | -----------: | ------------: | -----------: | -----------: | -----------: | -----------: | ---------: | -----------: |
| &#39;Insert - PostgreSQL jsonb (GIN)&#39;    | 10       |     10.88 ms |      8.192 ms |     0.449 ms |     10.71 ms |     10.54 ms |     11.39 ms |          - |     15.44 KB |
| &#39;Insert - MySQL JSON&#39;                | 10       |     16.12 ms |      9.107 ms |     0.499 ms |     15.94 ms |     15.73 ms |     16.68 ms |          - |     26.36 KB |
| &#39;Insert - PostgreSQL jsonb (No GIN)&#39; | 10       |     19.80 ms |    290.174 ms |    15.905 ms |     11.06 ms |     10.18 ms |     38.16 ms |          - |     15.64 KB |
| &#39;Insert - PostgreSQL jsonb (GIN)&#39;    | 1000     |    695.73 ms |    405.566 ms |    22.230 ms |    688.04 ms |    678.37 ms |    720.79 ms |          - |   1072.83 KB |
| &#39;Insert - PostgreSQL jsonb (No GIN)&#39; | 1000     |    700.01 ms |    224.318 ms |    12.296 ms |    698.81 ms |    688.36 ms |    712.86 ms |          - |   1080.64 KB |
| &#39;Insert - MySQL JSON&#39;                | 1000     |    938.38 ms |    603.450 ms |    33.077 ms |    921.15 ms |    917.48 ms |    976.52 ms |          - |   1928.66 KB |
| &#39;Insert - PostgreSQL jsonb (No GIN)&#39; | 10000    |  7,052.34 ms |  1,255.910 ms |    68.841 ms |  7,050.34 ms |  6,984.53 ms |  7,122.17 ms |          - |  10783.64 KB |
| &#39;Insert - PostgreSQL jsonb (GIN)&#39;    | 10000    |  7,299.68 ms |    145.755 ms |     7.989 ms |  7,301.71 ms |  7,290.87 ms |  7,306.45 ms |          - |  10705.82 KB |
| &#39;Insert - MySQL JSON&#39;                | 10000    |  8,310.88 ms | 10,085.672 ms |   552.830 ms |  8,038.33 ms |  7,947.25 ms |  8,947.07 ms |  1000.0000 |  19224.98 KB |
| &#39;Insert - PostgreSQL jsonb (GIN)&#39;    | 100000   | 71,595.40 ms | 89,859.219 ms | 4,925.487 ms | 74,359.54 ms | 65,908.67 ms | 74,517.98 ms |  6000.0000 | 107034.21 KB |
| &#39;Insert - PostgreSQL jsonb (No GIN)&#39; | 100000   | 71,612.43 ms |  4,839.844 ms |   265.288 ms | 71,648.69 ms | 71,330.88 ms | 71,857.73 ms |  6000.0000 | 107815.46 KB |
| &#39;Insert - MySQL JSON&#39;                | 100000   | 77,690.55 ms |  6,776.047 ms |   371.418 ms | 77,706.88 ms | 77,311.24 ms | 78,053.54 ms | 11000.0000 | 192194.38 KB |
{{< /details >}}

![insert_benchmark](images/DB&JSON/insert_benchmark.svg)

- Insert 성능은 전반적으로 **PostgreSQL jsonb가 MySQL JSON보다 우세**
- `jsonb`에 **GIN 인덱스를 추가해도 Insert 성능 저하는 제한적**
- **대량 삽입 구간에서는 GIN 적용 시 실행 편차가 더 커지는 모습**
- MySQL JSON은 모든 구간에서 더 느렸고, **클라이언트 측 메모리 할당량도 더 큼**

#### Query
##### SQL
```SQL
MySqlQuery
SELECT COUNT(*)  
FROM benchmark_mysql  
WHERE JSON_UNQUOTE(JSON_EXTRACT(payload, '$.profile.tier')) = @Tier;  
  
PostgreSqlNoIndexQuery
SELECT COUNT(*)  
FROM benchmark_pg_noindex  
WHERE payload @> CAST(@FilterJson AS jsonb); 
  
PostgreSqlGinQuery 
SELECT COUNT(*)  
FROM benchmark_pg_gin  
WHERE payload @> CAST(@FilterJson AS jsonb);
```

##### 결과
{{< details summary="결과값 열기 / 닫기" >}}
> | Method                              | RowCount | Mean      | Error      | StdDev     | Min       | Max       | Median    | Allocated |  
|------------------------------------ |--------- |----------:|-----------:|-----------:|----------:|----------:|----------:|----------:|  
| &#39;Query - PostgreSQL jsonb (No GIN)&#39; | 10       |  1.167 ms |   1.181 ms |  0.0647 ms |  1.101 ms |  1.230 ms |  1.169 ms |    3.7 KB |  
| &#39;Query - PostgreSQL jsonb (GIN)&#39;    | 10       |  1.177 ms |   2.193 ms |  0.1202 ms |  1.051 ms |  1.291 ms |  1.189 ms |   3.51 KB |  
| &#39;Query - PostgreSQL jsonb (No GIN)&#39; | 1000     |  1.630 ms |   3.227 ms |  0.1769 ms |  1.430 ms |  1.766 ms |  1.693 ms |   3.13 KB |  
| &#39;Query - PostgreSQL jsonb (GIN)&#39;    | 1000     |  1.878 ms |   3.128 ms |  0.1715 ms |  1.694 ms |  2.034 ms |  1.907 ms |   2.82 KB |  
| &#39;Query - MySQL JSON&#39;                | 10       |  2.119 ms |   4.400 ms |  0.2412 ms |  1.883 ms |  2.365 ms |  2.109 ms |   5.45 KB |  
| &#39;Query - MySQL JSON&#39;                | 1000     |  2.556 ms |   1.259 ms |  0.0690 ms |  2.487 ms |  2.625 ms |  2.556 ms |   6.38 KB |  
| &#39;Query - PostgreSQL jsonb (No GIN)&#39; | 10000    |  4.742 ms |   6.853 ms |  0.3756 ms |  4.523 ms |  5.176 ms |  4.527 ms |   3.13 KB |  
| &#39;Query - PostgreSQL jsonb (GIN)&#39;    | 10000    |  4.883 ms |   7.160 ms |  0.3924 ms |  4.619 ms |  5.334 ms |  4.695 ms |   2.95 KB |  
| &#39;Query - MySQL JSON&#39;                | 10000    |  7.827 ms |  12.241 ms |  0.6710 ms |  7.184 ms |  8.523 ms |  7.773 ms |   6.38 KB |  
| &#39;Query - PostgreSQL jsonb (GIN)&#39;    | 100000   | 36.106 ms |  44.767 ms |  2.4538 ms | 33.909 ms | 38.754 ms | 35.656 ms |   3.47 KB |  
| &#39;Query - PostgreSQL jsonb (No GIN)&#39; | 100000   | 37.393 ms |  57.768 ms |  3.1665 ms | 33.738 ms | 39.324 ms | 39.116 ms |    3.6 KB |  
| &#39;Query - MySQL JSON&#39;                | 100000   | 61.964 ms | 316.713 ms | 17.3601 ms | 51.674 ms | 82.007 ms | 52.211 ms |   6.38 KB |
{{< /details >}}

![query_benchmark](images/DB&JSON/query_benchmark.svg)
- **PostgreSQL `jsonb`가 전 구간에서 MySQL `JSON`보다 빠름**  
- **데이터 건수가 커질수록 PostgreSQL과 MySQL의 격차가 더 분명**  
- **대규모 데이터에서는 GIN이 소폭 유리한 결과**  
- **PostgreSQL은 인덱스 없이도 충분히 강한 조회 성능**
- **MySQL은 평균 응답 시간뿐 아니라 메모리 할당량도 더 크게 나타남.**  

#### UPDATE
##### SQL
```SQL
MySqlUpdate
UPDATE benchmark_mysql  
SET payload = JSON_SET(  
    payload,    '$.flags.reviewed', TRUE,    '$.metrics.score', CAST(JSON_UNQUOTE(JSON_EXTRACT(payload, '$.metrics.score')) AS UNSIGNED) + 1,    '$.updatedAt', @UpdatedAt)  
WHERE JSON_UNQUOTE(JSON_EXTRACT(payload, '$.profile.tier')) = @Tier;
  
PostgreSqlNoIndexUpdate
UPDATE benchmark_pg_noindex  
SET payload = jsonb_set(  
    jsonb_set(        jsonb_set(payload, '{flags,reviewed}', 'true'::jsonb, true),        '{metrics,score}',        to_jsonb(((payload #>> '{metrics,score}')::int + 1)),        true    ),    '{updatedAt}',    to_jsonb(CAST(@UpdatedAt AS text)),    true)  
WHERE payload @> CAST(@FilterJson AS jsonb);
  
PostgreSqlGinUpdate
UPDATE benchmark_pg_gin  
SET payload = jsonb_set(  
    jsonb_set(        jsonb_set(payload, '{flags,reviewed}', 'true'::jsonb, true),        '{metrics,score}',        to_jsonb(((payload #>> '{metrics,score}')::int + 1)),        true    ),    '{updatedAt}',    to_jsonb(CAST(@UpdatedAt AS text)),    true)  
WHERE payload @> CAST(@FilterJson AS jsonb);
```

##### 결과
{{< details summary="결과값 열기 / 닫기" >}}
> | Method                               | RowCount | Mean         | Error        | StdDev      | Median       | Min          | Max          | Allocated |  
|------------------------------------- |--------- |-------------:|-------------:|------------:|-------------:|-------------:|-------------:|----------:|  
| &#39;Update - PostgreSQL jsonb (No GIN)&#39; | 10       |     2.490 ms |     5.599 ms |   0.3069 ms |     2.502 ms |     2.178 ms |     2.791 ms |    4.2 KB |  
| &#39;Update - PostgreSQL jsonb (GIN)&#39;    | 10       |     2.579 ms |     4.978 ms |   0.2729 ms |     2.547 ms |     2.323 ms |     2.866 ms |   4.14 KB |  
| &#39;Update - MySQL JSON&#39;                | 10       |     6.998 ms |    10.375 ms |   0.5687 ms |     7.162 ms |     6.365 ms |     7.466 ms |   6.19 KB |  
| &#39;Update - PostgreSQL jsonb (No GIN)&#39; | 1000     |     8.758 ms |    20.583 ms |   1.1282 ms |     9.157 ms |     7.485 ms |     9.633 ms |   3.51 KB |  
| &#39;Update - PostgreSQL jsonb (GIN)&#39;    | 1000     |    10.077 ms |    23.819 ms |   1.3056 ms |    10.450 ms |     8.626 ms |    11.156 ms |   3.68 KB |  
| &#39;Update - MySQL JSON&#39;                | 1000     |    12.995 ms |    15.461 ms |   0.8475 ms |    13.264 ms |    12.046 ms |    13.675 ms |   5.55 KB |  
| &#39;Update - PostgreSQL jsonb (No GIN)&#39; | 10000    |    39.891 ms |    23.417 ms |   1.2835 ms |    39.372 ms |    38.948 ms |    41.352 ms |   3.81 KB |  
| &#39;Update - MySQL JSON&#39;                | 10000    |    56.704 ms |   326.157 ms |  17.8778 ms |    46.828 ms |    45.944 ms |    77.341 ms |   5.91 KB |  
| &#39;Update - PostgreSQL jsonb (GIN)&#39;    | 10000    |    64.534 ms |    89.235 ms |   4.8913 ms |    66.715 ms |    58.932 ms |    67.957 ms |    3.8 KB |  
| &#39;Update - PostgreSQL jsonb (No GIN)&#39; | 100000   |   538.435 ms | 2,743.160 ms | 150.3619 ms |   615.209 ms |   365.185 ms |   634.910 ms |   3.51 KB |  
| &#39;Update - MySQL JSON&#39;                | 100000   |   631.533 ms | 4,656.165 ms | 255.2201 ms |   487.963 ms |   480.432 ms |   926.203 ms |   5.91 KB |  
| &#39;Update - PostgreSQL jsonb (GIN)&#39;    | 100000   | 1,382.050 ms | 3,623.424 ms | 198.6121 ms | 1,465.259 ms | 1,155.367 ms | 1,525.524 ms |    3.8 KB |
{{< /details >}}

![query_benchmark](images/DB&JSON/update_benchmark.svg)
- **전반적인 Update 성능은 PostgreSQL `jsonb (No GIN)`이 가장 좋게 나타남**  
- **GIN 인덱스는 Update 성능에 명확한 쓰기 오버헤드를 만듦**  
- **대규모 Update에서는 GIN의 비용은 매우 큼**  
- **이번 결과는 “조회 최적화용 인덱스가 쓰기 성능을 희생한다”는 점을 잘 보여줌**  

#### 벤치마크 총평
이번 벤치마크를 통해 MySQL `JSON`, PostgreSQL `jsonb`, 그리고 PostgreSQL `jsonb + GIN`의 차이는 단순히 “어느 쪽이 더 빠른가”보다, **어떤 작업을 더 중요하게 보느냐에 따라 선택이 달라진다**는 점이 분명하게 드러났다.

먼저 **Insert**에서는 PostgreSQL `jsonb`가 전반적으로 MySQL `JSON`보다 더 빠른 성능을 보였고, GIN 인덱스를 추가하더라도 쓰기 오버헤드는 예상보다 크지 않았다. 즉, 단순 적재 관점에서는 PostgreSQL 쪽이 유리했고, GIN 인덱스도 Insert 단계에서는 비교적 감당 가능한 비용으로 보였다.

반면 **Query**에서는 PostgreSQL `jsonb`가 전 구간에서 MySQL보다 더 좋은 성능을 보였다. 다만 이번 실험에서 GIN 인덱스의 효과는 기대만큼 극적으로 드러나지 않았다. 이는 사용한 쿼리 패턴이 GIN의 장점을 최대한 끌어내는 형태가 아니었기 때문이며, 현재 결과는 **PostgreSQL `jsonb` 자체의 조회 성능이 이미 충분히 우수하다**는 점을 더 강하게 보여준다. 즉, PostgreSQL은 인덱스가 없어도 강했고, GIN은 특정 조건에서만 추가 이점을 제공하는 모습이었다.

가장 큰 차이는 **Update**에서 나타났다. PostgreSQL `jsonb`는 인덱스가 없을 때 가장 좋은 Update 성능을 보였지만, GIN 인덱스를 추가하면 데이터 규모가 커질수록 쓰기 비용이 크게 증가했다. 특히 대량 Update에서는 GIN 유지 비용이 매우 크게 드러나, 조회 최적화를 위한 인덱스가 쓰기 성능에는 분명한 페널티를 만든다는 점이 확인됐다. 이 부분은 JSON 문서를 자주 수정하는 시스템에서 매우 중요한 판단 기준이 된다.