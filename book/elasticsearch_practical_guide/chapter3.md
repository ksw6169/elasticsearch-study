# 03. 데이터 모델링

## 1. 매핑(Mapping)

- 인덱스에 추가되는 필드의 데이터 타입을 구체적으로 정의하는 것

- RDBMS에서 DDL문을 이용해 테이블을 생성할 때, 컬럼들의 데이터 타입을 지정하는 것과 유사함
- 생성된 매핑의 타입은 변경할 수 없음. 타입을 변경하려면 인덱스를 삭제한 후 다시 생성하거나 매핑을 다시 정의해야 하므로 주의 필요
- ES에서는 사전에 매핑을 설정하면 지정된 데이터 타입으로 색인되지만 매핑을 설정해두지 않으면 ES가 자동으로 필드의 데이터 타입을 결정해서 데이터를 색인함(=동적 매핑). 이는 예기치 않은 문제를 발생시킬 수 있으므로 주의 필요

<br>

**동적 매핑의 단점**

```
# doc 1
{
	"create_date":"20201126"
}

# doc 2
{
	"create_date":"test date"
}
```

- doc 1에서 create_date 필드에 대한 데이터를 색인하면 데이터 타입은 숫자 타입으로 매핑되는데, doc 2에서 text 타입의 데이터를 색인하려고 하면 색인에 실패한다. date 타입의 필드에 text 타입의 데이터를 색인하려고 했기 때문이다. 한번 정의된 필드의 타입은 변경할 수 없기 때문에 create_date 필드에 대한 데이터 타입은 인덱스를 재생성하거나 매핑을 다시 정의하지 않는 한 수정될 수 없다. 
- 이 밖에도 동적 매핑은 여러 단점을 내포하고 있다. 따라서 사용을 지양해야 한다.

<br>

**인덱스 생성 쿼리**

- 인덱스 생성 시 인덱스와 필드에 대한 설정 가능

```
PUT test_index
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard"
      },
      "content": {
        "type": "text",
        "analyzer": "standard"
      },
      "genre": {
        "type": "keyword"
      }
    }
  }
}
```

- `settings` : 인덱스에 대한 설정
  - number_of_shards : Primary shard 수 지정
  - number_of_replicas : Replica shard 세트 수 지정

- `mappings.properties` : 필드에 대한 설정
  - 필드명과 type(필드의 데이터 타입), analyzer(값 분석 시 사용할 분석기) 등을 지정할 수 있음

<br>

**인덱스의 매핑 확인 쿼리**

```
GET test_index/_mapping
```

<br>

## 2. 매핑 파라미터

매핑 파라미터를 이용해 필드의 데이터를 어떻게 저장할지 다양한 옵션을 지정할 수 있다.

- `analyzer` : 색인, 검색 시 필드의 데이터를 분석할 분석기를 지정하는 옵션

  - `analyzer ` 는 사전에 제공되는 analyzer도 있고, 사용자 정의하여 사용할 수도 있다.

  - `text` 데이터 타입은 analyzer를 별도로 지정하지 않을 경우 `standard analyzer`로 지정된다.

<br>

- `normalizer` : `keyword` 타입의 데이터 처리를 위해 사용되는 분석기를 지정하는 옵션

  - `keyword` 타입은 analyzer를 사용하지 않는 대신 `normalizer` 적용이 가능

  - `normalizer` 는 Character filter와 token filter만 조합해서 사용 가능(keyword 타입의 데이터가 별도의 토큰으로 분리되지 않기 때문에 analyzer에서 tokenizer만 제외된 구성)

<br>

- `boost` : 필드에 가중치(Weight)를 부여하는 옵션 (default : 1.0)
  - 가중치에 따라 유사도 점수(_score)가 달라지기 때문에 boost를 설정하면 검색 결과의 노출 순서에 영향을 줄 수 있음 (boost를 다른 필드에 비해 상대적으로 높게 줄 경우, 해당 필드로 검색된 데이터를 검색 결과의 상단에 보다 더 노출시킬 수 있음)
  - 검색 시에만 설정 가능

<br>

- `coerce` : 색인 시 자동 변환을 허용할지 여부(true/false)를 설정하는 옵션 (default : true)
  - 예를 들어, 숫자 형태의 문자열이 integer 필드에 들어올 경우 자동으로 형변환을 수행할지 여부로, false로 설정한 상태에서 이러한 경우가 발생하면 색인에 실패한다. 

<br>

- `copy_to` : 지정한 필드로 값을 복사하는 옵션
  - A 필드에서 copy_to로 B 필드명을 지정하면, A 필드의 값이 B 필드로 복사된다. 
  - 복사된 B 필드의 값은 _source에 표기되지 않으며, 검색만 가능하다. 
  - 여러 개의 필드 데이터를 하나의 필드에 모아서 전체 검색하기 위한 용도로도 사용된다. 

<br>

- `fielddata` : ES의 메모리 캐시 사용 여부 (default : false)
  - `fielddata` 는 ES가 힙 공간에 생성하는 메모리 캐시를 말하며, 여러 문제로 인해 현재는 거의 사용되지 않는다. 최신 버전의 ES에서는 `doc_values` 라는 새로운 메모리 캐시를 사용한다.
  - `text` 타입의 필드를 제외한 모든 필드는 기본적으로 `doc_values` 캐시를 사용한다. 
  - `fielddata` 는 메모리 소모가 크기 때문에 기본적으로 비활성화 되어 있다. 

<br>

- `doc_values` : ES의 메모리 캐시 사용 여부 (default : true)
  - 루씬을 기반으로 하는 캐시로 `text` 타입을 제외한 모든 타입에서 기본적으로 사용한다.
  - 이전의 `fielddata`는 캐시를 모두 메모리에 올려 사용했으나 `doc_values`는 OS의 파일 시스템 캐시를 이용해 디스크에 힙 사용에 대한 부담을 없앴다.
  - 한번 지정하면 인덱스를 재색인하지 않는 한 변경이 불가능하다. 

<br>

- `dynamic` : 필드 추가 시 동적 생성 지원 여부 (default : true)
  - 매핑에 정의되지 않은 필드를 추가할 경우 이를 허용할 것인지 지정하는 옵션
  - `true`, `false`, `strict` 지정 가능
    - `true` : 허용
    - `false` : 새로 추가되는 필드 무시. 필드는 색인되지 않아 검색은 불가능하나 _source에는 표시됨
    - `strict` : 필드가 새로 추가되면 이를 감지해 예외를 발생시키고 필드는 색인하지 않음

<br>

- `enabled` : 데이터를 색인하지 않고, 저장만 할지 여부 (default : false)
  - 최상위 매핑 정의 및 `object` 필드에만 설정 가능
  - enabled가 true로 지정된 필드는 색인이 되지 않으니 구문 분석을 건너뜀
  - `_source` 필드에서 검색 가능하나, 다른 방법으로는 검색할 수 없음

<br>

### 작성중

