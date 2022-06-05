# Chapter 2: Getting Started With async/await
## Getting started

'SuperStorage'는 클라우드에 저장된 파일을 찾아보고 local, on-device preview를 위해 파일을 다운로드할 수 있는 앱입니다. 실버, 골드, 클라우드 9의 3가지 구독 플랜을 제공합니다.

<img width="574" alt="스크린샷 2022-06-05 오후 3 25 29" src="https://user-images.githubusercontent.com/9502063/172038295-8fb44638-133a-4657-a772-c0aff270044c.png">



projects/starter 에 있는 SuperStorage의 starter version을 엽니다. 

서버는 mock data를 리턴합니다. (실제로 작동 중인 클라우드 솔루션이 아님) 

또한 느린 다운로드, 에러 시나리오를 재현할 수 있게 해줍니다. 그러므로다운로드 속도에 개의치 마십시오. 당신의 기계에는 아무 이상이 없습니다.



### A bird’s eye view of async/await

async/await은 너의 의도에 따라 다양한 입맛으로 작성할 수 있습니다. 

- function을 aynchronous 하게 선언하려면, throws 나 return type 앞에 async 키워드를 추가합니다.
  function을 await 과 함께 콜하고 function이 throwing이라면, try도 같이 붙여줍니다. 

<img width="673" alt="스크린샷 2022-06-05 오후 3 35 32" src="https://user-images.githubusercontent.com/9502063/172038599-ff754042-7b30-47ca-82db-e6a002fb8219.png">

- computed property를 asynchronous 하게 선언하려면, getter에 async 키워드를 추가합니다.
  그리고 value에 접근할 때 await을 붙여줍니다. 

  <img width="662" alt="스크린샷 2022-06-05 오후 3 38 25" src="https://user-images.githubusercontent.com/9502063/172038700-c0c457fa-d1f4-4d4a-89cd-4c9b51156f30.png">

- closure의 경우, signature에 async를 붙여줍니다. 
  <img width="666" alt="스크린샷 2022-06-05 오후 3 39 02" src="https://user-images.githubusercontent.com/9502063/172038722-bb0c6548-fd54-4f04-aad7-1291d1822be4.png">



### Getting the list of files from the server

app의 모델에 웹서버로부터 파일 리스트를 fetch 해오는 메서드를 추가해봅시다. 

`SuperStorageModel.swift` 를 열고 아래 메소드를 추가하세요

```swift
func availableFiles() async throws -> [DownloadFile] {
  guard let url = URL(string: "http://localhost:8080/files/list") else {
    throw "Could not create the URL."
  }
}
```

throwing, asynchronous function을 선언했기 때문에 다음과 같습니다. 

- 컴파일러는 함수가 작업을 일시 중단하거나 재개할 수 없는 동기 컨텍스트 (synchronous context)에서 이 함수를 호출하지 않도록 합니다.
- 런타임은 새로운 공동 스레드 풀을 사용하여 메서드의 부분 태스크를 스케쥴하고 실행합니다.



```swift
  func availableFiles() async throws -> [DownloadFile] {
    guard let url = URL(string: "http://localhost:8080/files/list") else {
      throw "Could not create the URL."
    }

    let (data, response) = try await URLSession.shared.data(from: url)

    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
      throw "The server responded with an error."
    }

    guard let list = try? JSONDecoder().decode([DownloadFile].self, from: data) else {
        throw "The server response was not recognized."
     }

    return list
  }
```



### Showing the service status

그리고 파일리스트 밑에  server usage 를 표현하기 위해 

<img width="393" alt="스크린샷 2022-06-05 오후 3 55 15" src="https://user-images.githubusercontent.com/9502063/172039271-448fa961-affb-436c-bafe-cfe90486b29f.png">

`SuperStorageModel.swift`  에 Status 라는 메서드를 추가합니다.

```swift
  func status() async throws -> String {
    guard let url = URL(string: "http://localhost:8080/files/status") else {
      throw "Could not create the URL."
    }

    let (data, response) = try await
      URLSession.shared.data(from: url, delegate: nil)

    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
      throw "The server responded with an error."
    }

    return String(decoding: data, as: UTF8.self)
  }
```



### Async let

그리고  `ListView.swift ` 로 가서 `.task` view modifier 안에

아래의 코드를 추가합니다. 

```swift
.task {
  guard files.isEmpty else { return }

  do {
    async let files = try model.availableFiles()
    async let status = try model.status()

    let (filesResult, statusResult) = try await (files, status)

    self.files = filesResult
    self.status = statusResult
  } catch {
    lastErrorMessage = error.localizedDescription
  }
}
```



Swift는 여러 비동기 호출을 그룹화하고 모두 함께 대기할 수 있는 특수 구문을 제공합니다.

두개의 동시 요청을 실행하기 위해 **async let** syntax 를 사용했습니다. 

async let binding은 로컬상수를 만들어줍니다. 



Option-Click으로 Quick Help를 열 수도 있습니다. 

<img width="532" alt="스크린샷 2022-06-05 오후 4 03 24" src="https://user-images.githubusercontent.com/9502063/172039512-5d93f4ba-53af-4f83-b52b-b5d63b1576f9.png">



async let은 선언에 명시적으로 비동기가 표시되어있습니다. 그러므로 이 value를 await 없이 접근할 수 없습니다. 

`files`와 `status` 바인딩은 특정 유형의 값이나 오류를 나중에 사용할 수 있음을 약속합니다.

이 두개를 array에 모을 수도 있고 tuple로 모을 수도 있습니다. 

