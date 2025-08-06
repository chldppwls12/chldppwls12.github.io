---
title: Elastic Search 사용해보기 - CRUD
date: 2025-08-05 09:40:00 +/-TTTT
categories: [DB]
tags: [Elastic Search, NoSQL]
---

지난 글에서 [Elastic Search 기초](https://chldppwls12.github.io/posts/basics-of-elastic-search/)에 대해 알아봤습니다. 이번에는 실제로 Elastic Search 사용 방법에 대해 작성합니다.

<img src="/assets/img/using-elastic-search/elastic-search.png" width="300" alt="ES Logo">

## 설치

Docker를 사용해 Elasticsearch와 Kibana를 설치합니다. <br/>
(Kibana의 Dev Tools를 사용하면 API 테스트가 훨씬 편리하기에 같이 설치해주도록 합니다.)

```shell
# 네트워크 생성 (Elasticsearch와 Kibana가 통신할 수 있도록)
docker network create testnetwork

# Elastic search 실행
docker run -d --name elasticsearch \
  --net testnetwork \
  -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  elasticsearch:9.1.0

# Kibana 실행
docker run -d --name kibana \
  --net testnetwork \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  kibana:9.1.0
```

설치가 완료되면 다음 URL로 접근할 수 있습니다: <br/>

- Elasticsearch: [http://localhost:9200](http://localhost:9200)
- Kibana: [http://localhost:5601](http://localhost:5601)

Kibana 접속 후 좌측 메뉴에서 Management > Dev Tools로 이동하면 Elasticsearch API를 편리하게 테스트할 수 있습니다.

<img src="/assets/img/using-elastic-search/dev-tools.png" width="600" alt="Dev Tools">

## 2\. Index 관련 API 사용해보기

### Mapping
매핑이란 인덱스에 저장될 문서의 구조를 정의하는 것 입니다. RDB의 스키마와 비슷하지만 더 유연합니다.
```
# 요청
PUT /products
{
  "mappings": {           # 이 인덱스의 구조를 정의
    "properties": {       # 필드들의 속성 정의
      "name": {
        "type": "text",         # 전문검색이 가능한 텍스트
        "analyzer": "standard"  # 어떻게 분석할지 정의
      },
      "category": {
        "type": "keyword"       # 정확히 일치하는 검색용 (필터링, 집계)
      },
      "price": {
        "type": "integer"       # 숫자 타입
      },
      "description": {
        "type": "text"          # 전문검색 가능한 긴 텍스트
      }
    }
  }
}

# 결과
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "products"
}
```

#### 필드 타입 별 차이점 (text vs keyword)
```
# text: 전문검색용 (토큰화됨)
"name": {
  "type": "text"
}
# "삼성 갤럭시 S24" → ["삼성", "갤럭시", "S24"]로 분리되어 저장

# keyword: 정확한 매칭용 (토큰화 안됨)
"category": {
  "type": "keyword" 
}
# "스마트폰" → "스마트폰" 그대로 저장
```

### 생성 확인
```
# 인덱스 목록 확인
GET /_cat/indices?v

# 특정 인덱스 매핑 확인
GET /products/_mapping

# 인덱스 전체 정보
GET /products
```

## 3\. 문서 관련 API 사용해보기
### 단일 문서 생성
문서 생성 시, 문서 ID를 지정해주지 않으면 자동으로 생성됩니다.
```
# 문서 ID를 직접 지정하여 생성
PUT /products/_doc/1
{
  "name": "삼성 갤럭시 S24 울트라",
  "category": "스마트폰",
  "price": 1500000,
  "description": "최신 AI 기능이 탑재된 프리미엄 스마트폰입니다."
}

# 문서 ID를 자동 생성
POST /products/_doc
{
  "name": "아이폰 15 프로",
  "category": "스마트폰", 
  "price": 1350000,
  "description": "티타늄 소재로 제작된 Apple의 최신 플래그십 모델입니다."
}

# 결과
{
  "_index": "products",
  "_id": "1",
  "_version": 1,
  "result": "created",    # created 또는 updated
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  }
}
```

### Bulk insert로 한 번에 여러 문서 생성
```
POST /products/_bulk
{"index":{"_id":"3"}}
{"name":"LG 그램 노트북","category":"노트북","price":1200000,"description":"초경량 비즈니스 노트북"}
{"index":{"_id":"4"}}
{"name":"맥북 에어 M3","category":"노트북","price":1490000,"description":"Apple M3 칩셋이 탑재된 노트북"}
{"index":{"_id":"5"}}
{"name":"에어팟 프로","category":"이어폰","price":320000,"description":"노이즈 캔슬링 기능이 있는 무선 이어폰"}
```

### 조회
```
# 전체 문서 조회
GET /products/_doc/1

# 특정 필드만 조회
GET /products/_doc/1?_source=name,price

# 모든 문서 조회
GET /products/_search

# 특정 조건으로 검색
GET /products/_search
{
  "query": {
    "match": {
      "name": "갤럭시"
    }
  }
}

GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"category": "스마트폰"}}
      ],
      "filter": [
        {"range": {"price": {"gte": 1000000, "lte": 2000000}}}
      ]
    }
  }
}

# 결과
{
  "took": 5,
  "timed_out": false,
  "_shards": {"total": 1, "successful": 1, "skipped": 0, "failed": 0},
  "hits": {
    "total": {"value": 2, "relation": "eq"},
    "max_score": 0.18232156,
    "hits": [
      {
        "_index": "products",
        "_id": "1",
        "_score": 0.18232156,
        "_source": {
          "name": "삼성 갤럭시 S24 울트라",
          "category": "스마트폰",
          "price": 1500000,
          "description": "최신 AI 기능이 탑재된 프리미엄 스마트폰입니다."
        }
      }
    ]
  }
}
```

### 수정
전체 문서를 수정할 때는 PUT을 사용합니다.
```
PUT /products/_doc/1
{
  "name": "삼성 갤럭시 S24 울트라 512GB",
  "category": "스마트폰",
  "price": 1600000,
  "description": "512GB 저장공간을 가진 프리미엄 스마트폰입니다.",
  "color": "블랙"  # 새 필드 추가
}
```

부분적으로 수정할 경우에는 POST를 사용합니다.
```
POST /products/_update/1
{
  "doc": {
    "price": 1400000,           # 가격만 변경
    "discount_rate": 10         # 새 필드 추가
  }
}
```

### 삭제
```
DELETE /products/_doc/1
```
---
참고 자료
- [Elastic 가이드북](https://esbook.kimjmin.net/04-data)
