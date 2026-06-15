
Kafka 의 Consumer Lag, 파티션 오프셋, 토픽/파티션 개수를 매트릭으로 받아오기 위해 `kafka-exporter` 의 별도 컨테이너를 두고, Alloy 에서 매트릭을 수집해야 한다.

Lag 은 특정 컨슈머 그룹이 얼마나 메시지를 못 읽고 있는지에 대한 지표이다.

CPU/메모리 사용량, 요청 지연 시간 등을 알려면 별도로 JMX Exporter 도 띄워서 매트릭을 수집해야 되지만,
일단 `kafka-exporter` 로만 받아볼 것이다.
### config.alloy

```xml
// Kafka 매트릭 수집 설정  
prometheus.scrape "kafka_exporter" {  
    targets = [{  
    __address__ = "kafka-exporter:9308",  
    }]  
  
    scrape_interval = "15s"  
    forward_to = [prometheus.remote_write.default.receiver]  
  
}
```


