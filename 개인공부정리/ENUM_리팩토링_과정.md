# ENUM

업무중에  enum을 사용해서 타입을 정한 부분이 있었다. 

2020년 12월쯤 짰던 코드 같은데, 만들 당시에도 switch문이 과도하게 들어가고  list에 add를 반복하여 사용하는 등 만족스럽지 못한 코드였다.

물론 그당시에도 enum을 썼었다. ~~그런데 switch문이.. 2개나.. ㅠㅠ..~~



이번에 다시 이 기능을 손보며 불만족 스러운 코드를 바꿔보았다. 



### 기존코드

기존 코드를 그대로 가져올 수 없어서 enum 의 대표 예제인 계산기와 내 코드에서 발생한 문제를 조합하여 예시를 만들어보았다.



계산기를 기본과 공학용으로 나눠  BASIC타입에는 덧셈, 뺄셈 ENGINEERING에는 덧셈, 뺄셈, 곱셈, 나눗셈이 있도록 만들었다.

각 계산의 기호도 베이직과 공학용으로 나눠 넣었다. 기본형식에는 곱셈과 나눗셈이 없으니 null로 처리하고.. 

~~사실 이 코드도 한번 수정을 거친 것이다..^^..~~

```java
@Getter
public enum Operator {

    SUM("+","++"),
    SUBTRACT("-","--"),
    MULTIPLY(null,"X"),
    DIVISION(null,"/"),
    ;

    private String basicOperator;
    private String engineeringOperator;


    Operator(String basicOperator, String engineeringOperator) {
        this.basicOperator = basicOperator;
        this.engineeringOperator = engineeringOperator;
    }
}

```



```java
@Getter
public enum CalculatorType {

    BASIC(basicCalculator()),
    ENGINEERING(engineeringCalculator()),
    ;

    private List<Operator> operatorList;

    CalculatorType(List<Operator> operatorList) {
        this.operatorList = operatorList;
    }

    private static List<Operator> engineeringCalculator() {

        List<Operator> list = new ArrayList<>();

        list.add(Operator.SUM);
        list.add(Operator.SUBTRACT);
        list.add(Operator.MULTIPLY);
        list.add(Operator.DIVISION);

        return list;
    }

    private static List<Operator> basicCalculator() {
        List<Operator> list = new ArrayList<>();

        list.add(Operator.SUM);
        list.add(Operator.SUBTRACT);

        return list;
    }


}
```



### 기존코드의 문제점

각 연산에서 모든 결과 값을 가져오는 코드에서, 계산기의 타입별로 삼항연산자가 계속 들어가는 문제점이 있었다. 

또한,  기본계산기는 valuse에 값이 2개지만 공학용 계산기는 값이 3개가 들어오고, ++ 연산 등의 다른점이 있어 Calculator의 값을 구분해서 로직을 만들었어야 했다.

```java
public class Calculation {

      public List<Integer> getCalculateValues(CalculatorType calculatorType, Operator operator, List<Integer> values)
          throws Exception {

        List<Integer> results = new ArrayList<>();
        
        for (Operator op : calculatorType.getOperatorList()) {
           Integer result =  calculatorType == CalculatorType.BASIC ?  //계산기의 타입 구분을 위해 삼항연산자 
                    getBasicValue(op, values)
                    : getEngineeringValue(op, values);
           
           results.add(result);
        }
        
        return results;

    }

    private Integer getEngineeringValue(Operator operator, List<Integer> values) throws Exception {
        Integer value = 0;
        switch (operator) {
            case SUM:
                value = values.get(0) + values.get(1) + values.get(2);
                return value++;
            case SUBTRACT:
                value = values.get(0) - values.get(1) - values.get(2);
                return value--;
            case MULTIPLY:
                value = values.get(0) * values.get(1) * values.get(2);
                return value;
            case DIVISION:
                value = values.get(0) / values.get(1) / values.get(2);
                return value;
            default:
                throw new Exception("올바른 연산자가 아닙니다.");
        }
    }

    private Integer getBasicValue(Operator operator, List<Integer> values) throws Exception {
        Integer value = 0;
        switch (operator) {
            case SUM:
                value = values.get(0) + values.get(1);
                return value;
            case SUBTRACT:
                value = values.get(0) - values.get(1);
                return value;
            default:
                throw new Exception("올바른 연산자가 아닙니다.");
        }
    }

}


```



