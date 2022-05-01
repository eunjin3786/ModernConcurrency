# Chapter 1: Why Modern Swift Concurrency?



## 실습 소개

앞으로 'Little John' 이라는 주식 거래 앱을 만들 것이다. (SwiftUI로)

![스크린샷 2022-04-28 오후 11 03 23](https://user-images.githubusercontent.com/9502063/165770459-9d0f7436-6917-4228-bc57-810d745e3739.png)

이 책에 있는 대부분의 프로젝트들은 JSON 데이터를 가져오고 이미지를 다운로드하기 위해 웹 API에 액세스해야한다. 

이 책에는  book server 라고 불리는 자체 서버 앱이 포함되어 있는데, 이 앱은 실습하는 동안 항상 실행해야한다.



터미널을 열고 [00-book-server folder](https://github.com/raywenderlich/mcon-materials/tree/editions/1.0/00-book-server) 로 이동해서 앱을 시작해라

```swift
swift run
```



서버를 처음 실행하면 몇 가지 종속성을 다운로드하여 빌드한다.

이 작업은 1~2분 정도 걸릴 수 있다. 다음 메시지를 보면 서버가 작동 중임을 알 수 있다.

```swift
[ NOTICE ] Server starting on http://127.0.0.1:8080
```



http://localhost:8080/hello 로 접속해보면 현재 시간이 나올 것이다. 

![스크린샷 2022-04-28 오후 11 02 44](https://user-images.githubusercontent.com/9502063/165770473-7777f7e2-1d89-4a27-84df-eb0dde471540.png)

서버를 종료하고 싶으면 터미널에서 Control-C 를 눌러라. 



## 첫 실습~!

📘 예제프로젝트: [01-hello-modern-concurrency](https://github.com/raywenderlich/mcon-materials/tree/editions/1.0/01-hello-modern-concurrency)



메인 앱 화면에 비동기 코드를 추가해보자.



#### 1.

`LittleJohnModel.swift` 를 열고 아래 메소드를 추가해라

```swift
func availableSymbols() async throws -> [String] {
  guard let url = URL(string: "http://localhost:8080/littlejohn/symbols") 
  else {
    throw "The URL could not be created."
  }
}
```

method’s definition의 `async` 키워드는 코드가 비동기 컨텍스트(asynchronous context)에서 실행된다는 것을 컴파일러에게 알려준다. 

다른 말로하면 코드가 마음대로 중단됐다가 재개될 수 있다는 이야기다. 

또한 이 메소드가 완료하는데 얼마나 시간이 걸리는 지 상관없이, 궁극적으로 synchronous method 가 하는 방식과 동일하게 값을 리턴한다. 



#### 2.

URLSession을 호출하고 book server로 부터 데이터를 fetch 하는 코드를 추가하자. 

```swift
let (data, response) = try await URLSession.shared.data(from: url)
```

비동기 메서드인 ` URLSession.data(from:delegate:) ` 를 호출하면 

`availableSymbols()` 은 일시중단되고 server 로부터 데이터를 받아왔을 때 다시 재개(resume) 된다. 



![스크린샷 2022-04-28 오후 11 27 50](https://user-images.githubusercontent.com/9502063/165775486-6869b2e4-3427-4f38-8e1e-0afa34f7f17d.png)



`await` 을 쓰는 것은 런타임에 중단지점(suspension point) 를 제공하는 것이다. 

(메서드를 일시중지하는 장소. 먼저 실행할 다른 작업이 있는지 고려한 후, 코드를 계속 실행함)

너는 스레드나 주변 폐쇄(?)에 대해 걱정할 필요가 없다. 



#### 3.

서버 응답을 확인하고 가져온 데이터를 반환하자.

```swift
guard (response as? HTTPURLResponse)?.statusCode == 200 else {
  throw "The server responded with an error."
}

return try JSONDecoder().decode([String].self, from: data)
```



#### 최종코드

```swift
  func availableSymbols() async throws -> [String] {
    guard let url = URL(string: "http://localhost:8080/littlejohn/symbols")
    else {
      throw "The URL could not be created."
    }
    
    let (data, response) = try await URLSession.shared.data(from: url)
    
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
      throw "The server responded with an error."
    }

    return try JSONDecoder().decode([String].self, from: data)
  }
```





## Using async/await in SwiftUI

`SymbolListView.swift` 로 가자.

#### 1. 

 아래 코드를 추가해라.

```swift
.onAppear {
  try await model.availableSymbols()
}
```

그러면 컴파일 에러를 보게 될 것이다. 

```swift
❌ Invalid conversion from throwing function of type '() async throws -> Void' to non-throwing function type '() -> Void
```

 `onAppear(...)`에서  코드가 동기적(synchronously)으로 실행되는데, 

이런 non-concurrent context 에서 비동기함수 (asynchronous function) 를 호출하려고 했기 때문에 에러가 발생한 것이다. 



#### 2.

`.task(priority:_:)` 라는 view modifier 를 쓰면 된다. 이것은 asynchronous function 을 바로 호출할 수 있게 해준다. 

`onAppear(...)` 를 지우고 아래 코드로 교체하라.

```swift
.task {
  guard symbols.isEmpty else { return }
}
```



`.task(priority:_:)`  를 사용하면 비동기 함수를 호출할 수 있으며 `onAppear(...)`  과 마찬가지로 

view 가 screen 에 나타날 때 호출된다. 그래서 symbols 를 이미 가지고 있지 않은 지 확인하는 것으로 시작한다. 



#### 3.

이제 비동기함수를 호출하자

```swift
do {
  symbols = try await model.availableSymbols()
} catch {
  lastErrorMessage = error.localizedDescription
}
```

`try await`을 사용해서 메서드가 에러를 throw 하거나 값을 비동가적으로 리턴할 수 있음을 나타낸다. 



#### 최종코드

```swift
.task {
  guard symbols.isEmpty else { return }
  do {
    symbols = try await model.availableSymbols()
  } catch {
    lastErrorMessage = error.localizedDescription
  }
}
```



서버가 running 한지 확인하고 앱을 빌드해보자!

![스크린샷 2022-04-28 오후 11 47 32](https://user-images.githubusercontent.com/9502063/165779742-b650bd53-31a0-464e-9ea1-5cb867938a3a.png)

Control-C 를 눌러서 서버를 종료시키고 빌드해보자!

![스크린샷 2022-04-28 오후 11 48 26](https://user-images.githubusercontent.com/9502063/165779947-2f2f6a8f-8b62-4554-bc8a-0d0dbfa1d4ed.png)



에러처리까지 잘동작하는 것을 확인했다!





## Using asynchronous sequences

asynchronous sequence 는 swift standard library에 있는 sequence 와 비슷하다.

비동기 시퀀스의 hook는 시간이 지남에 따라 점점 더 많은 요소를 사용할 수 있게 되면서 비동기적으로 요소를 반복할 수 있다는 것이다. 

` TickerView.swift ` 를 열어보자.  (`ForEach` 를 사용하는 SwiftUI 뷰. 시간의 경과에 따른 주가 변동을 보여줌.)

<br />

이전 섹션에서 비동기 네트워크 요청을 "실행"하고 결과를 기다린 다음 반환했다.

TickerView의 경우 요청이 완료되기를 기다렸다가 데이터를 표시할 수 없기 때문에 동일한 접근 방식이 작동하지 않는다. 

데이터는 무기한으로 계속 들어오고 가격 업데이트를 계속 해야한다. 

<br />

여기서 서버는  시간이 지남에 따라 더 많은 텍스트를 추가하면서 하면서 single long-living response 를 보낸다. 

각 text line은 디코딩 가능한  JSON array 이다. 

```swift
‘[{"AAPL": 102.86}, {"BABA": 23.43}]
// .. waits a bit ...
[{"AAPL": 103.45}, {"BABA": 23.08}]
// .. waits some more ...
[{"AAPL": 103.67}, {"BABA": 22.10}]
// .. waits even some more ...
[{"AAPL": 104.01}, {"BABA": 22.17}]
// ... continuous indefinitely ...’
```



live ticket screen에서 각 line에 대해 반복하고 실시간으로 가격을 화면에 업데이트 해보자!



#### 1.

실시간 가격 업데이트를 시작하는 모델의 메소드를 호출하자.

```swift
.task {
  do {
    try await model.startTicker(selectedSymbols)
  } catch {
    lastErrorMessage = error.localizedDescription
  }
}
```

메서드가 결과를 리턴하지 않는 점만 제외하면 `SymbolListView.swift`에 추가한 코드랑 비슷하다.

single return value 가 아니라 continuous updates 를 핸들링할 것이다. 



#### 2.

`LittleJohnModel.swift` 를 열고 `startTicker` 메서드에 가보자. 여기에 live update 코드를 추가할 것이다. 

```swift
class LittleJohnModel: ObservableObject {
  /// Current live updates.
  @Published private(set) var tickerSymbols: [Stock] = []

  /// Start live updates for the provided stock symbols.
  func startTicker(_ selectedSymbols: [String]) async throws {
    tickerSymbols = []
    guard let url = URL(string: "http://localhost:8080/littlejohn/ticker?\(selectedSymbols.joined(separator: ","))") else {
      throw "The URL could not be created."
    }
  }
  
  /// A URL session that lets requests run indefinitely so we can receive live updates from server.
  private lazy var liveURLSession: URLSession = {
    var configuration = URLSessionConfiguration.default
    configuration.timeoutIntervalForRequest = .infinity
    return URLSession(configuration: configuration)
  }()
  
  ...
}
```

published property 인 `tickerSymbols`를 이미 UI에 연결시켰으므로

이 프로퍼티를 업데이트 하면 뷰에  변경 내용이 전파될 것이다. 



 `startTicker` 메서드에 아래 코드를 추가하자. 

```swift
let (stream, response) = try await liveURLSession.bytes(from: url)
```



`URLSession.bytes(from:delegate:) ` 는 이전에 사용한 ` URLSession.data(from:delegate:) `  와 유사하다.

하지만  데이터 대신 시간이 지남에 따라 반복할 수 있는 비동기 시퀀스를 반환한다.  



또한 shared URLSession 을 사용하는 대신 liveURLSession 이라는 custom pre-configured session 을 사용해서 

requests 가 만료되거나 시간 초과되지 않도록 한다. 이렇게 하면 매우 긴 서버 응답을 무한정 계속 수신할 수 있다. 



#### 3.

아래 코드를 이어서 추가하자.

```swift
guard (response as? HTTPURLResponse)?.statusCode == 200 else {
  throw "The server responded with an error."
}
```



#### 4.

이제 재밌는 부분이다!~!  loop 문을 추가하자

```swift
for try await line in stream.lines {

}
```



stream 은 서버가 응답으로 보내는  sequence of bytes 이다. 

line 해당 응답의 텍스트 행을 하나씩 제공하는 시퀀스의 추상화이다.



#### 5. 

한줄씩  JSON으로 디코딩할 것이므로 for문 안에 아래 코드를 추가한다.

```swift
let sortedSymbols = try JSONDecoder()
  .decode([Stock].self, from: Data(line.utf8))
  .sorted(by: { $0.name < $1.name })

tickerSymbols = sortedSymbols
```



decoder가 성공적으로 각 라인을 symbol 리스트로 디코딩하면,  그것을 sort 해서 `tickerSymbols` 에 할당한다. 

그리고 `tickerSymbols`  은 화면에 렌더링 될 것! 

만약 디코딩이 실패하면 JSONDecoder는 단순히 error을 throw 한다. 



#### 최종코드

```swift
@Published private(set) var tickerSymbols: [Stock] = []

  /// Start live updates for the provided stock symbols.
  func startTicker(_ selectedSymbols: [String]) async throws {
    tickerSymbols = []
    guard let url = URL(string: "http://localhost:8080/littlejohn/ticker?\(selectedSymbols.joined(separator: ","))") else {
      throw "The URL could not be created."
    }
    
    let (stream, response) = try await liveURLSession.bytes(from: url)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
      throw "The server responded with an error."
    }
    
    for try await line in stream.lines {
      let sortedSymbols = try JSONDecoder()
        .decode([Stock].self, from: Data(line.utf8))
        .sorted(by: { $0.name < $1.name })

      tickerSymbols = sortedSymbols
    }
  }
```



서버를 실행시키고 앱을 빌드하자.

몇개의 주식을 체크한 후, Live ticker 버튼을 눌러보자



![스크린샷 2022-04-29 오전 12 16 40](https://user-images.githubusercontent.com/9502063/165786143-2726268d-9751-4e6b-8d4b-dfbe3f6dc395.png)

가격이 실시간으로 업데이트 되지만,

코드에디터에서 보라색 워닝이 뜬다. 

![스크린샷 2022-04-29 오전 12 17 06](https://user-images.githubusercontent.com/9502063/165786218-53141dab-4a0c-4b34-9963-fcdf0df637a9.png)





## Updating your UI from the main thread

예전에는 `@State ` 프로퍼티를 업데이트해서 뷰를 업데이트 시켰다. 

이 때 SwiftUI는 main thread 를 통해 업데이트를 하도록 주의를 기울였다.

<br />

이제 UI update 라고 명시하지 않고, 비동기 작업을 실행하는 동일 컨텍스트에서 `tickerSymbols` 를 업데이트하면 

코드는 임의 스레드 풀(arbitrary thread in the pool)에서 실행된다. 

<br />

그래서 메인 스레드로 전환시키는 코드가 필요하다. 

`tickerSymbols = sortedSymbols` 이 코드를 다음과 같이 교체하자



```swift
await MainActor.run {
  tickerSymbols = sortedSymbols
  print("Updated: \(Date())")
}
```



`MainActor` 는 main thread에서 코드를 실행하는 타입이다.

`MainActor.run(_:)` 를 호출하면 모든 코드를 쉽게 실행할 수 있다. 

<br />

print문을 추가했는데  업데이트가 완료되었는지 확인하는 데 도움이 된다. 

앱을 빌드하면 보라색 워닝이 사라지는 것을 볼 수 있다. 

## Canceling tasks in structured concurrency

앞서 언급했듯이 스위프트와 동시 프로그래밍의 큰 도약 중 하나는 현대의 동시 코드가 **구조화된 방식(structured way)**으로 실행된다는 것이다. 

태스크는 엄격한 계층 구조(hierarchy)로 실행되므로 런타임은 task의 parent가 누구인지 알 수 있고 새로운 task가 상속해야하는 기능이 무엇인지 알 수 있다. 

예를들어, TickerView의 `task(_:)`modifier 를 살펴보자.

1. `startTicker(_:)`  를 asynchronously 하게 콜한다. 
2. `startTicker(_:)` 는 asynchronously 하게  `URLSession.bytes(from:delegate:)`를 기다린다.  
3.  `URLSession.bytes(from:delegate:)` 는 `async sequence` 를 반환한다.  
4. 비동기 시퀀스 ( `async sequence`  ) 를 반복시킨다. 
   
   

<img width="563" alt="스크린샷 2022-05-01 오후 2 59 58" src="https://user-images.githubusercontent.com/9502063/166134227-3e780dd1-7452-437d-8f7d-765e79c5673d.png">



각각의 `suspension point ` (너가 `await` 키워드를 보는 곳) 에서 스레드가 변경될 수 있다. 

전체 프로세스를 `task(_:)` 안에서 시작시키므로 그 `async task` 는 모든 다른 task들의 parent 이다. (다른 task들의 실행 스레드나 일시중단 상태에 상관없이)

`task(_:) ` modifier는 뷰가 사라지면 asynchronous code 를 취소시킨다.

이 책의 뒷부분에서 자세히 알게 될 것이지만, 구조화된 동시성(structured concurrency) 덕분에 

사용자가 화면 밖으로 이동할 때 모든 비동기 작업도 취소된다.



<img width="537" alt="스크린샷 2022-05-01 오후 3 08 37" src="https://user-images.githubusercontent.com/9502063/166134448-34d62d10-f937-444b-9e51-670b6883bde7.png">



실제로 이 작업이 어떻게 작동하는지 확인하려면 업데이트 화면으로 이동하여 Xcode 콘솔을 확인해보자.  LittleJohnModel.startTicker(_:)의  debug print 를 볼 수 있다. 

<img width="590" alt="스크린샷 2022-05-01 오후 3 10 30" src="https://user-images.githubusercontent.com/9502063/166134495-971b21ff-fb39-437a-9dda-ba5ebc228c3b.png">

그리고 뒤로가기를 해보자. TickerView 가 사라지고 `task(_:) ` modifier의 작업이 취소된다.

이는 모든 child tasks를 취소시고 콘솔의 debug logs 도 멈춘 것을 볼 수 있다.

<br/>

그러나 콘솔에 추가 메세지가 출력되는 것을 확인할 수 있다. 

<img width="584" alt="스크린샷 2022-05-01 오후 3 14 11" src="https://user-images.githubusercontent.com/9502063/166134590-3027e01b-4148-4402-b48f-bb832e9fe27d.png">

SwiftUI는 TickerView가 dismiss 된 후에 alert을 띄우려는 코드 문제를 로깅하고 있다.

이 문제는 런타임에서 `startTicker`에 대한 호출을 취소할 때, 일부 내부 tasks가 cancel error를 throw하기 때문에 발생한다.



## Handling cancellation errors

일시 중단된 작업 중 하나가 취소되어도 상관 없는 경우가 있고

task가 취소되었을때 특별한 작업을 수행하는 경우도 있다. (위의 alert 띄우기 처럼) 

<br/>

TickerView의 `task(_:) ` modifier 로 가보자.

```swift
.task {
  do {
    try await model.startTicker(selectedSymbols)
  } catch {
    lastErrorMessage = error.localizedDescription
  }
}
```

<br/>

여기서 모든 error을 catch에서 메세지를 보여주기 위해 저장하고 있다. 

그러나 콘솔에서의 런타임 warning을 피하기 위해서는 취소를 다른 에러와 다르게 처리해야한다. 

`Task.sleep(nanoseconds:) ` 같은 최신 비동기 API는 `CancelationError`를 발생시킨다. 

사용자 지정 오류를 발생시키는 다른 API는 전용 취소 에러 코드가 있다. (ex. `URLSession`)

```swift
} catch {
  if let error = error as? URLError, 
    error.code == .cancelled {
    return
  }
  
  lastErrorMessage = error.localizedDescription
}
```

<br/>

새로운 catch block 은 던져진 에러가 cancelled error code가 있는 `URLError` 인지 확인한다.

이 경우 화면에 메세지를 표시하지 않는다. 

<br/>

`URLError`  는 실시간 업데이트를 가져오는 진행 중인 `URLSession` 에서 발생한다. 

만약 다른 modern API를 사용하다면, 그대신 `CancellationError`가 던져질 것이다. 

<br/>

앱을 한 번 더 실행하고 런타임 경고가 더 이상 나타나지 않는지 확인한다. 



## Challenges

#### Challenge: Adding extra error handling

앱이 여전히 적절하게 처리하지 못하는 edge case가 하나 있다. 

사용자가 가격 업데이트를 관찰하는 동안 서버를 사용할 수 없게 되면 어떻게 됩니까?

가격 화면으로 이동한 다음 터미널 창에서 Control-C를 눌러 서버를 중지하면 이 상황을 재현할 수 있다.

<br/>

에러가 없기 때문에 에러메시지가 팝업으로 뜨지 않는다. 

실제로 서버가 응답 시퀀스를 닫으면 응답 시퀀스가 완료된다. 

이 경우 코드는 에러 없이 계속 실행되지만 더 이상의 업데이트는 생성되지 않는다.

<br/>

이 challenge에서 비동기 시퀀스가 종료될 때 `LittleJohnModel.tickerSymbols`를 reset하는 코드를 추가한 다음 

업데이트 화면 밖으로 이동한다. 

`LittleJohnModel.startTicker(_:)` 안에서 for 루프 뒤에 비동기 시퀀스가 예기치 않게 종료될 경우 `tickerSymbols`를 빈 배열로 설정하는 코드들 추가한다.

`MainActor` 를 사용해서  이 업데이트를 하는 것을 잊지 마라. 



```swift
func startTicker(_ selectedSymbols: [String]) async throws {
    ... 
    for try await line in stream.lines {
      let sortedSymbols = try JSONDecoder()
        .decode([Stock].self, from: Data(line.utf8))
        .sorted(by: { $0.name < $1.name })

      await MainActor.run {
        tickerSymbols = sortedSymbols
        print("Updated: \(Date())")
      }
    }

    // Challenge code.
    await MainActor.run {
      tickerSymbols = []
    }
 }
```

(실행해보면 for문이 계속 반복되다가 서버가 정지되면 for문을 나와 마지막 코드가 실행됨을 볼 수 있음)



<br/>

그리고 TickerView로 가서 새로운 modifier를 추가해라. 

관찰 중인 ticker symbol 수를 옵져빙하고 selection이 reset 되는 경우, 뷰를 dismiss 한다.

```swift
.onChange(of: model.tickerSymbols.count) { newValue in
  if newValue == 0 {
    presentationMode.wrappedValue.dismiss()
  }
}
```



그러면 앱에서 실시간 업데이트를 보다가 서버를 정지하면 

자동으로 업데이트 화면을 해제하고 symbols 목록으로 돌아갈 것이다. 

<br/>

## Key points

- Swift 5.5는 새로운 동시성 모델을 도입했다. 이는 기존 동시성 모델이 가지고 있는 많은 문제 해결한다.
  (참고: [기존 동시성 모델의 문제들](https://github.com/eunjin3786/ModernConcurrency/blob/master/Chapter1-1.md#reviewing-the-existing-concurrency-options))
- `async`키워드는 함수를 비동기 함수로 정의한다. `await` 을 사용하면 비동기 함수의 결과를 비동기 방식으로 기다릴 수 있다.
- 비동기 코드를 실행할 때 `onAppear(_:) `  대신 ` task(priority:_:) ` 라는 view modifier를 사용한다.  
- `for try await` loop syntax 를 사용하면 비동기시퀀스를 자연스럽게 루프 돌릴 수 있다. 






