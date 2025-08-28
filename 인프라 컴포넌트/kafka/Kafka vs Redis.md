#kafka #redis

### Redis

- **싱글스레드 처리 방식**  
- **병렬 처리 지원 미흡**  
  - RPOP 등을 병렬 호출해 별도 구조 설계 필요  
  - 병렬 처리 시 순서 보장 불가, 중복 처리 위험  
- **Pub/Sub 모델 제약**  
  - Subscribe된 consumer만 메시지 수신  
  - 동일 알림을 여러 consumer에 전달하면 각자 중복 처리 가능  
- **메시지 유실 위험**  
  - Redis List/Stream 활용 시에도 설정에 따라 메시지 손실  
  - 결제 실패 시 메시지를 잃어버리면 별도 재큐(re-queue) 로직 필요  

### Kafka

- **분산형 아키텍처 & 병렬 처리**  
  - 다중 파티션(Partition)으로 자연스럽게 병렬 처리  
- **Consumer Group**  
  - 그룹 내 각 consumer가 고유 파티션 처리 → 중복 처리 방지  
- **영속성(Durability) 보장**  
  - 디스크 기반 로그 저장 → 커밋 전까지 메시지 보관, 장애 시 재전송 가능  
- **순서 보장(Ordering Guarantee)**  
  - 파티션 내 오프셋(offset) 기준 처리로 메시지 순서 유지  
- **처리 보장(Processing Guarantees)**  
  - At-least-once, Exactly-once semantics(중복성 방지)지원  
  - Idempotent producer & Transactional API 제공  
- **확장성(Scalability) & 내결함성(Fault-tolerance)** 
  - 브로커·파티션 추가로 수평 확장  
  - 자동 리밸런싱, ISR 관리  
- **모니터링·운영 편의성**  
  - Consumer lag, Throughput, Broker 상태 지표 제공  
- **메시지 재처리·감사(Audit)**  
  - 로그 보존(retention) 설정으로 과거 이벤트 재처리 가능  

---

## 결제 로직에서 Kafka를 선택해야 하는 이유

1. **내구성 확보**  
   - 디스크 기반 저장으로 결제 요청 메시지 유실 방지  
2. **정확한 처리 보장**  
   - Exactly-once semantics로 중복·누락 방지  
   - 트랜잭션 API로 원자적 메시지 처리  
3. **순서 일관성 유지**  
   - 결제 승인 → 결제 완료 흐름에서 순서 보장  
4. **수평 확장 용이**  
   - 대규모 동시 결제 트래픽에도 안정적 처리  
5. **유연한 Consumer Group 설계**  
   - 승인, 취소, 알림 등 역할별 그룹 분리 → 책임 경계 명확  
6. **운영·모니터링 지원**  
   - 장애 시 자동 리밸런싱 및 실시간 지표 확인  
7. **감사(Audit) 요구 대응**  
   - 로그 보존으로 트랜잭션 이력 재검증 가능  
8. **금융 보안·컴플라이언스**  
   - TLS, ACL, RBAC 등 금융권 보안 요구사항 충족  

> **결론:**  
> Redis 기반 큐는 가벼운 비동기 작업에 적합하지만, **결제 시스템**처럼  
> “내구성 + 순서 보장 + 정확한 처리 보장”이 필수적인 워크로드에는  
> Kafka가 훨씬 안정적이고 관리하기 쉬운 선택입니다.  

producer - 예약 요청 url (서버)
consumer - 결제 요청(서버)
