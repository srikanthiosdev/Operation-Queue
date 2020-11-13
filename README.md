# Operation and OperationQueue
Session notes to demonstrate an intro about Operation and OperationQueue works. And a deep look about custom Operation and dependencies.
## Session Lineup
1. OperationQueue
2. Operation
3. Operation Creation
4. Execution
5. Queue Priority
6. Operation States
7. KVO - complaint
8. Dependencies
9. Concurrent Operation
10. Create own custom concurrent Operation to support dependencies with above learnings

## 1. OperationQueue
It regulate execution of Operation based on priority, readiness and operation dependencies.<br/>
Shouldn't rely on queue semantics.<br/>
Run on an internal dispatch queue rather than manage threads directly.<br/>
Concurrent by default.<br/>
Once we add Operation we can’t remove it.<br/>
Retain Operation until finished. Suspending[isSuspended] it with operations that aren't finished result in memory leak.<br/>
Uses Qos to determine the resource to utilise.<br/>
 
## 2. Operation
An abstract class to represent a code and data associated with single task. They have built in logic to coordinate execution, we can focus on implementation of task.

## 3. Operation Creation
1. Custom subclass<br/>
2. Use system defined: NSInvocationOperation[Available in Objective-C], BlockOperation.
 
```swift
class CustomOperation: Operation {}
let queue = OperationQueue()
let customOperation = CustomOperation()
```
## 4. Execution
Single shot object once executed can’t used to execute again. We can execute Manually by ourselves or using OperationQueue

1. Manual<br\>
We should call start() to execute operation immediately in current thread rather than new thread. If operation is not ready when executing will result in exception.

If operation is not ready when executing will result in exception.
 ```swift
 extension CustomOperation {

    override func main() {
        print("Executed Custom Operation")
    }

    override var isReady: Bool {
        return false// If false when executing, will result in exception
    }
}
customOperation.start()
```
 2. OperationQueue
  ```swift
queue.addOperation(customOperation)
```
## 5. Queue Priority
  ```swift
class QueuePriorityDemo {

    init() {}

    func executePriority() {
        let operationQueue = OperationQueue.main
        let veryHighBlockOperation = BlockOperation { print("QueuePriority.veryHigh \(Operation.QueuePriority.veryHigh.rawValue)")}
        let highBlockOperation = BlockOperation { print("QueuePriority.high \(Operation.QueuePriority.high.rawValue)")}
        let normalBlockOperation = BlockOperation { print("QueuePriority.normal \(Operation.QueuePriority.normal.rawValue)")}
        let lowBlockOperation = BlockOperation { print("QueuePriority.low \(Operation.QueuePriority.low.rawValue)")}
        let veryLowBlockOperation = BlockOperation { print("QueuePriority.veryLow \(Operation.QueuePriority.veryLow.rawValue)")}

        // Setting custom priority will switch the priority towards the normal priority, setting 10 will execute after 8[veryhigh priority]
        let customPriority = Operation.QueuePriority(rawValue: 5) ?? .normal
        let customPriorityBlockOperation = BlockOperation()
        customPriorityBlockOperation.queuePriority = customPriority
        customPriorityBlockOperation.addExecutionBlock { print("Custom QueuePriority \(customPriority.rawValue)")}

        // QueuePriority influence order of Operations Dequeued and Executed. We should use to classify non-dependent Operations. For dependent Operations we can use addDepedency(_:) method
        veryHighBlockOperation.queuePriority = .veryHigh
        highBlockOperation.queuePriority = .high
        normalBlockOperation.queuePriority = .normal
        lowBlockOperation.queuePriority = .low
        veryLowBlockOperation.queuePriority = .veryLow

        // Execute in the proper order if the queue is main queue
        operationQueue.addOperations([veryLowBlockOperation, lowBlockOperation, normalBlockOperation, highBlockOperation, veryHighBlockOperation, customPriorityBlockOperation], waitUntilFinished: false)
    }
}

var priorityDemo: QueuePriorityDemo = QueuePriorityDemo()
priorityDemo.executePriority()
```
## 6. Operation States
Operation objects store the state for proper execution and notify clients progression of lifecycle. Custom subclass maintain the state to ensure proper execution.

