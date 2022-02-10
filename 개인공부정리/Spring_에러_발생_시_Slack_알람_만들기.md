# Spring 에러 발생 시, Slack 알람 만들기

지금 진행하고 있는 프로젝트 중 환자의 유전자 데이터를 분석하여 관련 데이터를 제공하는 기능이 있다. 

사용자 요청이 들어오면, 분석 서버로 분석을 요청하고, 그 후 작업은 비동기로 처리하고 있는데,  서버로 올렸을 경우 에러가 언제 발생했는지 모르고 그냥 지나치는 경우가 종종 있었다!  이것 말고도, 사용자가 사용했을 때, 예상하지 못했던 에러가 발생했어도 모르고 지나가는 일도 있었고.. 

무튼, 여러모로 에러가 발생하면 알림오면 좋겠다고 생각을 했었다. 

2가지 방법으로 테스트를 했었고, 이에대한 기록을 남기고자한다!



## 1. logback-slack-appender (라이브러리) -> 사용X

Spring에 관련 기능이 있을까 하여 검색해보니 가장 대표적으로 사용하는 것이 logback-slack-appender 라이브러리 였다. 

logback에서 error 로그와 관련한 부분이 생기면 이벤트를 발생시켜 사용하는 방법인 것 같다.

### 적용 방법 

기본적인 사용 방법은 아래 링크의 블로그를 참고하여 개발하였다.

https://velog.io/@haerong22/Spring-%EC%8A%AC%EB%9E%99%EC%97%90-%EB%A1%9C%EA%B7%B8-%EB%82%A8%EA%B8%B0%EA%B8%B0 

에러 알림을 만든다고 하였을 때, alpha 서버의 에러만 받도록 해달라는 팀장님의 요청이 있었고, 어찌어찌 alpha 서버 관련한 profile만 적용할 수 있도록 수정하였다. 

```xml
<!--알림이 필요한 profile-->	
<springProfile name="alpha">
    <property resource="logback-alpha.yml"/>
    <appender name="SLACK_ERROR" class="com.github.maricn.logback.SlackAppender">
      <webhookUri>${webhook-uri}</webhookUri>
      <channel>#${channel}</channel>
        <layout class="ch.qos.logback.classic.PatternLayout">
          <pattern>*${LOG_PATTERN}*%n</pattern>
        </layout>
      <username>${username}</username>
      <iconEmoji>:${emoji}:</iconEmoji>
      <colorCoding>true</colorCoding>
    </appender>
    <appender name="ASYNC_SLACK_ERROR" class="ch.qos.logback.classic.AsyncAppender">
      <appender-ref ref="SLACK_ERROR"/>
      <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>ERROR</level>
      </filter>
    </appender>
  </springProfile>

<!--알림이 필요하지 않는 profile-->	
  <springProfile name="beta">
    <property resource="logback-beta.yml"/>
  </springProfile>
```



#### 후기 및 장, 단점

회사에서 서버를 3개를 쓰고 있는데, 무사히 alpha서버만 에러를 띄우는데 성공했다! 

하지만 성공하고 에러를 보니, 장점보단 단점이 너무 명확하게 보였다.. 



일단, 위에 logback-spring.xml 처럼 profile 별로 나눠 사용하는 것은 좋았으나,  ${username} 같은 관련 설정은 알람 기능을 사용하지 않는 beta 서버에도 작성을 해주어야 했었다. (logback-beta.yml 파일에.. 이건 내가 몰라서 그랬을지도... )

또한, 알림이 오는 것 까진 좋았지만 exception에 관련한 거의 모든 데이터가 넘어와 어디서 에러가 발생했는지 확인하기 어려웠다. 

마지막으로 확인하고 싶지 않은 에러까지 다 넘어왔다. 예를들면 회원이 비밀번호가 틀렸다.. 이런 에러까지.. 

장점은.. logback 기반이다보니, log.error() 코드로 작성한 에러도 잡힌다는 점..? 



아무튼, 이런 로그는 받아봤자 별로일 것 같아서 다시 관련된 사항을 검색해 보았고, 

우아한 형제 테크코스에서 손너잘님이 만드셨다는 슬랙알람 기능을 보게되었다!



## 2. Custom slack alaram -> 사용

일단 사용 후기 작성 전, 손너잘님께 감사의 인사를 보냅니다..!

