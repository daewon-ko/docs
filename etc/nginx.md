# Nginx
#docs/ETC/nginx


### Nginx란?
* Web Server
  * 클라이언트 요청에 따른 정적 리소스를 빠르게 응답해준다.
* 단순 정적 리소스 응답 뿐 아니라 중개자로서 리버스 프록시, Load Balancer와 같은 역할을 수행할 수 있다.
* 클라이언트와 서버간의 트래픽을 암호화 가능
  * 클라이언트와의 연결에 대하여 SSL 끝점 역할을 수행한다.
  * 수신 SSL 연결을 처리 및 해독하고 

### 리버스 프록시
* 클라이언트의 요청을 받아 내부 서버로 전달하는 기능
  * 클라이언트의 요청을 WAS 서버로 중개하는 역할
* 역할
  * 캐시 기능 수행 가능
    * 자주 사용되는 정적파일을 캐시에 저장 후 빠르게 제공 가능
    * 또한 서버의 응답을 캐시에 저장 후, 서버에 재요청하지 않고 응답해줄 수 있다.
    * 리소스 절약 가능
  * 내부 WAS 보호
  * 로드 밸런싱 역할 수행
    * 많은 요청을 처리하기 위해 여러 대의 서버에 부하 분산
  
### HTTPS?
* 암호화 및 인증이 있는 HTTP
* 유일한 차이는 HTTPS가 TLS(SSL)을 사용하여 암호화한다.
* HTTPS를 제공하는 웹사이트는 CA(독립된 인증 기관)에서 SSL/TLS 인증서를 획득해야한다.
* HTTPS가 브라우저와 서버 간에 동작하는 매커니즘 및 순서는 다음과 같다.
~~~
  1 사용자 브라우저의 주소 표시줄에 *https://* URL 형식을 입력하여 HTTPS 웹 사이트를 방문합니다.
  2 브라우저는 서버의 SSL 인증서를 요청하여 사이트의 신뢰성을 검증하려고 시도합니다.
  3 서버는 퍼블릭 키가 포함된 SSL 인증서를 회신으로 전송합니다.
  4 웹 사이트의 SSL 인증서는 서버 아이덴티티를 증명합니다. 브라우저에서 인증되면, 브라우저가 퍼블릭 키를 사용하여 비밀 세션 키가 포함된 메시지를 암호화하고 전송합니다.
  5 웹 서버는 개인 키를 사용하여 메시지를 해독하고 세션 키를 검색합니다. 그런 다음, 세션 키를 암호화하고 브라우저에 승인 메시지를 전송합니다.
  6 이제 브라우저와 웹 서버 모두 동일한 세션 키를 사용하여 메시지를 안전하게 교환하도록 전환합니다.
~~~


참고
[HTTP와 HTTPS 비교 - 전송 프로토콜 간의 차이점 - AWS](https://aws.amazon.com/ko/compare/the-difference-between-https-and-http/)

### SSL 인증서란? 
* HTTPS를 사용하게 될 시에 클라이언트와 서버(이 경우엔 Nginx) 사이에 본인을 인증하는 서류같은 개
* SSL/TLS는 OSI 7 Layer의 특정 계층에 속하진 않지만, ‘세션 계층’과 ‘전송 계층’ 사이에서 동작한다고 간주
  * [참고](https://www.techtarget.com/searchsecurity/definition/Transport-Layer-Security-TLS)

### HTTPS를 적용하기 위해선 도메인을 구매해야만 하는가? 즉 필수인가? 


### Nginx 설치방법
참고
[Nginx 설치방법](https://velog.io/@jkijki12/%EB%B0%B0%ED%8F%AC-Aws-%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4%EC%97%90-Nginx-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0)



### Nginx Https 적용방법
[10분만에 끝내는 EC2 생성, NGINX 구성, SSL적용](https://creampuffy.tistory.com/191)
[웹서버에 HTTPS 적용하기 \(Let’s Encrypt, Nginx, AWS EC2\)](https://velog.io/@100journey/%EC%9B%B9%EC%84%9C%EB%B2%84%EC%97%90-HTTPS-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0-Lets-Encrypt-Nginx-AWS-EC2)


### CI/CD를 통해 스프링부트를 배포한다고 할때, nginx는 어떻게 설정해야하는가? 
* 클라이언트 - Nginx - WAS (복수)
  * Nginx가 정적 리소스를 응답해주기도 하고, 리버스프록시, 로드 밸런싱 등의 역할을 수행한다.
  
