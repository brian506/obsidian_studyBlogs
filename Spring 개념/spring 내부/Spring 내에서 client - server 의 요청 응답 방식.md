#spring개념 


# 정적 페이지(static pages)

![[스크린샷 2025-08-28 오후 9.32.48.png]]

- Web Server 에 미리 저장된 파일(HTML, CSS, JS, 이미지)을 불러와 구성하는 페이지

# 동적 페이지 (dynamic web page)

![[스크린샷 2025-08-28 오후 9.34.00.png]]


# Web Server

**특징** :
- HTTP 프로토콜 기반으로 클라이언트의 요청 담당
- *정적인 컨텐츠*를 요청 받았을 때는 *WAS 를 거치지 않고 바로 정적 컨텐츠 전달*
- *동적인 컨텐츠*를 제공할 때는 WAS 로 요청을 보내고 WAS 가 처리한 결과를 클라이언트에게 전달
- **정적 컨텐츠만 처리할 수 있도록 하는 것이 주요 기능**
- 필터링 하는 역할로 서버 부하를 줄일 수 있음

*예시* : Apache Server, [[Nginx]]

___

# WAS(Web Application Server)

> Servlet Container 라고도 불림

**특징** :
- 모든 HTTP 동적 요청이 들어올 때 WAS 로 들어옴
- **DB 조회나 다양한 동적 컨텐츠**를 제공하기 위해 만들어진 Application Server
- Servlet 을 관리하고 실행

*예시* : Tomcat

___

## Servlet

> 클라이언트의 요청을 처리하고 결과를 반환
![[스크린샷 2025-08-28 오후 9.45.33.png]]


**순서** :
1. 클라이언트 요청이 들어오면 Servlet Container 는 *Servlet 이 힙 영역에 있는지* 확인
2. 없으면 init() 을 호출하여 *초기화하고 적재*
3. 요청에 따라서 *service() 를 통해 요청에 대한 응답 생성*
4. Servlet 종료 요청을 하면 destroy() 호출

## Servlet Container

**특징** :
- Servlet의 생성, 초기화, 호출, 종료하는 생명 주기 관리
- 웹 서버와 클라이언트의 *요청에 대한 응답을 소켓으로 통신*
- 요청이 들어올때마다 새로운 자바 쓰레드를 생성하여 멀티 쓰레드 방식으로 처리

___

# Web Service 구조

![[스크린샷 2025-08-28 오후 9.01.56.png]]

**요청/응답 flow**

1. 클라이언트가 HTTP 요청을 보냄
	- _정적 요청_ 일 경우 Web Server 에서 파일 경로를 통해 **File-Content** 를 바로 반환
	- _동적 요청_ 일 경우 Web Server 는 **WAS(Web Application Server)** 로 요청을 보냄
2. WAS 는 관련된 Servlet 을 힙 메모리에 올림
3. WAS 는 yml 파일을 참조하여 해당 **Servlet 에 대한 쓰레드를 생성**
4. 컨테이너에서 **HttpServletRequest 와 HttpServletResponse 객체**를 생성하여 **Servlet 에 전달**
5. 쓰레드는 Servlet 의 **service() 호출**
6. service() 는 요청에 맞는 **doGet() or doPost() 호출**
7. 6번 과정에서 _DispatcherServlet → Controller → Service → Repository_ → 응답 생성 의 순서로 실행
8. Servlet 은 인자에 맞게 생성된 **동적 페이지**를 Response 객체에 담아 **WAS 에 전달**
9. WAS 는 Response 객체를 HttpResponse 형태로 바꾸어 Web Server 에 전달
10. 생성된 쓰레드를 종료하고 관련된 객체 모두 제거

___


## 요청 처리

> Spring 은 일반적으로 동기 + 블로킹 방식으로 요청을 처리

### 동기 방식 (synchronous)

- *요청하고 응답이 올때까지 기다리는* 방식
- Connection 유지

### 비동기 (asynchronous) 

- 요청하고 응답이 오지 않아도 *다른 작업을 처리하다가 응답 신호가 오면 결과를 읽어서 처리*하는 방식
- Connection 유지 X 
- 서로 간에 *event* 를 통해 통신하는 방식

### 블로킹 (blocking)

- *응답이 올때까지 쓰레드가 다음으로 진행하지 않고 계속 기다리는* 방식

### 논블로킹 (non-blocking)

- 작업이 끝날 때까지 쓰레드가 *다른 일도 동시에 처리*하는 방식