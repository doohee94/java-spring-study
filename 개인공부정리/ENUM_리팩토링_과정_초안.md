# ENUM

업무중에  enum을 사용해서 타입을 정한 부분이 있었다. 

2020년 12월쯤 짰던 코드 같은데, 만들 당시에도 switch문이 과도하게 들어가고  list에 add를 반복하여 사용하는 등 만족스럽지 못한 코드였다.

물론 그당시에도 enum을 썼었다. ~~그런데 switch문이.. 2개나.. ㅠㅠ..~~



이번에 다시 이 기능을 손보며 불만족 스러운 코드를 바꿔보았다. 



### 기존코드

기존 코드를 그대로 가져올 수 없어서 계산기로 예시를 만들어보았다.



계산기를 기본과 공학용으로 나눠  BASIC타입에는 덧셈, 뺄셈 ENGINEERING에는 덧셈, 뺄셈, 곱셈, 나눗셈이 있도록 만들었다.

각 계산의 기호도 베이직과 공학용으로 나눠 넣었다. 기본형식에는 곱셈과 나눗셈이 없으니 null로 처리하고.. 

~~사실 이 코드도 한번 수정을 거친 것이다..^^..~~

```java
@Getter
public enum Calculation {

    SUM("+","+"),
    SUBTRACT("-","-"),
    MULTIPLY(null,"X"),
    DIVISION(null,"/"),
    ;

    private String basicSign;
    private String engineeringSign;


    Calculation(String basicSign, String engineeringSign) {
        this.basicSign = basicSign;
        this.engineeringSign = engineeringSign;
    }
}
```



```java
@Getter
public enum Calculator {

    BASIC(basicCalculator()),
    ENGINEERING(engineeringCalculator()),
    ;

    private List<Calculation> calculationList;

    Calculator(List<Calculation> calculationList) {
        this.calculationList = calculationList;
    }

    private static List<Calculation> engineeringCalculator() {

        List<Calculation> list = new ArrayList<>();

        list.add(Calculator.SUM);
        list.add(Calculator.SUBTRACT);
        list.add(Calculator.MULTIPLY);
        list.add(Calculator.DIVISION);

        return list;
    }

    private static List<Calculation> basicCalculator() {
        List<Calculation> list = new ArrayList<>();

        list.add(Calculator.SUM);
        list.add(Calculator.SUBTRACT);

        return list;
    }


}
```



### 기존코드의 문제점

기본계산기는 valuse에 값이 2개지만 공학용 계산기는 값이 3개가 들어오는 등의 다른점이 있어 Calculator의 값을 구분해서 로직을 만들었어야 했다.

```java
public class CalculateMain {

    public Integer getCalculateValue(Calculator calculator, Calculation calculation, List<Integer> values) {

        return calculator == Calculator.BASIC ? 
                getBasicValue(calculation, values) 
                : getEngineeringValue(calculation, values);

    }

    private Integer getEngineeringValue(Calculation calculation, List<Integer> values) {
        Integer value = 0;
        switch (calculation) {
            case SUM:
                value = values.get(0) + values.get(1);
                break;
                
                // 생략
                
        }
        return value;
    }

    private Integer getBasicValue(Calculation calculation, List<Integer> values) {
        Integer value = 0;
        switch (calculation) {
            case SUM:
                value = values.get(0) + values.get(1) + values.get(2);
                
			 // 생략
        }
        return value;
    }

}

```

이런 문제 뿐만 아니라 "1 + 1 " 이런 계산식을 출력할 때에도 타입별로 가져와야하는 단점이 있었다.



## 해결방법

 