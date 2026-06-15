## 람다 표현식

자바의 메서드를 간격한 함수 식으로 표현한 것이다.

참고 : [람다 표현식과 함수형 인터페이스](람다%20표현식과%20함수형%20인터페이스.md) 

## 함수형 인터페이스  (Functional Interface)

추상 메서드가 오직 하나인 인터페이스를 의미한다.
여러 개의 디폴트 메소드와 static 메소드가 있어도 함수형 인터페이스로 취급한다.
함수형 인터페이스는 **단일 추상 메서드**를 가지는 인터페이스를 말한다.

## 디폴트 메소드 (Default Method)

인터페이스 내에서 구현이 가능한 메소드를 의미한다.
static 메소드도 추가로 구현이 가능하다.

이전 버전에서는 인터페이스 내에서 추상메소드만을 선언을 할 수 있었지만, static, default 메소드를 구현할 수 있게 되었다.

## 스트림 (Stream)


## 옵셔널 Optional

## JVM 의 변화

### Java7 JVM
![](../../images/스크린샷%202026-01-22%2016.51.51.png)
기존의 Java7에서는 Permanent Generation이 Non Heap Area에 있었다.

Permanent 영역에 저장되어 있던 정보
- Class의 메타 데이터 (바이트코드 포함)
- Method의 메타 데이터
- static 객체, static 상수
- 상수화된 String Object
- Class와 관련되 배열 객체 메타데이터
- JVM 내부적인 객체들과 JIT 의 최적화 정보

이렇게 많은 것들이 Permanent Generation 에 있다보니 OOM 이 발생했다.

### Java8 JVM
![](../../images/스크린샷%202026-01-22%2016.55.13.png)

Java 8 부터는 Permanent Generation 영역을 삭제하고 **Metaspace 영역을 추가해 Native 메모리 영역으로 이동**시켰다.  

기존의 Permanent 영역에 저장되어 있던 정보들의 이동 위치

- Class의 메타 데이터 (바이트코드 포함) -> Metaspace 로 이동
- Method의 메타 데이터 -> Metaspace로 이동
- static 객체, static 상수 -> Heap으로 이동
- 상수화된 String Object -> Heap으로 이동
- Class와 관련되 배열 객체 메타데이터 -> Metaspace로 이동
- JVM 내부적인 객체들과 JIT 의 최적화 정보 -> Metaspace로 이동

Heap에 저장되는 데이터는 GC의 대상이 될수 있게 되었고 Navtive 영역은 OS레벨에서 관리해 자동으로 크기를 조절하고, 개발자가 메모리 영역확보의 상한을 크게 인식할 필요가 없게 되었다.