자세한 개발 방법은 손너잘님의 블로그를 참고!

https://bperhaps.tistory.com/entry/Spring-Slack-Alarm-Logger-%EB%A7%8C%EB%93%A4%EA%B8%B0 



사실 참고라고 할 것도 없이, 개발하신 코드를 복붙해서 사용했다 ^^.. 

손너잘님의 코드는 한번에 사용할 수 있을 정도로 너무 좋았고 깔끔했고 짱이었고 최고였고 다했지만.. 

원하는 부분이 조금(?) 달라 수정하여 사용하였다. 



수정 부분은 다음과 같다 

#### 1. 개발자가 정의한 Exception과 의도치 않은 Exception 구분

우리 프로젝트에서 Exception 전략을 https://cheese10yun.github.io/spring-guide-exception/ 를 참고하여 만들었다.. 

BusinessException은 개발자가 정의한 Exception이고, 그렇지 않은 경우는 그냥 Exception이 발생하게 된다. 

팀원들은 어떻게 생각할지 모르겠지만.. 그냥 Exception이 발생한 경우가 좀 더 급한(?) 상황이라고 생각하였고, 이 경우에는 슬랙 메세지에 :rotating_light:를 달아 보내도록 수정해 보았다!

사실 별건 없고, headerMessage를 정할 시,  `arg instanceof BusinessException ` 를 통해 구분만 해주었다.

#####  SlackMessageGenerator.java

```java
public class SlackMessageGenerator {
	
   //...
	private static final String EXCEPTION_MESSAGE_FORMAT = "`[%s] %s.%s:%d`\n`%s`";
	private static final String EXCEPTION_HEADER_MESSAGE = ":rotating_light: *[Exception 발생]* :rotating_light: \n";

	public String generate(ContentCachingRequestWrapper request, Object arg,
		SlackAlarmErrorLevel level) {

		try {
			String headerMessage =
				arg instanceof BusinessException ? BUSINESS_EXCEPTION_HEADER_MESSAGE : EXCEPTION_HEADER_MESSAGE;
		
            //후략...
	}

}

```



#### 2. 원하는 Exception만 알림 오게 하기 

프로젝트 상에서 모든 Exception들을 @ExceptionHandler로 지정해 놓지 않았기 때문에, 대부분의 Exception은 BusinessException으로 들어온다. 

이에 따른 문제점이 logback-slack-appender을 사용할 때와 같이 대부분의 에러가 넘어온다는 것 이었다! (사용자 패스워드 틀림과 같은.. ) 그래서 이 부분을 나름대로 수정해보았다. 



Exception 전략에서 ErrorCode를 지정하여 메세지나 상태 코드 등을 지정하는 부분이 있었는데, 커스텀하게 만든 Exception들은 각자의 ErrorCode를 가지도록 개발하였다. 

예를 들면,  아래 코드와 같이 구성한다! 자세한건 위의 블로그.. 

간단히 설명하면, 로직 중 UserHasNotAuthException를 발생 시키면 @ExceptionHandler가 BusinessException을 받아 응답하는 구조이다. 

```java
public class UserHasNotAuthException extends BusinessException {
  public UserHasNotAuthException() {
    super(ErrorCode.USER_HAS_NOT_EXCEL_AUTH);
  }
}

public enum ErrorCode {
  USER_HAS_NOT_EXCEL_AUTH(403, "P001", "파일 다운로드 권한이 없습니다."),
  //...
}
```



여기서 ErrorCode ENUM에 `alamable` 값을 추가하여 원하는 에러만 알람을 받을 수 있도록 수정해 보았다. 

##### ErrorCode.java

```java
public enum ErrorCode {
  USER_HAS_NOT_EXCEL_AUTH(403, "P001", "파일 다운로드 권한이 없습니다.", true),
  OTHER_ERROR(500, "A001", "...", false),
  //...중략
   private final boolean alamable;
   public boolean isAlamable(){return alamable;}
}
```

##### ExceptionAppender.java

```java
//...
public void appendExceptionToResponseBody(JoinPoint joinPoint) {
    //...

    if(args[0] instanceof BusinessException){

        BusinessException exception= (BusinessException)args[0];

        if(!exception.getErrorCode().isAlamable()){
            return;
        }
    }
    //...
}
```

ExceptionAppender에 위의 코드 if문만 추가해 주었다. 