**isReady:** To know when operation is ready to execute. It's determined by dependent operation, external conditions or our own implementation.

**isExecuting:** To know whether operation is working on a task. When replacing start method we should replace isExecuting and generate KVO notifications when execution state changes.

**isFinished:** to know operation is finished or cancelled and exiting. Operation clears dependency when this key path becomes true. Also operationqueue won’t dequeue an operation until isFinished becomes true.

**isCancelled:** to know cancellation was requested. Its encouraged to support cancellation but not required. We shouldn't send KVO for this key path. Our custom start() method must be prepared to handle early cancellation.

## 7. KVO - complaint
Operation may execute in any thread, KVO notifications associated with that operation may similarly occur in any thread. Shouldn't bind with UI
Preceding or additional properties/implementation should comply with KVO.
```swift
 let blockOperation = BlockOperation(block: {
    Thread.sleep(forTimeInterval: 2)
    print("Executing Block Operation")
})

class SimpleObserver: NSObject {

    func observe(operation: Operation) {
        print("isReady initial value \(operation.isReady)")
        operation.addObserver(self, forKeyPath: "isExecuting", options: .init(arrayLiteral: [.new, .old]), context: nil)
        operation.addObserver(self, forKeyPath: "isFinished", options: .init(arrayLiteral: [.new, .old]), context: nil)
    }

    override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
        if let key = keyPath, let change = change {
            switch key {
                case "isExecuting":
                    if let value = change[.newKey] as? Bool {
                        print("Update isExecuting to \(value)")
                    }
                    if let value = change[.oldKey] as? Bool {
                        print("Old value of isExecuting is \(value)")
                    }
                case "isFinished":
                    if let value = change[.newKey] as? Bool {
                        print("Update isFinished to \(value)")
                    }
                    if let value = change[.oldKey] as? Bool {
                        print("Old value of isFinished is \(value)")
                }
                default:
                    return
            }
        }
        print("KVO Thread \(Thread.current)")
    }

    func ignore(operation: Operation) {
        operation.removeObserver(self, forKeyPath: "isExecuting")
        operation.removeObserver(self, forKeyPath: "isFinished")
    }

}

let observer = SimpleObserver()
observer.observe(operation: blockOperation)
print("Start Operation")
print("Current Thread \(Thread.current)")
queue.addOperation(blockOperation)
```
**waitUntilAllOperationsAreFinished:** Blocks current thread, then wait for receiver current and queued operation to finish executing and returns. At that time current thread can't add operations and other threads can.
 
 ```swift
queue.waitUntilAllOperationsAreFinished()
print("End Operation")
observer.ignore(operation: blockOperation)
```
## 8. Dependencies
We can execute operation in a specific order by adding or removing dependency.
Operation is considered as ready when all dependent operation objects finished executing[it can be success or failure-cancelling]. It’s upto us to determine the operation with dependencies can proceed or not[which give us flexibility to monitor any kind of errors].
 ```swift
let operation1 = BlockOperation {
    print("Operation 1 is starting")
    Thread.sleep(forTimeInterval: 1)
    print("Operation 1 is finishing")
}

let operation2 = BlockOperation {
    print("Operation 2 is starting")
    Thread.sleep(forTimeInterval: 1)
    print("Operation 2 is finishing")
}

operation2.addDependency(operation1)

// Way 1
print("Start dependency operations")
let queue = OperationQueue()
queue.addOperation(operation1)
queue.addOperation(operation2)
queue.waitUntilAllOperationsAreFinished()
print("Done!")

// Way 2
let queue1 = OperationQueue()
let queue2 = OperationQueue()
print("Start dependency operations")
queue1.addOperation(operation1)
queue2.addOperation(operation2)
queue2.waitUntilAllOperationsAreFinished()
print("Done!")
```
## 9. Concurrent Operation:
Override start(), isAsynchronous, isExecuting, isFinished.
start() method is responsible for starting the operation. Upon starting the operation, it should also update the execution state of the operation.
Shouldn't call super we should have our own implementation

