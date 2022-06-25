# Chapter 3: AsyncSequence & Intermediate Task

## Getting started with AsyncSequence



<img width="377" alt="스크린샷 2022-06-25 오후 8 23 05" src="https://user-images.githubusercontent.com/9502063/175771360-e4beabf3-46cc-44e7-a21a-2d54ef6d51b1.png">



이전 챕터에서는 실버 다운로드 옵션 (전체 파일을 한번에 가져오고 화면 미리보기를 제공하는 옵션) 을 코딩했습니다.



이번 챕터에서는 골드 다운로드 옵션 (파일이 다운로드될 때 점진적인 UI 업데이트를 제공하는 옵션) 을 구현하겠습니다.



서버에서 파일을 바이트의 비동기 시퀀스로 읽음으로써 (by reading the file as an asynchronous sequence of bytes)

파일 내용을 수신할때 프로그레스바를 업데이트 시킬 수 있습니다.



### Adding your asynchronous sequence

`SuperStorageModel.swift`의 `downloadWithProgress(fileName:name:size:offset:)` 로 갑니다. 



이미 코드가 작성되어있고 화면에 다운로드 상황을 보여주는 `addDownload(name:)` 도 콜하고 있습니다. 

```swift
class SuperStorageModel: ObservableObject {
  func downloadWithProgress(file: DownloadFile) async throws -> Data {
    return try await downloadWithProgress(fileName: file.name, name: file.name, size: file.size)
  }

  /// Downloads a file, returns its data, and updates the download progress in ``downloads``.
  private func downloadWithProgress(fileName: String, name: String, size: Int, offset: Int? = nil) async throws -> Data {
    guard let url = URL(string: "http://localhost:8080/files/download?\(fileName)") else {
      throw "Could not create the URL."
    }
    await addDownload(name: name)
    return Data()
  }
}

extension SuperStorageModel {
  /// Adds a new download.
  @MainActor func addDownload(name: String) {
    let downloadInfo = DownloadInfo(id: UUID(), name: name, progress: 0.0)
    downloads.append(downloadInfo)
  }
}
```

<br/>

`downloadWithProgress(fileName:name:size:offset:)` 의 return 문 앞에 아래 코드를 추가합니다. 

```swift
let result: (downloadStream: URLSession.AsyncBytes, response: URLResponse)
```

<br/>

기존에는 `URLSession.data(for:delegate:) ` 를 사용해서 `Data`를 return 했지만, 이번에는 대체 API인 `URLSession.AsyncBytes` 를 사용하겠습니다.

이 시퀀스는 URL 요청에서 받은 바이트를 비동기식으로 제공합니다. 

<br/>

HTTP 프로토콜을 통해 서버는 부분 요청 (partial request) 를 지원하도록 정의할 수 있습니다.

서버가 이를 지원하는 경우, 한번에 전체 응답을 주는 대신, 응답의 바이트 범위를 반환하도록 요청할 수 있습니다. 

좀 더 흥미롭게 하기 위해서, 앱에서 표준 요청 (standard request)와 부분 요청 (partial request) 를 모두 지원할 것입니다. 



<img width="600" alt="스크린샷 2022-06-25 오후 8 34 20" src="https://user-images.githubusercontent.com/9502063/175771758-bea9c83e-fe6c-4c9e-bb46-c23aa8dbc652.png">



부분 응답(partial response) 을 사용하면 파일을 여러 부분으로 분할하여 병렬로 다운로드할 수 있습니다. 

이 장 뒷부분에 있는 Cloud 9 라는 다운로드 옵션을 구현할 때 이 기능이 필요합니다. 

<br/>

부분 파일 요청을 만들기 위해 아래 코드를 추가하여 계속합니다. 

```swift
if let offset = offset {
      let urlRequest = URLRequest(url: url, offset: offset, length: size)
      result = try await
        URLSession.shared.bytes(for: urlRequest, delegate: nil)

      guard (result.response as? HTTPURLResponse)?.statusCode == 206 else {
        throw "The server responded with an error."
      }
    } else {
      result = try await URLSession.shared.bytes(from: url, delegate: nil)

      guard (result.response as? HTTPURLResponse)?.statusCode == 200 else {
        throw "The server responded with an error."
      }
    }
```

코드가 offset을 지정하는 경우, URL 요청을 작성하여 ` URLSession.bytes(for:delegate:)`  에 전달합니다. 

그러면 이런 `(downloadStream: URLSession.AsyncBytes, response: URLResponse)`  튜플을 반환해주는데, 

파일의 바이트를 열거하는 비동기 시퀀스, 응답 세부정보 로 구성되어있습니다. 