BusinessException 타입인 경우 isAlamable() 함수를 통해 알람을 보낼지 말지 결정한다. 



#### 3. 비동기로 동작하는 부분 알림 설정 

사실 내가 알람 기능을 개발한 이유가 이 비동기로 작동하는 부분 때문인데, 비동기 처리시 발생하는 알림은 오지 않았다. 

그러다가 비동기 에러는 따로 설정을 해주어야 한다는 것을 깨닫고 관련 설정을 해주었다. 

##### AsyncConfig.java

```java
//...
public class AsyncConfig implements AsyncConfigurer {

  private final Slack slack;
  private final SlackMessageGenerator slackMessageGenerator;
 
    //....
    
  @Override
  public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return new CustomAsyncExceptionHandler(slack, slackMessageGenerator);
  }

}
```

위의 코드와 같이 getAsyncUncaughtExceptionHandler()를 Override하여 CustomAsyncExceptionHandler를 만들어 주어야 한다. 이 때, 알림 관련 bean들을 주입해 주었다.

##### CustomAsyncExceptionHandler.java 

```java
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

	private final Slack slack;
	private final SlackMessageGenerator slackMessageGenerator;

	public CustomAsyncExceptionHandler(Slack slack,
		SlackMessageGenerator slackMessageGenerator) {
		this.slack = slack;
		this.slackMessageGenerator = slackMessageGenerator;
	}

	@Override
	public void handleUncaughtException(
		Throwable throwable, Method method, Object... obj) {

		String message = slackMessageGenerator.generateAsync(throwable,method);
		slack.send(message);
		throwable.printStackTrace();
		
	}

}
```

CustomAsyncExceptionHandler에서 비동기 에러가 발생할 시, 메세지를 정의하고 슬랙 알람을 보냈다. 

기존에 사용하던 `generate`메소드와는 다른 부분이 있어 `generateAsync`를 새로 만들어서 사용하였다. 

##### SlackMessageGenerator.java

```java
public class SlackMessageGenerator {
//...
  public String generateAsync(Throwable throwable, Method method) {

		try {
			String headerMessage = "*[Async Exception 발생]*\n";
			String currentTime = getCurrentTime();
			String userId = getUserId();
			String ip = "unknown";
			String errorLog = extractMessage((Exception)throwable, SlackAlarmErrorLevel.ERROR);
			String methodStr = "METHOD";
			String requestURI = method.getName();

			return toMessage(headerMessage,currentTime, userId,ip, errorLog, methodStr, requestURI);
		} catch (Exception e) {
			return String.format(EXTRACTION_ERROR_MESSAGE, e.getMessage());
		}
	}

//...
}
```

동기 통신이었다면, api 경로가 출력됐을텐데, 비동기라 메소드 이름 출력으로 대신하였다. 



#### 4. 기타 자잘한 사항.. 

알람 기능 개발 중 접속자 IP도 확인할 수 있으면 좋을 것 같다는 팀장님의 요청에 간단히 한 줄 추가하였다. 

비동기 통신은 ip를 확인 할 수 없어 unknown으로 대신하였다.. 

##### SlackMessageGenerator.java

```java
public String generate(ContentCachingRequestWrapper request, Object arg,
		SlackAlarmErrorLevel level) {
    	//...
			String ip = request.getRemoteAddr();
    	//... 
	}
```



*덧, 손너잘님의 코드는  Exception을 처리하는 로직이라 error.log는 잡지 못하는 경우도 처리하고 싶었지만... 이 부분은 과감히 포기했다... 좀 더 생각해 봐야 할 것 같다...* 



## 본격 후기

일단 슬랙 알람 만든 나에게 박수 :clap::clap::clap::clap:

좋은 코드 제공해주신 손너잘님 마지막으로 감사합니다 :)

손너잘님의 글을 보면서 spring을 깊게깊게 공부해야겠다는 생각이 빡 들었다. 

난 너무 겉핥기야.. 

조금 더 공부한다면 에러 알림 시 조금씩 발생하는 에러를 쉽게 고칠 수 있지 않을까 생각한다!

~~(request에 가끔 null이 들어오고 이런 문제들... 사실 scope의 문제인걸 알고있지만.. 안되는걸.. )~~

가끔 오는 에러 알림마다 뭔가 반가우면서 조금 짜증나지만 ^^.. 

즐겁게 발생하는 에러를 수정해야겠다! 후기 끝



