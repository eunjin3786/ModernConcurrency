## Chapter 1: Why Modern Swift Concurrency?

### 배경

애플이 비동기식 프레임워크에 대해 빅딜을 한 것은 2009년 맥 OS X 스노우 레오파드와 함께 그랜드 센트럴 디스패치(GCD)가 나왔을 때였다.

GCD는 Swift가 2014년에 출시할 때 첫날부터 동시성과 비동기성을 지원하도록 도왔지만, 이러한 지원은 기본이 아니었다.

이는 Objective-C의 요구와 기능을 중심으로 설계되었다. 

Swift는 언어를 위해 특별히 설계된 자체 메커니즘을 가질 때까지 동시성을 "차용" (“borrowed”) 했습니다.

스위프트 5.5는 비동기식 동시 코드 작성을 위한 새로운 네이티브 모델을 도입하면서 모든 것이 바뀌었다.



### 특징

새로운 동시성 모델은 Swift에서 안전하고 성능이 뛰어난 프로그램을 작성하는 데 필요한 모든 것을 제공한다.

- A new, native syntax for running asynchronous operations in a structured way.
- A bundle of standard APIs to design asynchronous and concurrent code.
- Low-level changes in the libdispatch framework, which make all the high-level changes integrate directly into the operating system.
- A new level of compiler support for creating safe, concurrent code.



### 제약조건

- If you’re using Xcode 13.2 or newer, it will bundle the new concurrency runtime with your app so you can target iOS 13 and macOS 10.15 (for native apps).
- In case you’re on Xcode 13 but earlier version than 13.2, you’ll be able to target only iOS 15 or macOS 12 (or newer).



## Understanding asynchronous and concurrent code

#### Synchronous execution

> **When your code runs synchronously, the execution happens sequentially.**

대부분의 코드는 코드 편집기에서 볼 수 있는 것과 같은 방식으로 실행된다. (위에서 아래로)

이것은 주어진 코드 라인이 언제 실행되는지 쉽게 결정할 수 있게 한다. 

synchronous context 에서 코드는 단일 CPU 코어의 하나의 실행 스레드에서 실행된다.

1차선 도로의 자동차와 같은 동기식 기능을 상상할 수 있다. 각각의 차는 그 앞의 차선을 따라 달린다. 비록 근무 중인 구급차와 같이 한 대의 차량이 더 높은 우선 순위를 가지고 있다고 해도, 그것은 나머지 차량들을 "뛰어넘겨" 더 빨리 운전할 수 없다.

<img width="523" alt="스크린샷 2022-04-24 오후 5 52 39" src="https://user-images.githubusercontent.com/9502063/164968490-698e5909-3720-43de-a374-4670355754f6.png">



#### Asynchronous execution

'비동기 실행은 사용자 입력, 네트워크 연결 등과 같은 많은 다양한 이벤트에 따라 한 스레드에서 다른 프로그램 조각들이 임의의 순서로 실행될 수 있게 한다.

비동기식 컨텍스트에서는 특히 여러 비동기식 함수가 동일한 스레드를 사용해야 하는 경우 함수의 실행 순서를 정확하게 구분하기가 어렵다. 정지등이 있는 도로와 교통이 양보해야 하는 곳에서 운전하는 것처럼, 기능들은 때때로 계속하기 위해 차례가 될 때까지 기다리거나 심지어 계속하기 위해 녹색등이 나올 때까지 멈춰야 한다.



<img width="461" alt="스크린샷 2022-04-24 오후 5 54 53" src="https://user-images.githubusercontent.com/9502063/164968563-850dd18f-56ab-403b-87e4-217894a15586.png">



비동기 호출의 한 가지 예는 네트워크 요청을 하고 웹 서버가 응답할 때 실행할 completion closure 를 제공하는 것이다.

completion callback 이 실행되기를 기다라는 동안, 앱은 다른 일을 하기 위해 시간을 사용한다.

