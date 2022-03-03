## Many-to-One & Many-to-Many Relationships `(p. 33)`

### 정규화(Normalization) 이슈
- 정규화 및 id 필드 사용의 장점
    - 중복 저장 제거 / 업데이트 용이성
    - 검색 용이성
    - 로컬라이제이션 용이성 (minerva-olivo 사전 프로젝트에서 db 스키마 설계 사례)
- 정규화는 결국 `many-to-one` 관계를 필요로 함
    - 문서 모델은 관계형 테이블 모델과 달리, `many-to-one` / `many-to-many` 관계 표현에 취약
        - 문서 모델은 `one-to-many` 관계를 상정한 트리 구조의 데이터를 표현하기 위한 형태
        - 문서 모델을 구현한 db 들은 `join` 기능 지원 취약
    - 문서 모델에서 `many-to-one` 관계를 위한 `join` 을 애플리케이션 코드 레벨에서 구현해야 함
        - 만약 각 디멘전 필드의 규모가 작거나 변화가 거의 없다면, 메모리 상에 올려두고 작업 수행 고려
        - 하지만 db 가 아닌 코드 레벨에서 처리해야 한다는 단점 존재
    - 초기 유즈케이스에는 `many-to-one` 관계가 없어서 `join-free` 문서 모델을 채택하였더라도, 서비스가 업데이트 됨에 따라 데이터는 점점 유기적이 되는 것이 일반적
        - 향후 추가 피쳐들에 대한 고려와 함께 데이터 모델이 선정되어야 한다

## 데이터 모델 역사의 되풀이? `(p. 36)`
### `many-to-many` 관계와 `join` 을 다루는 가장 베스트한 방법은 무엇인가에 대한 오랜 논의
- 1970 년대: IBM's IMS(Information Management System)
    - `one-to-many` 트리 구조의 계층적 모델(Hierarchical model) 채택: 문서 모델(JSON 모델)과 유사
    - `join` 미지원 & `many-to-many` 관계 표현 불편
    - 이에 따라 데이터를 중복해서 저장(denormalize)하거나, 테이블들 간의 참조(reference)를 만들지 등의 논의가 시작되고 대안들이 제시됨
        - 대안 1) 네트워크(Network) 모델: CODASYL(Conference on Data Systems Language) model
            - IMS 가 사용하는 계층적 모델(Hierarchical model)의 일반화된 모델
            - 기존 계층적 모델과 달리, 각 레코드는 여러 부모(multiple parents)를 둘 수 있도록 허용하여 many-to-one / many-to-many 관계를 표현
            - 각 레코드 간의 링크는 관계형 모델의 FK가 아닌 프로그래밍 언어의 포인터와 유사한 개념
            - 각 레코드에 접근하기 위해선 `access path`를 따라, 루트 레코드로부터 링크 체인을 따라 순회해서 접근해야 함
            - `many-to-many` 관계의 특성 상, 특정 노드에 접근하는 `access path` 는 유일하지 않을 수 있음 -> 개발 시 이를 염두에 두어야해서 개발 코스트 증가
            - 쿼리 역시 `access path`를 따라 커서를 움직이며 수행됨 -> `access path`에 변경이 있으면 쿼리 코드 역시 이에 맞게 수정되어야 함
        - 대안 2) 관계형(Relational) 모델: SQL!
            - lay out all the data in open! 
            - 선언형(`declarative`) 쿼리언어 SQL
            - 쿼리 옵티마이저(query optimizer)에 기반한 추상화: `access path`에 해당하는 실행 순서/사용 인덱스 등은 옵티마이저 레벨에서 자동으로 결정됨
            - CODASYL에서의 `join`이 `insert time`에 수행되는 것과 달리, `join`이 `query time`에 수행됨
- 문서 모델과의 비교
    - 트리 구조의 저장 형태 측면에서는 계층 모델(hierarchical model)과 유사
    - `many-to-one`, `many-to-many` 관계의 표현 측면에서는, 관계형 모델과 유사점을 가짐
        - 관계된 아이템 간에는 unique id 를 통해 서로 참조함
        - 관계형 모델의 `foreign key` = 문서 모델의 `document reference`
        - 이 참조된 unique id 에 대한 데이터 간 연동은 `read time`에 `join` 이나 `follow-up query` 를 통해 수행됨

### Relational vs Document Databases Today `(p. 38)`

#### data model 측면에서의 비교
- 문서 모델의 장점
    - 스키마 탄력성(schema flexibility): schemaless ~ implicit schema ~ dynamic schema ~ schema-on-read
    - 로컬리티 측면에서의 성능 최적화
    - 애플리케이션 코드 내에서 사용되는 자료 구조와의 호환성 (JSON: JavaScript Object Notation)
- 관계형 모델의 장점
    - `join`의 네이티브한 지원 -> `many-to-one` / `many-to-many` 관계 지원 용이

