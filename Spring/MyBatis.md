#  MyBatis



- SqlSession

  - DB연결을 위한 객체
    - Q)  db connection Pool을 해당 객체가 처리하나? 
  - db connection 1개와 맵핑됨. Connction 객체를 다루지 않고 쿼리 실행

- SqlSessionFactory 

  - SqlSession을 생성하는 팩토리.SqlSessionTemplate

  - Thread -Safe하게 SqlSession을 제어하기 위함
    - 내부적으로 커넥션관리 / 트랜잭션 연동 / 예외처리 변환까지 자동처리`
  