Create own custom concurrent Operation to support dependencies with above learnings
 ```swift
 enum OperationState: String {
    case ready = "Ready"
    case executing = "Executing"
    case finished = "Finished"
    fileprivate var keyPath: String { return "is" + self.rawValue }
}

class ConcurrentOperation: Operation {

    var operationName: String?

    init(operationName: String? = nil) {
        self.operationName = operationName
    }

    private let queue = DispatchQueue(label: "com.demo.operation", attributes: .concurrent)

    // Start method is responsible for starting operation in async manner. Spawn new thread or calling async function should be done from here.
    override func start() {
        // We should never call super, we need set up our own behaviour

        // First we need to check operation is cancelled before starting a task.
        guard !isCancelled else {
            currentState = .finished
            return
        }
        print("Operation \(operationName ?? "Default") is starting")
        // We need to update execution state as isExecuting though KVO.
        currentState = .executing
        Thread.sleep(forTimeInterval: 5)
        // On completion or cancel must generate KVO notification for both isExecuting and isFinished to mark final state.
         // It’s important to update these state, since queued operation will be removed only they finishes.
        print("Operation \(operationName ?? "Default") is finishing")
        self.currentState = .finished
    }

    // Our preceding or custom properties should provide status in thread safe manner.

    override var isAsynchronous: Bool {
        return true
    }

    override var isExecuting: Bool {
        queue.sync {
            return currentState == .executing
        }
    }

    override var isFinished: Bool {
        queue.sync {
            return currentState == .finished
        }
    }

    override var isReady: Bool {
        queue.sync {
            if dependencies.isEmpty {
                return currentState == .ready
            } else {
                return dependencies.allSatisfy({$0.isFinished})
            }
        }
    }

    override func cancel() {
        super.cancel()
        currentState = .finished
    }

    private var stateStore: OperationState = .ready

    var currentState: OperationState {
        get {
            queue.sync {
                return stateStore
            }
        }
        set {
            let oldValue = currentState
            willChangeValue(forKey: currentState.keyPath)
            willChangeValue(forKey: newValue.keyPath)
            queue.sync(flags: .barrier) {
                stateStore = newValue
            }
            didChangeValue(forKey: currentState.keyPath)
            didChangeValue(forKey: oldValue.keyPath)
        }
    }
}

let concurrentOperation1 = ConcurrentOperation(operationName: "1")
let concurrentOperation2 = ConcurrentOperation(operationName: "2")
concurrentOperation2.addDependency(concurrentOperation1)

// Way 1
print("Start dependency operations")
let operationQueue = OperationQueue()
operationQueue.addOperation(concurrentOperation1)
operationQueue.addOperation(concurrentOperation2)
operationQueue.waitUntilAllOperationsAreFinished()
print("Done!")

// Way 2
let queue1 = OperationQueue()
let queue2 = OperationQueue()
print("Start dependency operations")
queue1.addOperation(concurrentOperation1)
queue2.addOperation(concurrentOperation2)
queue2.waitUntilAllOperationsAreFinished()
print("Done!")

// Way 3
print("Start dependency operations")
concurrentOperation1.start()
concurrentOperation2.start()
concurrentOperation2.waitUntilFinished()
print("Done!")

// Way 4
print("Start dependency operations")
let operationQueue = OperationQueue()
operationQueue.maxConcurrentOperationCount = 1
operationQueue.addOperation(concurrentOperation1)
operationQueue.addOperation(concurrentOperation2)
operationQueue.waitUntilAllOperationsAreFinished()
print("Done!")
```
## 10. Understand Exercise
 ```swift
let concurrentOperation1 = ConcurrentOperation(operationName: "1")
let concurrentOperation2 = ConcurrentOperation()

let attachingOperation = BlockOperation {
    print("Injecting data from concurrentOperation1 to concurrentOperation2")
    concurrentOperation2.operationName = "2"
}
concurrentOperation2.addDependency(attachingOperation)
attachingOperation.addDependency(concurrentOperation1)
let exerciseOperationQueue = OperationQueue()
print("Start dependency operations")
exerciseOperationQueue.addOperations([concurrentOperation1, concurrentOperation2, attachingOperation], waitUntilFinished: true)
print("Done!")
```