여기서는 오직 두개이기때문에 tuple syntax 를 선택했습니다. 



<br/>

여기서 `filesResult` 와 `statusResult` 는 무엇일까요? 

````swift
let (filesResult, statusResult) = try await (files, status)
````

각각을 Option Click 해봅시다. 


<img width="683" alt="스크린샷 2022-06-05 오후 4 08 47" src="https://user-images.githubusercontent.com/9502063/172039679-8164f4d7-4ec0-4e68-8d66-58d6288bd0d0.png">

<img width="677" alt="스크린샷 2022-06-05 오후 4 09 03" src="https://user-images.githubusercontent.com/9502063/172039686-689c0e85-defa-4ef9-9dff-1e58f2041ef6.png">



이번에는 오직  `let constant`으로 선언했는데요, `filesResult` 와 `statusResult` 에 접근할 수 있을 때 

두 요청 모두 작업을 완료하고 최종 결과를 제공했음을 나타냅니다.

이 시점의 코드에서는 await이 throw를 하지 않았다면, 모든 동시 바인딩(concurrent bindings) 이 성공적으로 해결되었음을 알 수 있습니다.



### A quick detour through Task

우리가 사용한 `Task ` type이 어떤 건지 알아봅시다. 

Task는 최상위 비동기 작업(**top-level asynchronous task**)을 나타내는 유형입니다.

최상위 수준이라는 것은 동기 컨텍스트에서 시작할 수 있는 비동기 컨텍스트를 만들 수 있음을 의미합니다.

간단히 말해, 동기 컨텍스트(synchronous context)에서 비동기 코드(asynchronous code) 를 실행하려면 

언제나 새 Task가 필요합니다. 



다음 API를 사용하여 task의 실행을 수동으로 제어할 수 있습니다. 

- **Task(priority:operation)**:
  지정된 우선 순위를 사용하여 비동기 실행 작업을 스케줄 합니다. 현재 동기 컨텍스트에서 기본값을 상속합니다.

- **Task.detached(priority:operation)**:
  호출한 context의 기본값을 상속하지 않는다는 점을 제외하면 `Task(priority:operation)` 와 유사합니다. 

- **Task.value**:
  작업이 완료될 때까지 기다린 다음 값을 반환합니다. 다른 언어의 promise 와 유사합니다. 

  

Task를 메인스레드에서 만들면 task는 메인 스레드에서 실행됨을 기억하십시오. 



### MainActor

JPEG 파일을 하나고르고 Silver plan 다운로드를 눌러봅니다.

<img width="295" alt="스크린샷 2022-06-05 오후 4 47 14" src="https://user-images.githubusercontent.com/9502063/172040876-6b44c436-ab16-4783-8847-496ad5f25385.png">

progress bar가 깜박이고 중간 정도만 채워지는 경우가 있습니다. 이는 백그라운드 스레드에서 UI를 업데이트한다는 힌트입니다. 

또한 Xcode 콘솔에 로그 메시지가 표시되고 코드 에디터에 보라색 경고가 표시됩니다.

<img width="692" alt="스크린샷 2022-06-05 오후 4 48 21" src="https://user-images.githubusercontent.com/9502063/172040914-fe4d4efb-12a2-47b7-9a67-64cba48c9596.png">



`SuperStorageModel.swift ` 을 열고 `addDownload(file:):`와 ` updateDownload(name:progress:):` 앞에 `@MainActor` 를 붙여줍니다.

```swift
extension SuperStorageModel {
  /// Adds a new download.
  @MainActor func addDownload(name: String) {
    let downloadInfo = DownloadInfo(id: UUID(), name: name, progress: 0.0)
    downloads.append(downloadInfo)
  }
  
  /// Updates a the progress of a given download.
  @MainActor func updateDownload(name: String, progress: Double) {
    if let index = downloads.firstIndex(where: { $0.name == name }) {
      var info = downloads[index]
      info.progress = progress
      downloads[index] = info
    }
  }
}
```

이 두 메서드에 대한 모든 호출은 자동으로 main actor에서 실행되며 따라서 메인 스레드에서 실행됩니다. 

또한 두 메서드를 비동기적으로 호출해야합니다. 

`download(file:) ` 로 가서 컴파일 에러를 수정합니다. 



```swift
  /// Downloads a file and returns its content.
  func download(file: DownloadFile) async throws -> Data {
    guard let url = URL(string: "http://localhost:8080/files/download?\(file.name)") else {
      throw "Could not create the URL."
    }

    ✅ await addDownload(name: file.name)

    let (data, response) = try await
      URLSession.shared.data(from: url, delegate: nil)

    ✅ await updateDownload(name: file.name, progress: 1.0)

    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
      throw "The server responded with an error."
    }

    return data
  }
```



## Key points

- **async** 표시된 Functions, computed properties, closures 는 비동기 컨텍스트에서 비동기 실행됨. 
  한번 또는 여러번 일시중단했다가 다시 재개될 수 있음
- **await** 은  central async handler가 작업을 일시중단하고 다음에 실행할 건지 결정할 수 있게 함
- **async let** 바인딩은 나중에 값 또는 오류를 제공할 것을 약속함. 이 결과에 await을 사용해서 접근할 수 있음
- **Task()** 는 현재 actor에서 실행하기 위한 비동기 컨텍스트를 만듦. 또한 task의 우선순위를 지정할 수도 있음
- ` DispatchQueue.main` 과 비슷하게 **MainActor** 는 메인스레드에서 코드를 실행하는 유형임