<img width="632" alt="스크린샷 2022-04-24 오후 5 58 53" src="https://user-images.githubusercontent.com/9502063/164968691-1b77cfa0-8359-488c-8b22-2bb93465cfe1.png">

#### Concurrent execution

의도적으로 프로그램을 병렬로 실행하기 위해서 concurrent API 를 사용한다.

어떤 API는 고정된 수의 작업 (task)를 동시에 실행하는 것을 지원한다. 

한편 다른 API는 concurrent group 을 시작하고 임의의 수의 동시 작업을 허용한다. 



이는 수많은 동시성 관련 문제를 야기한다. 예를 들어, 프로그램의 서로 다른 부분이 서로의 실행을 차단하거나, 두 개 이상의 fuction이 동시에 동일한 변수에 액세스하여 예기치 않게 앱 상태가 손상되는 매우 혐오스러운 data-race 를 만날 수 있다.

그러나 주의하여 사용하면 동시성을 통해 여러 CPU 코어에서 서로 다른 기능을 동시에 실행하여 프로그램을 더 빠르게 실행할 수 있다. 

마찬가지로 주의 깊은 드라이버도 다중 차선 고속도로에서 훨씬 더 빨리 이동할 수 있다.



<img width="588" alt="스크린샷 2022-04-24 오후 6 04 42" src="https://user-images.githubusercontent.com/9502063/164968919-c2e49bda-2fde-4320-95bb-e505590f5821.png">



다중 차선은 더 빠른 차들이 더 느린 차들을 돌아 다닐 수 있게 해준다. 

더욱 중요한 것은, 구급차나 소방차와 같은 우선순위가 높은 차량들을 위해 비상 차선을 자유롭게 유지할 수 있다는 것이다.

마찬가지로, 코드를 실행할 때 우선 순위가 높은 태스크는 우선 순위가 낮은 태스크보다 먼저 대기열을 "점프"할 수 있으므로, 메인 스레드를 차단하지 않고 UI에 대한 중요한 업데이트를 위해 사용 가능한 상태로 유지할 수 있다. 



실제 사용 사례는 사진 브라우징 앱으로, 웹 서버에서 이미지 그룹을 동시에 다운로드하여 썸네일 크기로 축소하고 cache 캐시에 저장해야한다. 

<img width="668" alt="스크린샷 2022-04-24 오후 6 07 41" src="https://user-images.githubusercontent.com/9502063/164969038-e7f4f425-9d66-41db-b077-98ff62f6b17e.png">



비동기성과 동시성 (asynchrony and concurrency) 이 모두 훌륭해 보이지만, 스스로에게 물어보면 다음과 같다: 

"Swift는 왜 새로운 동시성 모델이 필요했을까요?" 이전에 위에서 설명한 기능 중 적어도 일부를 사용한 앱에서 작업한 적이 있을 것이다.

다음으로, Swift 5.5 이전의 동시성 옵션을 검토하고 새로운 비동기/대기 모델의 어떤 점이 다른지 알아보자. 



## Reviewing the existing concurrency options

Swift 5.5 이전 에서는 GCD를 사용하여 dispatch queue (스레드에 대한 추상화) 를 통해 비동기 코드를 실행했다. 

Operation, Thread 또는 C-based pthread library 같이 더 오래된 API도 사용했다. 



