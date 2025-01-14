---
layout: post
title: MongoDB Aggregation Pipeline 내부 동작 원리와 최적화
date: 2025-01-14 22:20
description:
tags: database
categories: study
---


[MongoDB에서 ‘pipeline’이라는 표현을 사용하는 이유](https://minngki.github.io/blog/2025/mongodb-pipeline/)가 궁금해 포스팅을 한 적이 있다.
그 과정에서 pipeline의 동작원리에 대해 더 깊은 궁금증이 생겨, 이번에는 MongoDB의 동작 원리를 파헤치고 정리한 글이다.


## Aggregation Pipeline의 내부 동작 개요
이전에 작성한 글에서 MongoDB는 Aggregation Pipeline 실행 시 각 단계(`$match`, `$group`, `$sort`, `$lookup`)를 순차적으로 적용시킨다고 정리했었다.
여기서 더 세부적으로 정리하자면, Mongo 엔진은 다음과 같은 과정을 거친다.

#### 1) Parsing
- 클라이언트가 전달한 pipeline array를 받아서, 각 단계에서 올바르고 유효한 문법인지 validate하는 과정이다.
- 즉, 구문 분석과 타당성 검증을 담당한다.

#### 2) Planning
- validation이 끝나면 실행 계획을 수립한다. 어떤 인덱스를 쓸지, 병렬 처리는 어떻게 할지, 단계의 순서는 어떻게 유지하거나 조정할지 등 실행 결로를 결정한다.
- 실질적인 Optimization 작업이 이루어진다.

#### 3) Stage Execution
- 실제 데이터(document)를 읽어서 각 단계의  **연산(filtering, grouping, sorting)을 수행**한다.
- 각 단계는 독립적인 연산자처럼 동작하며, 앞 단계의 결과를 받아 연산 후 다음 단계로 넘어가는 방식이다.

#### 4) Returning
- 최종 단계까지 처리된 결과를 클라이언트에게 반환하거나, 특정 collection에 저장한다.
- 해당 결과를 기존 collection에 덮어 쓸 수 있고, 새로운 collection 에 생성하거나 병합할 수 있다.
  - 특정 collection에 기록하는 방법
    ```js
      { 
          // "aggregatedResults" collection이 없다면 새로 생성, or not 덮어쓰며 결과 저장.
          "$out": "aggregatedResults" 
      }
    ```
    

## Optimization
최적화 로직에 대해 조금만 자세히 살펴보자면,

#### 1) 인덱스 사용
- `$sort` 앞 단계에서나 `$match`에 인덱스가 걸려 있으면, full collection 스캔을 최소화할 수 있게 index only scan을 진행하도록 실행 계획을 세운다.
  - Covered Query

#### 2) Pipeline stage 병합/재배치
- 내부적으로 `$match` 스테이지가 여러 번 등장하면, 가능한 한 앞단에서 합쳐서 수행하도록 최적화한다.
- `$limit`,` $skip` 등이 있다면, 이들을 앞단으로 당겨서 불필요한 연산을 줄일 수 있는지 검토한다.
- `$sort`와 `$group`의 순서에 따라, 인덱스를 사용하는 방식이나 내부 동작이 달라지므로, 가끔 순서를 조정할 수도 있다.

## mongo의 동작 원리 정리

##### 1. Document 스캔 및 조회
- 파이프라인의 첫 단계에서 mongoDB는 collection에서 document를 읽기 시작한다.
- `$match` 조건에 부합하는 문서만 통과시키기 때문에 앞 단에 `$match`로 filter를 걸어주는 게 효율적이다.

##### 2. 단계 별 연산 수행

##### 3. 순차적으로 앞 단계의 결과를 다음 단계로 전달

##### 4. 메모리/디스크 사용
- `$sort`나 `$group`과 같은 많은 document를 한 번에 다루는 단계는 메모리에 중간 단계의 데이터를 보관한다.
- mongoDB는 기본 100MB까지 메모리를 사용하고, 이를 초과하면 `allowDiskUse` 옵션이 켜진 경우 디스크에 임시 공간을 활용한다고 한다.
  - _회사에서 쌓인 log json은 크기가 커서 sort할 때, 과하게 오래 걸렸나 싶다.._

##### 5. 병렬 처리 (Sharded Cluster인 경우)
- 각 샤드에서 병렬로 부분 결과(ex.부분 집계)를 만들고, 해당 결과를 mongos(router) 혹은 최종 샤드에서 합치는 흐름을 거친다.
- 수평 확장을 통해 대규모 데이터를 빠르게 처리할 때 사용한다.
- ~~아직 사용해보지 않았지만 추후 다뤄보고 싶다..~~