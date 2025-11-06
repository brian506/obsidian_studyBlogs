징#spring개념 

## Controller

#### @RestController
- Controller + ResponseBody
- 메소드의 리턴값을 JSON(문자열) 형태로 반환

#### @ModelAttribute 
- 필터링 등 여러 개의 흩어진 옵션들의 파라미터들을 다룰때
- 조회(GET) 방식일 때 이용
- Content-type: application/x-www-form-urlencoded (key-value 형식)

#### @RequestBody 
- 프론트에서 body 에 담긴 JSON 데이터를 서버에서 자바객체로 변환시켜 저장
- 하나의 데이터 덩어리를 통째로 다룰 때 
- 데이터 생성,수정 등 POST,PUT,PATCH 방식 이용
- Content-type : application/ json 
- 모든 값들이 필요로 하게 될때

#### @ResponseBody 
- 자바 객체를 HTTP 응답 본문 내용으로 변환하여 클라이언트에게 전송

#### @PathVariable
- 경로 변수로 계층간 검색 및 조회에 쓰임
- 상세 조회, 삭제, 수정과 같은 작업에서 리소스 식별자로 쓰임
- URL1 -> URL2 로 넘어갈때 필요한 값을 그대로 가지고 오고 싶을때 다리 역할

#### @RequestParam
- HTTP 요청 파라미터를 받아오려고 할 때
	- 검색, 필터링, 페이징 등의 옵션을 클라이언트에게 받아올 때
	- 검색 조건이나 필터 조건을 URL 파라미터로 전달하고자 할 때


___