## 해결방법

 이렇게 switch문과 삼항연산자가 반복되는것이 별로 좋지 못한 코드라는 것을 깨닫고 enum에 각 타입별로 함수같은 것을 정의할 수 있는지 찾아보았다. 

 결론은 **확장 가능한 enum을 만들어야 한다면 인터페이스를 이용하라** 라는 것이다. *(enum 확장에 관한 자료는 밑에 참고자료를 활용해주세요)*



java에서 enum의 타입은 클래스이고,  java.lang.Enum을 부모로 가지고 있다.  

<u>java는 다중상속이 불가능</u>함으로 enum을 확장을 시키기 위해서는 인터페이스를 활용하여 확장시켜야 한다.



#### 그래서 저는 우선.. 

그래서 우선 Operator라는 인터페이스를 생성하고, basic과 engineering을 두 개의 enum으로 나누기로 하였다. 

그리고 공통된 기능을 Operator 인터페이스에 정의하고 각 enum에 구현하였다. 



``` JAVA
public interface Operator {
    String getOperatorValue();

    Integer getCalculatedValue(List<Integer>values);
}

```

```java
public enum BasicOperator implements Operator {

    SUM("+") {
        @Override
        public Integer getCalculatedValue(List<Integer> values) {
            return values.get(0) + values.get(1);
        }
    },
    SUBTRACT("-") {
        @Override
        public Integer getCalculatedValue(List<Integer> values) {
            return values.get(0) - values.get(1);
        }
    },
    ;

    private String operatorValue;

    BasicOperator(String operatorValue) {
        this.operatorValue = operatorValue;
    }

    @Override
    public String getOperatorValue() {
        return this.operatorValue;
    }
}

```

```java

public enum EngineeringOperator implements Operator {

    SUM("++") {
        @Override
        public Integer getCalculatedValue(List<Integer> values) {
            int value = values.get(0) + values.get(1) + values.get(2);
            return value++;
        }
    },
    SUBTRACT("--") {
        @Override
        public Integer getCalculatedValue(List<Integer> values) {
            int value = values.get(0) - values.get(1) - values.get(2);
            return value--;
        }
    },
    MULTIPLY("*") {
        @Override
        public Integer getCalculatedValue(List<Integer> values) {
            return values.get(0) * values.get(1) * values.get(2);
        }
    },
    DIVISION("/") {
        @Override
        public Integer getCalculatedValue(List<Integer> values) {
            return values.get(0) / values.get(1) / values.get(2);
        }
    },
    ;

    private String operatorValue;

    EngineeringOperator(String operatorValue) {
        this.operatorValue = operatorValue;
    }


    @Override
    public String getOperatorValue() {
        return this.operatorValue;
    }
}

```



그리고 이를 계산하는 함수를 다시 만들어 보았다.

```java
public class Calculation {
    public List<Integer> getCalculateValue(CalculatorType calculatorType, List<Integer> values) throws Exception {

        List<Integer> results = new ArrayList<>();

        for (Operator op : calculatorType.getOperatorList()) {
            results.add(op.getCalculatedValue(values));
        }
        return results;

    }
}

```



기존 코드에서는 각 타입별로 switch case문이 나오고, 삼항연산자도 나오고 하던 코드가 단 몇 줄로 깔끔해졌다!

기본 계산기와 공학 계산기의 타입별로 enum 한 곳에서 관리하니 코드도 더 좋은 느낌이 든다. 



# 공부를 마치고

기존의 코드와 마찬가지로, 지금의 코드도 사실 잘 짠 것 같지는 않다. 

그래도 enum이 이런식으로 확장이 가능하다는 사실을 공부하게 되었다는 사실에 만족한다.

다음에도 이런 개발 기회가 주어진다면 조금더 공부해서 더 잘 짠 코드를 만들어야겠다. ^^ 



# 참고 블로그

https://onsil-thegreenhouse.github.io/programming/java/2017/11/25/java_tutorial_1-13_enum/

http://wonwoo.ml/index.php/post/836

https://johngrib.github.io/wiki/java-enum/