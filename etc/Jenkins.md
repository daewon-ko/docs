# Jenkins(배포) 





- 최소 자바 버전 11이상을 지원함. 

- 젠킨스의 기본포트는 8080
  - 스프링이 기동될때 기본 포트가 8080이므로, 젠킨스용 별도의 서버를 두는게 아니라면 젠킨스의 포트를 바꿔서 기동하는게 필요. 
- 원격저장소(gitea / githbub) 등과 webhook으로 연동 필요
- 특징
  - github actions와 같이 클라우드 환경에 ci/cd를 수행하는게 존재하는게 아님
  - 특정 서버에 jenkins를 설치 후에 실행해야함. 
  - ci를 완전하게 구성하기 위해서는 물리적으로 별도의 '젠킨스 서버' 구축이 필요
    - application을 실행하는 환경과 jenkins서버를 별도로 분리하지 않는다면, 

---

- jenkins 개발 서버 내 설치경로

  >  /usr/lib/systemd/system/jenkins.service

  - 8085포트로 설정

### CLI 워크 플로우 

1. jenkins설치

   1. 리눅스 패키지 최신화

   2. jenkins설치

   3. jenkins실행 환경변수 변경(9090)

   4. jdk실행 버전 설정

      ```
      # JDK 17로 Jenkins 실행
      /usr/lib/jvm/java-17-openjdk-amd64/bin/java -jar jenkins.war
      ```

      sudo mkdir -p /etc/apt/sources.list.d

2. cli 다운로드

3. ssh 키 생성

4. Credential 등록(원격 저장소에)

5. JenkinsFile작성

6. job 생성

7. job 실행







https://blog.naver.com/cjhol2107/223500804255





