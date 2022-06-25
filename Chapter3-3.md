### Canceling tasks

동시성 모델이 효율적으로 작동하려면, 불필요한 작업을 취소하는 것이 필수적입니다. 

`TaskGroup` 이나 `async let` 같은 새로운 API를 사용할 때, 시스템은 일반적으로 필요한 순간에 자동으로 작업을 취소할 수 있습니다.

그러나 아래 Task API들을 사용하여 task-based 코드에 대한 세분화된 취소 전략을 구현할 수 있습니다. 



- `Task.isCancelled`:  작업이 아직 활성 상태이지만 마지막 일시 중단 시점 이후 취소된 경우, true를 반환합니다. 

- `Task.currentPriority`: 현재 작업의 우선 순위를 반환합니다.
- `Task.cancel()`: 작업 및 하위 작업들을 취소하려고 합니다. 
- `Task.checkCancellation()`: 작업이 취소된 경우 `CancellationError`를 throw하여 throwing context를 종료시키는 것을 쉽게 만들어줍니다.
- `Task.yield()`: 현재 작업의 실행을 일시중단하여 시스템에게 더 높은 우선순위를 가진 다른 작업을 실행할 수 있는 기회를 줍니다.



비동기 작업을 작성할 때 `checkCancellation()` 같은 throwing function이 필요한지 또는 `isCancelled`를 체크하여 직접 제어 흐름을 관리할 지에 따라 사용할 API 를 선택합니다. 



다음 section에서는 더 이상 필요하지 않은 다운로드 작업을 취소하는 custom logic을 구현해보겠습니다. 

### Canceling an async task

시기적절하게 작업을 취소하는 것이 중요한 이유를 입증하려면, 앱에서 다음과 같은 시나리오를 실행하십시오



TIFF 파일 중 하나를 선택하고 Gold를 눌러 다운로드를 시작합니다. 누적기(accumulator)가 파일 내용을 점점 더 많이 수집하면 Xcode의 콘솔에 로그가 나타납니다.

파일을 다운로드하는 동안 Back 버튼을 누르고 콘솔을 관찰합니다. 전체 파일을 다운로드할 때까지 다운로드가 계속됩니다!


### Manually canceling tasks
