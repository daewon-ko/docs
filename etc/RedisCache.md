# Redis Cache 도입
#docs/Redis
#docs/쇼핑몰프로젝트/고민사항

### 배경
결제 기능을 토이프로젝트에 도입했다. 
결제 기능을 대표적 PG인 토스의 외부 API 기능을 이용하게끔 구현했는데, 해당 결제내역을 조회함에 있어서 Redis의 Cache 기능을 도입해보려고 한다.
해당 기술 도입에 앞서, Redis의 Cache에 대해서  학습해보고자 한다.

### Redis Cache
* Cache hit
  * Cache Store에 Data가 있을 경우 바로 가져온다.(빠름)
* Cache miss
  * Cache Store에 Data가 없으면, DB에서 조회한다.(느림)

-> Redis Cache 이용시 주요한 이슈사항이 바로 ‘**데이터 정합성**’ 문제이다.

데이터 정합성 문제를 해결하기 위해 적절한 ‘캐시 쓰기 전략’과 ‘ 캐시 읽기 전략’을 구사해야한다.

### 캐시 읽기 전략
1. Look Aside 
   - 가장 일반적인 캐시 읽기 전략
   - **애플리케이션이 데이터 조회와 캐싱 로직을 책임진다.**
   - DB와 Cache Store가 이중화 되어있으므로 Redis가 다운되더라도 DB에서 데이터를 조회할 수 있으므로 서비스 자체에는 문제가 없다.
     - 그러나, Redis에 몰려있던 Connection이 순간 DB에 몰린다면 DB 부하 현상이 발생할 수 있다.
   - Data가 Cache Store에서 우선적으로 확인하고 없으면 DB에서 조회한다.
     - 1. Cache Store에서 조회
       2. Cache Hit 시 Data 반환 Or Cache miss시 DB에서 조회 
       3. DB에서 조회한 Data Cache Store에 Update
   - 반복적 읽기가 많은 호출에 적합하다.
     - 초기 조회 시 DB에서 무조건 조회하므로 단건 조회에는 적합하지 않음
   - Cache Warming
     - 미리 Cache로 DB의 데이터를 일정용량 넣어두는 것. 
     * 서비스 초기 트래픽 집중 시, Cache Miss를 방지할 수 있다.
2. Read Through 
   - 캐시 시스템 자체가 데이터 소스를 읽어오는 로직을 포함
   - **애플리케이션은 캐시에 접근만하며 캐시는 데이터르 조회하거나 읽어오는 과정을 관리**
   - Cache에서만 데이터를 읽어오는 전략
     - 1. Cache Store에서 조회 
       2. Cache Hit시 반환 / Cache Miss시 DB에서 데이터를 가져온다.
   - Look Aside와 비슷하지만 데이터 동기화를 라이브러리에 위임
   - 데이터 조회 속도가 느리다. 
   - Redis 다운시 서비스에 이용제한이 발생한다.
   

### 캐시 쓰기 전략
1. Write Back
   - 데이터가 먼저 캐시에 저장되고 이후 DB에 비동기적으로 저장됨
     - 캐시에서 오류 발생시, 데이터 영구 손실 발생 가능성 존재
   - 쓰기 성능이 가장 좋으나 캐시와 DB의 불일치성 문제 발생
   - Write가 빈번하지만 Read를 하는데 많은 양의 자원이 소모되는 서비스에 적합
2. Write Through
   - 데이터가 항상 캐시와 DB에 동기적으로 기록
   - 데이터 일관성이 높으나 이에따라 쓰기 작업이 느릴 수 있다.
   - 데이터 유실이 발생하면 안되는 상황에 적합하며, 매 요청시 두번의 쓰기작업이 수행되므로 빈번한 생성 및 수정이 발생하는 서비스에서는 성능 이슈 문제가 있다. 
3. Write Around
   - 캐시를 건너뛰고 모든 데이터를 DB에 저장한다.
   - 읽기 작업 발생시, Cache Miss 발생하는 경우에만 DB와 캐시에 데이터를 저장한다. 
   - 따라서 데이터 불일치 문제가 존재

### 캐시 조합
1. Look Aside + Write Around
2. Read Through + Write Through

일반적으로 가장 많이 사용된다.
