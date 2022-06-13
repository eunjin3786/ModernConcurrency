# Chapter 3: AsyncSequence & Intermediate Task

비동기 시퀀스(async sequences) 를 사용하여 작업을 더 쉽게 수행할 수 있습니다. 

비동기 시퀀스는 Swift 시퀀스를 통해 반복(iterating) 하는 것 처럼 비동기 결과(asynchronous results) 사용을 단순화합니다. 



Chapter2의 SuperStorage 앱을 계속 이어서 실습합니다. 

이 챕터가 끝나면 병렬 다운로드(parallel download) 기능을 사용할 수 있습니다.

<img width="368" alt="스크린샷 2022-06-05 오후 5 15 21" src="https://user-images.githubusercontent.com/9502063/172041830-8a17bc4b-fb1f-4f9a-afb1-51366901a085.png">



### Getting to know AsyncSequence

AsyncSequence는 '요소(element)를 비동기적으로 생성할 수 있는 시퀀스' 를 나타내는 프로토콜입니다. 

표면(surface) API는 Swift standard library의 Sequence와 동일합니다.

한가지 차이점이 있는데, next element를 기다려야한다는 점입니다. 

(일반 Sequence처럼 다음 요소를 즉시 사용할 수 없을 수 있으므로)



<br/>

비동기 시퀀스를 사용하는 몇가지 작업은 다음과 같습니다.

- Sequence 를 for loop으로 iterating 한다. `await`을 사용하고 AsyncSequence가 throwing 한다면, `try` 도 사용한다. 코드는 each loop iteration에서 다음 값을 얻기 위해 일시 중단된다. 

  ```swift
  for try await item in asyncSequence {
    // Next item from `asyncSequence`
  }
  ```


- standard library에 있는 iterator의 비동기 대안(asynchronous alternative) 을 사용한다. while loop와 함께.  
  동기식 시퀀스를 사용하는 것과 유사한데,  반복자(iterator)를 만들고 시퀀스가 끝날때까지 await을 사용하여 `next()`를 반복 호출해야한다. 

  ```swift
  var iterator = asyncSequence.makeAsyncIterator()
  while let item = try await iterator.next() {
    ...
  }
  ```

- `dropFirst(_:)`, `prefix(_:)`  , `filter(_:)`  와 같은 표준 시퀀스 메서드(standard sequence methods)를 사용한다. 

  ```swift
  for await item in asyncSequence
    .dropFirst(5)
    .prefix(10)
    .filter { $0 > 10 }
    .map { "Item: \($0)" } {
      ...
    }
  
  ```

- special raw-byte sequence wrapper 를 사용한다. (file contents 또는 서버 URL에서 fetching 해올때)

  ```swift
  let bytes = URL(fileURLWithPath: "myFile.txt").resourceBytes
  
  for await character in bytes.characters {
    ...
  }
  
  for await line in bytes.lines {
    ...
  }
  ```

- your own type에 AsyncSequence를 채택해서 custom sequence 를 만든다.

- `AsyncStream` 를 할용해서 your **very** own custom async sequence 를 만든다. 


<br/>
Apple에서 제공하는 모든 비동기 시퀀스 유형은

[AsyncSequence 문서](https://developer.apple.com/documentation/swift/asyncsequence) 의 Conforming Types를 보면 알 수 있습니다. 

![스크린샷 2022-06-13 오후 10 04 18](https://user-images.githubusercontent.com/9502063/173360036-4c7ad995-f0a2-4315-97c5-550ce4765376.png)