> 새로운 Swift 동시성 API가 GCD를 대체했기 때문에 이 책에서는 GCD를 사용하지 않을 것입니다
>
> 궁금하면 Apple의 GCD 설명서를 읽어 보십시오. 디스패치 문서 (https://apple.co/3tOlEuO).



이러한 API들은 모두 동일한 기반을 사용한다. POSIX 스레드, 즉 주어진 프로그래밍 언어에 의존하지 않는 표준화된 실행 모델입니다. 각 실행 흐름은 스레드이며, 위에서 설명한 다차선 차량의 예와 유사하게 여러 스레드가 겹쳐 동시에 실행될 수 있다. 



Operation과 Thread 같은 Thread wrapper는 실행을 수동으로 관리해야한다. 

즉, 스레드를 생성 및 폐기하고 동시 작업의 실행 순서를 결정하고 스레드에 걸쳐 공유 데이터를 동기화해야한다. 이것은 오류가 발생하기 쉽고 지루한 작업입니다



GCD의 queue-based model은 잘동작하나 종종 다음과 같은 문제를 일으킬 수 있다. 



- Thread explosion (스레드 폭발):
  너무 많은 동시 스레드를 만들려면 활성 스레드 간에 지속적으로 전환해야 합니다. 이렇게 하면 앱 속도가 느려진다.

- Priority inversion (우선순위 반전):

  임의의 low-priority task는 같은 queue에서 기다리고 있는 high-priority task 의 실행을 차단한다. 

- Lack of execution hierarchy (실행 계층이 없음):
  비동기 코드 블록에는 실행 계층이 없으므로 각 작업이 독립적으로 관리되었다. 이로 인해 실행 중인 작업을 취소하거나 액세스하는 것이 어려워졌다. 그것은 또한 작업이 호출자(caller)에게 결과를 반환하는 것을 복잡하게 만들었다.



이러한 단점을 해결하기 위해 Swift는 완전히 새로운 동시성 모델을 도입했다.

다음으로, Swift의 현대 동시성이 무엇인지 알아보자! 



## Introducing the modern Swift concurrency mode

#### 1. A cooperative thread pool

새로운 모델은 스레드 풀을 투명하게 관리하여 사용 가능한 CPU 코어 수를 초과하지 않도록 한다.

이렇게 하면 런타임에서 스레드를 생성 및 삭제하거나 지속적으로 값비싼 스레드 전환을 수행할 필요가 없다. 

대신, 코드는 일시 중단될 수 있으며, 나중에 풀에서 사용 가능한 스레드에서 매우 빠르게 재개될 수 있다.



#### 2. async/await syntax

Swift의 새로운 async/await syntax는 컴파일러와 런타임에 코드 조각이 앞으로 한 번 이상 실행을 일시 중지하고 재개할 수 있음을 알려준다. 

런타임에서 이를 원활하게 처리하므로 스레드 및 코어에 대해 걱정할 필요가 없다.

멋진 보너스로서, 새로운 언어 구문은 escaping closure 를 콜백으로 사용할 필요가 없기 때문에 self 또는 다른 변수를 weakly or strongly capture 할 필요가 없다. 



#### 3. Structured concurrency

각 비동기 작업은 이제 상위 작업(parent task)과 지정된 실행 우선 순위를 가진 계층(hierarchy)의 일부가 되었다. 

이 계층을 사용하면 부모가 취소될 때 런타임에서 모든 하위 작업을 취소할 수 있다. 

더나아가 모든 하위 항목이 완료될 때까지 기다린 후 상위 항목이 완료될 수 있다. 사방이 꽉 막힌 배 같다. 

이 계층은 계층에서 우선순위가 낮은 작업보다 우선순위가 높은 작업이 먼저 실행되는 큰 이점 및 보다 분명한 결과를 제공합니다.



####  4. Context-aware code compilation

컴파일러는 주어진 코드 조각이 비동기적으로 실행될 수 있는지 여부를 추적한다. 

이 경우  mutating shared state과 같은 잠재적으로 안전하지 않은 코드를 작성할 수 없다.



이 높은 수준의 컴파일러 인식은 actor 같은 정교한 새로운 기능을 가능하게 한다.

actor는 컴파일 시 상태에 대한 동기식 액세스와 비동기식 액세스를 구별하고 안전하지 않은 코드를 쓰는 것을 더 어렵게 만들어 의도하지 않게 손상된 데이터를 보호한다.



이러한 모든 이점을 염두에 두고 새로운 동시성 기능이 포함된 코드를 작성하는 작업으로 바로 이동하여 어떤 느낌인지 직접 확인해보자! 
