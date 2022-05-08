# Chapter 2: Getting Started With async/await

### async/await 전에는..

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