<br/>

response code가 206인 지 확인하여 부분 반응 (partial response)이 성공했음을 나타냅니다.

<br/>

그리고 else문을 추가해서 non-partial request 도 처리해줍니다. 

이 코드는 200 status 랑 성공적인 서버 응답을 확인하는 것을 제외하고 이전에 수행한 것과 비슷합니다. 

<br/>

부분 요청(partial request)든 표준 요청 (standard request) 든 상관없이 `result.downloadStream` 에서 비동기 바이트 시퀀스(asynchronous byte sequence)를 사용할 수 있게 됩니다.  



### Using ByteAccumulator

이제 응답 바이트 (response bytes) 에 대해 반복 (iterating)을 시작할 시간입니다. 이 장에서는 byte를 iterating하는 custom logic을 구현합니다.  `ByteAccumulator` 라는 타입을 사용해서 시퀀스에서 바이트 배치(batches of byte) 를 가져옵니다. 

<br/>

파일을  batches 로 처리해야하는 이유가 무엇입니까? 파일은 수백만 혹은 수십억 바이트를 포함할 수 있습니다. each byte 마다 UI를 업데이트 하고 싶지는 않을 것 입니다.

<br/>

`ByteAccumulator`는 각 바이트 배치를 가져온 후에 파일의 모든 내용을 수집하고 UI를 주기적으로 업데이트하는 데 도움이 됩니다. 



<img width="634" alt="스크린샷 2022-06-25 오후 8 56 20" src="https://user-images.githubusercontent.com/9502063/175772501-27a5e5f3-f531-4b64-90bb-e795a460d34a.png">



`ByteAccumulator` 는 아래와 같이 구현되어있습니다. 

```swift
class ByteAccumulator: CustomStringConvertible {
  private var offset = 0
  private var counter = -1
  private let name: String
  private let size: Int
  private let chunkCount: Int
  private var bytes: [UInt8]
  var data: Data { return Data(bytes[0..<offset]) }

  /// Creates a named byte accumulator.
  init(name: String, size: Int) {
    self.name = name
    self.size = size
    chunkCount = max(Int(Double(size) / 20), 1)
    bytes = [UInt8](repeating: 0, count: size)
  }

  /// Appends a byte to the accumulator.
  func append(_ byte: UInt8) {
    bytes[offset] = byte
    counter += 1
    offset += 1
  }

  /// `true` if the current batch is filled with bytes.
  var isBatchCompleted: Bool {
    return counter >= chunkCount
  }

  func checkCompleted() -> Bool {
    defer { counter = 0 }
    return counter == 0
  }

  /// The overall progress.
  var progress: Double {
    Double(offset) / Double(size)
  }

  var description: String {
    "[\(name)] \(sizeFormatter.string(fromByteCount: Int64(offset)))"
  }
}

```



`ByteAccumulator`  를 시작하기 위해, `downloadWithProgress(fileName:name:size:offset:) `의 return 문 앞에 아래 코드를 추가합니다. 

```swift
var asyncDownloadIterator = result.downloadStream.makeAsyncIterator()
```

<br/>

AsyncSequence는 `makeAsyncIterator()` 라는 메서드를 사용해서 시퀀스에 대한 비동기 반복자(asynchronous iterator) 를 반환합니다. 반환된 `asyncDownloadIterator` 를 사용하여 바이트에 대해 한번에 하나씩 반복합니다.

<br/>

이제 accumulator를 추가하고 모든 바이트를 수집하면서 accumulator를 만들고 사용하는 코드를 추가하세요

```swift
let accumulator = ByteAccumulator(name: name, size: size)

while !stopDownloads, !accumulator.checkCompleted() {
  let byte = try await asyncDownloadIterator.next() 
  accumulator.append(byte)
}
```



### Updating the progress bar

batch가 완료되고, while문 후에 download progress bar를 업데이트해주는 코드를 작성합니다. 

```swift
while !stopDownloads, !accumulator.checkCompleted() {
  while !accumulator.isBatchCompleted,
    let byte = try await asyncDownloadIterator.next() {
    accumulator.append(byte)
  }

  let progress = accumulator.progress
  Task.detached(priority: .medium) {
    await self
      .updateDownload(name: name, progress: progress)
  }
  print(accumulator.description)
}
```



##### 1.

`Task.detached(...)` 를 사용했는데 

`Task(priority:operation:)`  로 task를 만드는 rogue(거친..?) version 입니다. 

detached task는 상위 작업을 상속하지 않습니다.

