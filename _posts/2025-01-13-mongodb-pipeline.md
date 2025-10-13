---
layout: post
title: MongoDB는 왜 ‘query’가 아니라 ‘pipeline’이라고 표현할까?
date: 2025-01-13 4:20
description:
tags: database nosql
categories: study
---

jupyter에서 DB를 연결해서 데이터를 다룰 때, RDB는 당연하게도 query라는 변수를 사용하고,
MongoDB는 pipeline이라는 변수를 사용하는 것을 발견했다.
문득 왜 pipeline이라는 표현을 하는지 궁금해져서 정리하는 글이다.

##### 왜 Mongo는 pipeline이라는 표현을 사용할까 ?

`Query`라는 표현 자체가 사용자가 원하는 특정 데이터가 무엇인지 명시(질의)하는 행위를 의미한다.
반면에, `Pipeline`은 데이터를 여러 단계에 거쳐 데이터를 처리하는 방식이다.

`Query`는 단순 질의를 강조하는 반면, `Pipeline`은 **“처리 흐름”**을 강조하는 용어로 보면 된다!

겉으로는 당연한 것처럼 보일 수 있지만,
실제로 구조를 이해하면서 말로 풀어 보니 더 직관적으로 이해할 수 있었다.

## 파이프라인 개념

파이프라인이란 데이터를 **단계별로** 처리하는 **연속적인 작업** 흐름을 말한다. Mongo의 aggregation pipeline 은 순차적으로 각 단계에서 하나씩 작업을 수행한다.
각 단계에서의 filtering, grouping, sorting, converting 이 가능하며,
이전 단계의 결과가 다음 단계의 입력으로 전달되므로 각 단계에서의 흐름을 명확하게 파악할 수 있다.

## Query와 Pipeline 처리방식 차이

> - RDBMS Query: **주로 단일 SQL 쿼리**로 DB에서 **결과**를 즉시 반환.
> - MongoDB Pipeline: 데이터를 **여러 단계**로 **순차적으로** 가공 가능하며, 최종 결과를 도출하는 **처리 과정**.

- _물론 Query도 `with`문과 같은 CTE, 서브쿼리, 뷰 와 같은 데이터 처리 방식도 있어 복잡한 데이터 처리가 가능하다..!_

### 예시

##### Query

```sql
SELECT
    contract_number,
    age,
    gender,
    JSON_ARRAYAGG(
        JSON_OBJECT(
            'name', outer.name,
            'size', outer.size,
            'warehouse', outer.warehouse
        )
    ) AS orders
FROM table
WHERE response.result = TRUE
AND outer.status IN ('배송전', '배송취하')
GROUP BY contract_number;
```

##### Pipeline

```python
query = '...'
pipeline = [
    {
        "$match": query  # 원하는 필터 조건 적용
    },
    {
        "$match": {  # log.response.result가 True인 데이터만 필터
            "log.response.result": True
        }
    },
    {
        "$unwind": "$log.response.orders"  # orders 배열 펼침
    },
    {
        "$unwind": "$log.response.orders.outer"  # outer 배열 펼침
    },
    {
        "$match": {
            "log.response.orders.outer.status": { "$in": ["배송전", "배송취하"] }
        }
    },
    {
        "$group": {
            "_id": "$log.response.orders.contract_number",  # contract_number로 그룹화
            "original_id": { "$first": "$_id" },
            "age": { "$first": "$log.response.age" },
            "gender": { "$first": "$log.response.gender" },
            "outer": {
                "$push": {
                    "name": "$log.response.orders.outer.name",
                    "domain_name": "$log.response.orders.outer.domain_name",
                    "size": "$log.response.orders.outer.size",
                    "warehouse": "$log.response.orders.outer.warehouse",
                    "discount_target": "$log.response.orders.outer.discount_target",
                    "status": "$log.response.orders.outer.status"
                }
            }
        }
    },
    {
        "$project": {
            "_id": 0,  # id값 분리
            "original_id": 1,  # 원본 _id 반환
            "contract_number": "$_id",
            "age": 1,
            "gender": 1,
            "orders": 1
        }
    }
]
```

- $match: 필터링
- $group: 그룹화
- $sort: 그룹화된 데이터 정렬
- $unwind: 원하는 배열 데이터 레벨에 접근
- $project: 반환할 컬럼 set