#### 비교/선택 시 고려해야할 측면들
- 어떤 데이터 모델이 애플리케이션 코드를 더 간결하게 하는가? : 데이터 레코드 간의 관계 형태에 따라 최적의 답이 달라짐
- 스키마의 탄력성에 대한 고려 : `schema-on-read` (document db) vs `schema-on-write` (relational db)
    - cf) dynamic (runtime) type checking vs static (compile-time) type checking
- 쿼리되는 데이터의 로컬리티 측면에 대한 고려 : `storage locality` 관련 trade-off

#### 문서 db 와 관계형 db 는 수렴 중?! ~> 하이브리드화
- 2000년대 중반부터 관계형 db들이 JSON / XML 에 대한 지원을 시작함 ~ 문서 내의 구조에 대한 인덱스 및 쿼리 기능도 지원
- 문서형 db 들도 `join` 기능을 지원을 부분적으로나마 지원하기 시작

### Query Language for Data `(p. 42)`
- 명령형 언어(imperative language) vs 선언형 언어(declarative language)
    - 선언형 언어가 추상화에 유리함 : 사용 편의가 높을 뿐 아니라, 내부 구현 변경에도 용이
    - declarative language & automatic optimization
    - 선언형 쿼리 언어인 `SQL` 과 `쿼리 옵티마이저`의 조합 : 내부적으로 어떤 인덱스를 탈지, 어떤 순서로 실행될지, 어떤 조인 방식을 사용할지에 대해 사용자가 자유로움
    - 선언형 언어가 병렬화에 유리함 : 명령형 코드는 특정한 순서로 수행되도록 짜여지지만, 선언형 코드는 오직 결과 패턴만을 정의하므로
    - 생각해볼거리: 웹 프론트 ~ 반응형 프로그래밍, 쿠버네티스 커맨드라인 명령형/선언형 API
    > [Web 에서의 선언형 쿼리들]
    >
    > CSS selector & XSL/XPath 는 문서 스타일링을 위한 선언형 언어
- MapReduce 쿼리
    - MongoDB 들 일부 NoSQL 문서 db는 읽기 쿼리에 대해 제한적인 MapReduce 제공
    - 이때 map() / reduce() 함수는 `pure function`(순수 함수)여야 함 : 분산 처리시 어떤 순서로 어디에서 실행되더라도 문제 없고, 특정 태스크 단위에서만 실패 발생 시 독립적으로 재수행될 수 있도록
    > [순수함수(pure function)]: side-effect 가 없는 함수
    - MongoDB 에서는 MapReduce 에 대한 좀 더 상위 레벨의 추상화로서, JSON 기반 선언형 언어 형태의 `aggregation pipeline` 제공
    
## Graph-Like Data Models `(p. 49)`
- `many-to-many` 관계가 주가 되는 유즈 케이스라면..? -> Graph Model : vertice(=node) & edge(=relationship)
- 각 node/edge 는 homogeneous 할 필요가 없음 ~ Facebook 의 모델링 사례
- 생각해볼거리: Graph API / GraphQL

### Property Graphs (implemented by Neo4j)

### Cypher Query Language (declarative query language for property graphs[Neo4j])

### Graph Queries in SQL
- 최종적으로 목표로 하는 데이터에 이르기까지 몇번의 `join`이 필요한지 특정할 수 없음
- [SQL:1999] 표준부터 이러한 `variable-length traversal paths` 케이스에 대응하기 위해 `recursive common table expressions` 도입 (`WITH RECURSIVE` syntax)

### Triple-Stores and SPARQL
- property graph 모델과 유사하지만, 각각의 노드 연결 엣지를 튜플 형태로 저장한다는 점이 특징 `(subject, predicate[verb], object)`
- `subject`는 vertex 과 되며, `object`는 다른 vertex 일 수도 있고, (원시형 데이터타입일 경우에는) `subject` 노트의 프로퍼티가 됨
- 이러한 형태의 그래프 모델은 `semantic web` 에서 처음으로 쓰였음 ~ RDF(Resource Description Framework)
- SPARQL 은 RDF 데이터모델(triple-stores)에 대한 쿼리 언어임 (Cypher 는 SPARQL을 참조하여 개발됨)
> Graph Model vs Network Model (p. 60)
> 
> 네트워크 모델(CODASYL)은 데이터 계층에 대한 스키마를 정의하여 상/하위에 올 수 있는 레코드 타입이 제한됨 <-> 그래프 모델은 노드 연결 간에 스키마 제약이 없음
> 
> 네트워크 모델(CODASYL)은 특정 레코드 접근 시, access path 를 따라 순회해서만 접근할 수 있음 <-> 그래프 모델은 각 노드가 갖는 unique ID(UUID)를 통해, 각 노드에 직접 접근할 수 있음(refer directly by unique ID) + 인덱스를 둘 수 있음(index to find vertices with a particular value)
>
> 네트워크 모델(CODASYL)은 스토리지 레이아웃 특성 상 레코드의 순서에 대해 고려해야 함
> 
> 네트워크 모델(CODASYL)의 쿼리는 명령형인데 비해, 그래프 모델의 쿼리는 명령형/선언형 모두 사용 가능

### Datalog : 그래프 모델에 대해 처음으로 제안된 학술 모델 (상용 모델의 시초)
