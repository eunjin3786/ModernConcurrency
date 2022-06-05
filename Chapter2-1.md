# Chapter 2: Getting Started With async/await

## async/await 전에는..

Swift 5.5 전에는 비동기 코드 작성에 많은 단점이 있었다. 다음 예를 살펴보자.

```swift
class API {
  ...
  func fetchServerStatus(completion: @escaping (ServerStatus) -> Void) {
    URLSession.shared
      .dataTask(
        with: URL(string: "http://amazingserver.com/status")!
      ) { data, response, error in
        // Decoding, error handling, etc
        let serverStatus = ...
        completion(serverStatus)
      }
      .resume()
  }
}

class ViewController {
  let api = API()
  let viewModel = ViewModel()

  func viewDidAppear() {
    api.fetchServerStatus { [weak viewModel] status in
      guard let viewModel = viewModel else { return }
      viewModel.serverStatus = status
    }
  }
}
```

네트워크 API를 호출하고 그 결과를 뷰 모델의 property에 할당하는 짧은 코드 블록이다. 그것은 믿을 수 없을 정도로 단순하지만, 당신의 의도를 가리는 엄청난 양의 의식을 보여준다. 더 나쁜 것은, 코딩 오류를 위한 많은 공간을 만든다는 것이다: 오류를 확인하는 것을 잊었는가? 당신은 정말로 모든 코드 경로(every code path)에서 completion closure 를 호출했는가?

<br/>

Swift는 원래 Objective-C용으로 설계된 프레임워크인 GCD(Grand Central Dispatch)에 의존했었기 때문에 

처음부터 비동기식을 언어 설계에 긴밀하게 통합할 수 없었다.

<br/>

Objective-C 자체는 언어가 시작된 지 몇 년이 지난 후인 iOS 4.0에

블록(Swift closure의 병렬)을 도입했을 뿐이다. (??)

<br/>

잠시 시간을 내어 위의 코드를 검사해보자. 여러분은 다음과 같은 것을 알아차릴 수 있다.

- 컴파일러는 ` fetchServerStatus()` 내에서 너가 completion을 몇변 호출할 예정인 지 알 수 있는 명확한 방법이 없다. 

  따라서 수명(lifespan) 및 메모리 사용을 최적화할 수 없다

- 메모리 관리를 너 스스로 해야한다. (viewModel을 weakly capture 해서,,)
-  컴파일러는 너가 에러 처리를 했는 지 확인할 방법이 없다. 실제로, 너가 closure에서 error를 핸들링하는 것을 잊으거나 completion을 모두 호출하지 않으면, 메서드는 조용히 freeze 된다.. 

<br/>

Swift의 modern concurrency model은  컴파일러와 런타임 모두와 밀접하게 작동한다. 그것은 위에서 언급한 문제들을 포함하여 많은 문제들을 해결한다.

modern concurrency model은 세가지 툴을 제공한다. 

- **async**: 메서드 또는 함수가 비동기임을 나타낸다. 이를 사용하면 비동기 메서드가 결과를 리턴할 때 까지 실행을 일시중단 할 수 있다.
- **await**: async-annotated 메소드 또는 함수가 반환되기를 기다리는 동안, 코드가 실행을 일시 중지할 수 있음을 나타낸다. 
- **Task**: 비동기 작업의 단위(A unit of asynchronous work).  task 가 완료될 때까지 기다리거나 task가 완료되기 전에 취소할 수 있다.

<br/>

Swift 5.5에 도입된 최신 동시성 기능을 사용하여 위의 코드를 다시 쓰면 다음과 같다. 

```swift
class API {
  ...
  func fetchServerStatus() async throws -> ServerStatus {
    let (data, _) = try await URLSession.shared.data(
      from: URL(string: "http://amazingserver.com/status")!
    )
    return ServerStatus(data: data)
  }
}

class ViewController {
  let api = API()
  let viewModel = ViewModel()

  func viewDidAppear() {
    Task {
      viewModel.serverStatus = try await api.fetchServerStatus()
    }
  }
}
```

<br/>

이전 코드와 같은 라인 수를 가지고 있지만,  그 의도는 컴파일러와 런타임 모두에 더 명확하다.

구체적으로..

- `fetchServerStatus()` 는 실행을 일시 중단 및 재개할 수 있는 비동기 함수(asynchronous function) 이다.
  async 키워드를 사용하여 표시한다.
- `fetchServerStatus()`  는 `Data`를 리턴하거나 error을 throw 한다. 이는 컴파일 타임에 체크된다.(error일 경우를 처리하는 것을 까먹을 염려가 없다!)
- Task 는  asynchronous context 에서 주어진 closure를 실행한다. 그래서 컴파일러는 해당 closure 안에서 어떤 코드가 안전한지 또는 비안전한지 알 수 있다. 
- 마지막으로 await 키워드를 통해 비동기 함수를 호출할 때마다 런타임에 코드를 일시 중단하거나 취소할 수 있는 기회를 준다.  이를 통해 시스템은 현재 작업 대기열(current task queue)에 있는 우선순위를 지속적으로 변경할 수 있다. 

## Separting code into partial tasks 

