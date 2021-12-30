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