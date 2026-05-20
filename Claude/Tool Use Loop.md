> 사용자가 자연어로 명령하면, LLM이 문맥을 파악해 내 서버의 '특정 함수'를 실행하라고 트리거해주는 시스템

ex) 사용자가 채팅창에 "어제 시킨 맥북 언제 옴?"이라고 치면, LLM이 알아서 판단해 서버의 배송 상태 메서드를 대신 트리거 해준다.

즉, LLM은 스스로 물리적인 동작을 수행하는 것이 아닐, 필요한 파라미터를 입력해서 해당 함수를 실행하라는 지시서만 내려주는 역할을 한다. 실제 메서드 수행은 내 서버에서 한다.

## 동작 흐름

사용자가 "어제 시킨 맥북 언제 와?"라는 질문이 어떻게 처리되는지에 대한 동작흐름을 알아보자.

### 1. 내 서버 -> LLM : 질문 + 서버가 가진 함수에 대한 설명서(JSON) 전송
- 사용자의 질문과 함께 스킬 목록(함수 설명서)을 JSON 형태로 보낸다.
```json
// 내 백엔드 서버가 Claude API로 보내는 HTTP POST 요청
{
  "model": "claude-3-5-sonnet-20241022",
  "messages": [
    {"role": "user", "content": "어제 시킨 맥북 언제 와?"}
  ],
  "tools": [
    {
      "name": "check_delivery_status",
      "description": "사용자의 주문 및 배송 상태를 DB에서 조회합니다.",
      "input_schema": {
        "type": "object",
        "properties": {
          "item_name": { "type": "string", "description": "주문한 상품명" }
        },
        "required": ["item_name"]
      }
    }
  ]
}
```

- `check_delivery_status`라는 함수를 가지고 있고, 이 함수를 쓰려면 `item_name`이라는 파라미터를 사용해야 한다고 알려주는 과정

### 2. LLM -> 내 서버 : 함수 트리거
- 질문을 받은 LLM은 아까 받은 `check_delivery_status`함수를 기억하고 서버에게 해당 함수를 실행하도록 요청한다.
```json
// Claude API의 응답
{
  "stop_reason": "tool_use", 
  "content": [
    {
      "type": "text",
      "text": "주문하신 맥북의 배송 상태를 확인해 보겠습니다."
    },
    {
      "type": "tool_use",
      "id": "toolu_01ABC123",
      "name": "check_delivery_status",
      "input": { "item_name": "맥북" }
    }
  ]
}
```

`stop_reason: "tool_use"` : tool을 사용하기 위해서 LLM의 답변이 아직 끝나지 않음을 알림. 해당 응답을 끝내기 위해서 서버에 함수 실행을 위임하도록 하는 것
`id` : 해당 툴의 식별값으로, 어떤 호출의 결과인지 알기 위한 것이다. 하나의 호출에 여러 개의 tool을 이용해야 할 때, list에 같이 넣어서 처리한다.

**여기서 Claude는 도구를 직접 실행하는 것이 아니다.** 어떤 도구를 어떤 인자로 부를 지 알려줄 뿐, 실제 호출은 서버 코드가 실행한다.
- 매 호출은 **stateless** 하며, 이전 메시지 + 새 tool_result를 매번 다시 보내야 한다.
- `stop_reason`이 `end_turn` 이 될때까지 루프

### 3. 내 서버 : 실제 함수 실행
- 현재 시점에서의 LLM은 잠시 대기 상태이다. 
- 서버는 응답을 받아 파싱한다. `stop reason`이 tool_use인 것을 확인하고, Claude의 응답값을 기반으로 서버 내 코드를 실행한다.
```json
// Spring Boot 예시 로직
if ("tool_use".equals(stopReason)) {
    String itemName = toolInput.get("item_name"); // "맥북"
    
    // 이 부분은 온전히 내 서버의 로직! LLM은 모름!
    String deliveryStatus = myDatabase.query(
        "SELECT status FROM orders WHERE item = ?", itemName
    ); 
    // 조회 결과: "오늘 오후 3시 배송 완료 예정"
}
```

### 4. 내 서버 -> LLM : 응답 값 반환
- 서버에서 나온 응답 값을 LLM에게 다시 전달한다.
- 해당 응답 값을 통해서 LLM이 최종 답변을 만든다.
```json
// 내 백엔드가 다시 Claude API를 호출
{
  "messages": [
    {"role": "user", "content": "어제 시킨 맥북 언제 와?"},
    {"role": "assistant", "content": [/* Step 2에서 받은 내용 그대로 */]},
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01ABC123", // 아까 받은 작업 ID
          "content": "배송 상태: 오늘 오후 3시 배송 완료 예정" // 실제 DB 조회 결과
        }
      ]
    }
  ]
}
```

### 5. LLM -> 내 서버 : 최종 자연어 답변 반환
- LLM이 최종적으로 만든 결과값을 다시 서버에게 전달하고, 이를 서버는 클라이언트에게 처리하도록 한다.
```json
// Claude API의 최종 응답
{
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "조회해 본 결과, 주문하신 맥북은 **오늘 오후 3시에 배송이 완료될 예정**입니다! 조금만 더 기다려주세요. 😊"
    }
  ]
}
```


## 테스트

1. Anthropic API 키 발급
2. 가장 단순한 도구 1개(`get_current_time`)로 루프 코드 작성
3. tool_use → tool_result 핸드오프 정확히 동작하는지 확인
4. 도구 2~3개로 늘려서 Claude가 여러 도구를 골라 부르는지
5. **검증**: max_turns 도달, max_tokens 초과 케이스 모두 처리