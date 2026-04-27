Nignx에서 받은 GPS 값을 Spring에서 검증 이후에 LLM과 RDS 에 쏴주는 방식을 Redis Streams 가 아닌 SQS + SNS 로 유실되지 않고 운영 효율적으로 메시지를 관리하기 위해서 사용

**동작 흐름**
앱 서버 -> SNS Topic -> SQS A or SQS B ...
- SNS는 메시지를 한번 쏴주면 저장하지 않기 때문에, 만약 consumer가 다운되면 메시지가 유실되므로, SNS에서 바로 쏴주지 않고, SQS 로 처리하도록 한다.