(일반적으로 `Task.detached(...)` 는 동시성 모델의 효율성에 부정적인 영향을 미치기 때문에 사용하지 않는 것이 좋습니다. 그러나 이 경우, 구문을 배우기 위해 그것을 사용하는 것에는 아무런 문제가 없습니다.)

<br/>

medium priority 로 설정해서 작업을 생성하면 진행 중인 다운로드 작업의 속도가 느려지지 않습니다. 이 작업은  `userInitiated` priority와 함께 실행됩니다. 사용자 action에 의해 시작된 작업이기 때문입니다. 



##### 2.

또다른 흥미로운 측면은 클로저에서 self를 캡쳐한다는 것입니다. 

`SuperStorageModel`은 클래스이며 reference type 입니다. 즉 일반적인 메모리 관리 규칙이 escaping closure를 사용할 때 적용됩니다. 여기서는 그렇지 않지만, 만약 reference cycle을 만들 기회가 있다면, weak self 를 사용해서 약하게 캡쳐하십시오.

 

##### 3.

마지막으로 현재 progress 를 계산하여 `updateDownload` 로 넘기고 현재 누적 상태를 콘솔로 출력하여 개발 중에 다운로드를 추적할 수 있습니다. 



### Returning the accumulated result

`downloadWithProgress(fileName:name:size:offset:)`의  `return Data()` 를 

아래 코드로 바꿉니다. 

```swift
return accumulator.data
```

<br/>

`downloadWithProgress(fileName:name:size:offset:)` 가 완성되었습니다. 

```swift
  private func downloadWithProgress(fileName: String, name: String, size: Int, offset: Int? = nil) async throws -> Data {
    guard let url = URL(string: "http://localhost:8080/files/download?\(fileName)") else {
      throw "Could not create the URL."
    }
    await addDownload(name: name)

    let result: (downloadStream: URLSession.AsyncBytes, response: URLResponse)

    if let offset = offset {
      let urlRequest = URLRequest(url: url, offset: offset, length: size)
      result = try await
        URLSession.shared.bytes(for: urlRequest, delegate: nil)

      guard (result.response as? HTTPURLResponse)?.statusCode == 206 else {
        throw "The server responded with an error."
      }
    } else {
      result = try await URLSession.shared.bytes(from: url, delegate: nil)

      guard (result.response as? HTTPURLResponse)?.statusCode == 200 else {
        throw "The server responded with an error."
      }
    }

    var asyncDownloadIterator = result.downloadStream.makeAsyncIterator()

    let accumulator = ByteAccumulator(name: name, size: size)

    while !stopDownloads, !accumulator.checkCompleted() {
      while !accumulator.isBatchCompleted,
        let byte = try await asyncDownloadIterator.next() {
        accumulator.append(byte)
      }
      let progress = accumulator.progress
      Task.detached(priority: .medium) {
        await self
          .updateDownload(name: name, progress: progress)
      }
      print(accumulator.description)
    }
    
    return accumulator.data
  }
```

<br/>

완성된 메소드는 다운로드 시퀀스를 반복하고 모든 바이트를 수집합니다. 그런 다음, 각 배치의 끝에 파일 진행률을 업데이트 합니다. 

<br/>

이제 `DownloadView.swift ` 로 가서 
`downloadWithUpdatesAction`에 아래와 같은 코드를 작성해줍니다. 

```swift
downloadWithUpdatesAction: {
    // Download a file with UI progress updates.
    isDownloadActive = true
    Task {
      do {
        fileData = try await model.downloadWithProgress(file: file)
      } catch { }
      isDownloadActive = false
    }
}
```

` swift run` 으로 서버를 실행시켜주고 (참고: Chapter 1)

파일 선택 > 골드옵션 선택 > progress bar 가 파일 다운로드 중에 반복적으로 업데이트 되는 지 테스트해봅니다.

<img width="371" alt="스크린샷 2022-06-25 오후 9 27 16" src="https://user-images.githubusercontent.com/9502063/175773452-a25ab14c-73ef-4352-bbd9-4904ebbc6514.png">

더욱 안심되는 것은 while 루프가 각 배치 후에 출력하는 콘솔 출력입니다. 

```swift
[graphics-project-ver-1.jpeg] 0.9 MB
[graphics-project-ver-1.jpeg] 1 MB
[graphics-project-ver-1.jpeg] 1.2 MB
[graphics-project-ver-1.jpeg] 1.3 MB
[graphics-project-ver-1.jpeg] 1.4 MB
[graphics-project-ver-1.jpeg] 1.6 MB
...
```



이 출력은 다운로드의 어느 지점에서 progress bar를 업데이트 하는 지 알려주며 각 batch의 크기를 쉽게 계산할 수 있습니다. 

