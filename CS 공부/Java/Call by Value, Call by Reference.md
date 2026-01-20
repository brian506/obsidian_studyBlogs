자바의 데이터형은 2가지로 나뉜다.
**기본형 (Primitive Type)** : boolean, NumericType (short, int, long, float 등)
**참조형 (Reference Type)** : ClassType, Interface Type, Array Type, Enum Type 등
## Call by Value

: 값에 의한 호출을 의미한다.
- 함수에 값을 전달할 때 **값을 복사해서 전달**하고, 전달한 값을 **변경해도 원본은 변경되지 않는다.**
## Call by Reference

: 참조에 의한 호출을 의미한다.
함수에 값을 전달할 때 참조 값을 전달한다.

## 코드로 이해하기

```java
public class Main {
	public static void main(String[] args) {
		int var = 1;
		int[] arr = {1};
				
		add_value(var); // 변수 자체를 보냄
		System.out.println(var); // 1: 값 변화가 없음
		
		add_reference(arr); // 배열 자체를 보냄
		System.out.println(arr[0]); // 101 : 값 변화
		
	}
	static void add_value(int var_arg) {
		var_arg += 100;
	}
	static void add_reference(int[] arr_arg) {
		arr_arg[0] += 100;
	}
}
```

변수를 **기본형으로 보냈을 때** 해당 함수 안의 로직이 동작하지 않고 **값 변화가 일어나지 않았다.**
반면에 변수를 **참조형으로 보냈을** 때는 해당 함수 안의 로직이 동작하여 **값 덧셈이 정상적으로 일어났다.**

똑같이 두 변수를 메서드의 입력값으로 보냈지만, 결과가 다른 이유는 자체 값의 복사(call by value)와 값의 주소 참조(call by reference)가 일어났기 때문이다.

## 기본형, 참조형 call by value / reference 과정

### 동작 순서

**1) 변수 초기화** : 
- main 스택 프레임에 기본형 타입인 `var`은 그대로 원시값 1을 가지고,
- 참조형 타입인 `arr` 의 **실제 데이터는 Heap 영역**에 저장되고,
- 참조할 **주소값은 Stack 영역**에 저장된다.
   
**2) 기본형 call by value 과정** : 
- 아래와 그림과 같이 `add_value` 메서드 내부의 지역변수인 `var_arg` 변수가 **새롭게 저장**된다. 
- 이때 `add_value` 가 호출되면 `var_arg` 변수값이 100이 더해져 101이 된다.
- 여기서 더해지는 것은 `var_arg` 변수값이지, `var` 값이 아니다.
	- `var` 변수와 `var_arg` 변수는 다른 변수이다.
- 이후에 `add_value`메서드가 종료되면 해당 Stack 영역에 있는 `var_arg` 프레임은 삭제된다.
- 그리하여 결과를 출력하면 Stack 영역에 아직 남아있는 `var = 1` 의 스택 프레임이 찍히는것이다.
![](../../images/스크린샷%202026-01-20%2015.31.47.png)

**3) 참조형 call by refrence 과정** 
- 아래 그림과 같이 `add_reference` 메서드가 호출되면서, `add_reference` 스택 프레임이 생성되고 그 안에 지역변수 `arr_arg`가 생성된다.
- 여기서 주의할 것은 지역변수 `arr_arg` 가 **`arr` 배열의 주소값**을 가지고 있다.메서드가 실행될때 해당 **참조형의 주소값**을 가지고 수행하게 되므로 해당 메소드가 종료되고 나서도 Heap 영역에 있는 값이 변하게 되는 것이다.

![](../../images/스크린샷%202026-01-20%2015.47.35.png)

## 자바에는 call by reference 개념이 없다.

위에서 본 것처럼 자바에도 call by reference 개념이 있다고 생각할 것이다.
하지만 자바에서는 C와 달리 포인터를 철저하게 숨겨 개발자가 직접 메모리 주소에 접근 하지 못하게 조치했기 때문이다.

위의 예시에서 보면 두 배열 변수 `arr_arg` 와 `arr` 은 같은 주소를 가지고 있을뿐, 서로 다른 변수이고 별도로 존재한다.
`arr_arg`는 **`arr` 의 주소를 가진 새로운 변수**를 선언하는 것이므로 단순히 **주소값의 복사**이다.
즉, 각 변수는 서로 다른 scop 에 존재한다.

결론적으로 자바에서는 call by value 로만 작동하며, **원시값이 복사 되느냐, 주소값이 복사되느냐** 차이만 있다.