"각 await에 코드가 일시중지 될 수도 있다" 라는 말의 정확한 의미는 무엇입니까?

CPU 코어 및 메모리와 같은 공유 리소스를 최적화하기 위해 Swift는 코드를 논리적 단위 (**partial tasks** 또는 **partials** 라고 불리는) 로 분할합니다.

<img width="375" alt="스크린샷 2022-06-05 오후 2 56 42" src="https://user-images.githubusercontent.com/9502063/172037444-ddbed991-6754-4de8-ae19-69d3bf8d935d.png">

Swift 런타임은 비동기 실행을 위해 이러한 각 부분을 별도로 스케쥴링 합니다.

각 부분 작업(partial task)이 완료되면 시스템은 시스템의 부하와 보류 중인 작업의 우선순위에 따라 '코드를 계속할지 아니면 다른 작업을 실행할지'를 결정합니다. 

그렇기 때문에 await이 붙인 각 부분 작업이 시스템의 재량에 따라 다른 스레드에서 실행될 수 있다는 점을 기억하는 것이 중요합니다. 

스레드가 변경될 수 있을 뿐만 아니라 대기 후 앱 상태에 대한 가정을 해서는 안 됩니다. 비록 두 줄의 코드가 차례로 표시되지만, 시간이 지나면 서로 다르게 실행될 수 있습니다. 대기하는 데 임의 시간이 걸리고 그 사이에 앱 상태가 크게 변경될 수 있습니다.

요약하자면, async/await 문법은 매우 간단하며 많은 정보를 담고 있습니다.  

이것은 컴파일러가 안전하고 확실한 코드를 작성할 수 있도록 가이드해주고,  런타임은 공유 시스템 자원의 잘 조정된 사용을 최적화합니다. 



### Executing partial tasks

동시성 모델의 기본은 Executor(실행자)에서 실행하는 partial tasks로 비동기코드를 나누는 것입니다. 



<img width="614" alt="스크린샷 2022-06-05 오후 3 03 44" src="https://user-images.githubusercontent.com/9502063/172037627-67becb01-6f7c-46b4-80bc-ef67feb7937f.png">

Executors는 GCD queues 와 비슷하지만, Executors가 더 파워풀하고 lower-level 입니다. 

Executors는 작업을 신속하게 실행하고 실행 순서, 스레드 관리 등과 같은 복잡성을 완전히 숨길 수 있습니다.



### Controlling a task’s lifetime

현대 동시성의 필수적인 새로운 특징 중 하나는 비동기 코드의 수명을 관리하는 시스템의 능력입니다. 

기존 멀티 스레드 API의 큰 단점은 일단 비동기 코드 조각이 실행되기 시작하면, 코드가 제어를 포기하기로 결정할 때까지 시스템이 CPU 코어를 점잖게 회수할 수 없다는 것입니다. 이것은 한 작업이 더 이상 필요하지 않은 후에도, 그것은 여전히 자원을 소비하고 실질적인 이유 없이 일을 수행한다는 것을 의미합니다

원격 서버에서 콘텐츠를 가져오는 서비스가 좋은 예입니다. 이 서비스를 두 번 호출하면 시스템에 처음 호출한 불필요하게 사용된 리소스를 회수하는 자동 메커니즘이 없으므로 리소스가 불필요하게 낭비됩니다.

새 모델은 코드를 부분 분할하여 런타임으로 체크인할 수 있는 중단 지점을 제공합니다. 이것은 시스템이 코드를 일시 중단할 뿐만 아니라 재량에 따라 완전히 취소할 수 있는 기회를 제공합니다.

새로운 비동기 모델 덕분에, 작업을 취소하면 런타임은 asyn hierarchy로 걸어내려가 모든 child tasks를 취소할 수 있습니다. 

<img width="553" alt="스크린샷 2022-06-05 오후 3 11 08" src="https://user-images.githubusercontent.com/9502063/172037847-8ff59253-f67e-4900-9232-1fb5a22ccaca.png">



하지만 중단지점(suspension point) 없이  길고 지루한 계산을 수행하는 힘든 작업이 있다면 어떨까요? 

이러한 경우 Swift는 현재 작업이 취소되었는지 탐지할 수 있는 API를 제공합니다. 이 경우 수동으로 실행을 중지할 수 있습니다.



마지막으로 중단지점(suspension point) 은  에러에 대한 탈출경로(escape route)를 제공하여 

에러를 catch하고 핸들링하는 지점으로 hierarchy를 타고 올라가게 해줍니다. 

<img width="448" alt="스크린샷 2022-06-05 오후 3 14 53" src="https://user-images.githubusercontent.com/9502063/172037951-2163c0f9-d56d-48d1-8f36-4d12b3f1fc4a.png">

이런  throwing functions은  작업이 오류를 발생시키는 즉시 빠른 메모리 해제를 위해 최적화되어있습니다.

<br/>

최신 Swift 동시성 모델에서 반복되는 주제는 **안전성, 최적화된 리소스 사용 및 최소 구문(minimal syntax)** 입니다. 이 장의 나머지 부분에서는 이러한 새로운 API에 대해 자세히 알아보고 직접 사용해 볼 것입니다.
