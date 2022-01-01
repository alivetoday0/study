## Reactive Streams란?
- 참조 사이트 : https://github.com/reactive-streams/reactive-streams-jvm
- RxJava는 Reactive Streams의 인터페이스를 구현한 구현체
  - Publisher(생산자): 데이터 생성 및 통지
  - Subscriber(소비자): 통지된 데이터를 전달받아 처리
  - Subsription: 전달 받을 데이터의 개수를 요청하고 구독을 해지
  - Processor:  Publisher와 Subscriber 기능이 모두 있음
### 프로세스 흐름
- Subscriber가 publisher 데이터를 구독한다. (subscribe)
- Publisher가 subscriber에게 데이터를 통지할 준비가 되었음을 알린다. (onSubscibe 호출)
- subscriber는 publisher에게 전달 받을 통지 데이터 개수를 요청한다. (subscription request)
- 요청받은 개수만큼 데이터를 통지한다. (onNext)
- publisher가 데이토가 통지가 완료되었음을 알린다. (onComplete)
- 완료 전 에러가 발생할 경우 publisher는 에러를 통지한다. (onError)
### Cold Publisher & Hot Publisher
- Cold Publisher
  - 생산자는 소비자가 구독 할때마다 데이털르 처음부터 보낸다. 
  - 데이터를 통지하는 새로운 타임 라인이 생성된다. 
- Hot Publisher
  - 생산자는 소비자와 관계없이 데이터를 한 번만 통지한다. 
  - 데이터를 통지하는 타임 라인은 하나이다. 

## Flowable/Observable
|비교 유형|Flowable|Observable|
|--- |--- |--- |
|reactive streams<br/>interface 구현 여부| O | X|
|데이터처리|Subscriber|Observer|
|Back Pressure 기능 여부| O | X |
|데이터 개수 제어|O|X|
|구독해지|Subscription|Disposable|

- 예시
  - [Flowable](https://github.com/alivetoday0/rxjava-test/blob/master/src/main/java/com/alivetoday0/FlowableExam.java)
  - [Observable](https://github.com/alivetoday0/rxjava-test/blob/master/src/main/java/com/alivetoday0/ObservableExam.java)

### Back Pressure 란?
Flowable에서 데이터를 전달하는 속도가 Subscriber에서 받는 속도보다 빠를 때 밸런스를 맞추기 위해 전달량을 제어하는 기능이다. 위와 같이 경우 RxJava에서는 다음 에러를 발생한다. 
> io.reactivex.exceptions.MissingBackpressureException: Can't deliver value 128 due to lack of requests

### Back Pressure Strategy
- Missing 전략
  - Back Pressure를 사용 안 함.
  - onBackpressureXXX()로 배압 적용을 할 수 있다
- Error 전략 
  - 전달된 데이터가 버퍼의 크기를 초과하면 MissingBackpressureException 에러 발생
- [Buffer 전략 - DROP LATEST](https://github.com/alivetoday0/rxjava-test/blob/master/src/main/java/com/alivetoday0/BackPressureDropLatestStrategy.java)
  - 버퍼가 차면 가장 최근에 들어온 데이터를 DROP 한 후에 대기 데이터로 채운다.
- [Buffer 전략 - DROP OLDEST](https://github.com/alivetoday0/rxjava-test/blob/master/src/main/java/com/alivetoday0/BackPressureDropOldestStrategy.java)
  - 버퍼가 차면 가장 오래된 데이터를 DROP 한 후에 대기 데이털르 채운다. 
- [Drop 전략](https://github.com/alivetoday0/rxjava-test/blob/master/src/main/java/com/alivetoday0/BackPressureDropStrategy.java)
  - 버퍼에 데이터가 가득 찬 시점 이후로 들어오는 데이터는 버린다. 
  - 이 후 버퍼가 비워지는 시점에 버려지지 않는 데이터부터 버퍼에 담는다.
- [LATEST 전략](https://github.com/alivetoday0/rxjava-test/blob/master/src/main/java/com/alivetoday0/BackPressureLatestStrategy.java)
  - 대기 중인 데이터 중에서 가장 최근 데이터를 버퍼에 담는다.