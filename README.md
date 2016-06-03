# swift-toolbox
Contains various Swift implementations that can be useful in every project.

This is still a work in progress, and many of the components in this fraemwork could be broken into several submodules.  But, for now, keeping it all in one.

## Features

### Task Framework for Passing Future Values

There is always a need to pass around a value that may not be available immediately. This handy framework allows for an easy to use and elegant approach.

##### Components of the Framework

- A 'TaskKernel', an object which performs the task's work.
- A `TaskScheduler`, an object which is responsible for starting the kernel, and notifying the result. *The default task scheduler uses GCD global dispatch queue.*
- A `TaskFactory`, an object which vends new `Task<>`, constructed with defaults of the properties set on it.
- A method for *cooperative cancellation*, which allows a task's kernel to acknowledge a cancellation request so it can exit gracefully.

##### The Duties of `Task<>`

- Holds a refernce to the `TaskKernel` that will be doing the work.
- Keeps track of status of the task's execution, such as `Waiting`, `Running`, `Completed`, etc.
- Deals with errors thrown by the kernel.
- Maintanis a list of *continuation tasks*, which are tasks that are started when the task completes.

##### Cooperative Cancellation

The use of throwing errors, `TaskCancelled`, is used to convey a cooperative cancellation.  

The class `TaskCancellationSource` provides a property which returns an instance of value-type `TaskCancellationToken`.  The `TaskCancellationToken` is passed to the task when it is created, and subsequently passed to the kernel.  The token is used by the kenel to determine if cancellation has been requested so you can cancel gracefully.

##### Continuation Tasks

A task contains a method `continueWith`, which has an argument list similiar to when you create new tasks.  This method returns another task which is started when the prior tasks completes.

#### Examples

```swift
// Here's a quick example of a "useless" kernel that spins forever on a seperate thread, but supports cancellation.
// This also shows how to use continuation tasks to schedule onto the Main task scheduler.

func doUselessWork() {
  TaskFactory.Default.start { (cancelToken) in
    while true {
      try cancelToken.checkpoint(); // This will throw the appropatie TaskCancelled error if cancellation has requested.
    }
  }
  .continueWith(scheduler: TaskSchedulers.Main) { (prevTask) in
    /* The "antecedent" task is passed to the continuation task's kernel closure, 
      and attemping to access the value will throw, or return the result.  Since antecedent task
      doesn't return anything (aka Void), we don't care about the result. */
      
      guard prevTask.status.isCancelled else {
        // The task cancelled cooperatively.
        someUIElement.text = "The operation was cancelled.";
        return;
      }
      
      do {
        try prevTask.value(); 
      } catch {
        // Something else happened while executing the previous task.  Handle the error.
        someUIElement.text = "Ooops!  Something wrong while trying to do that.";
      }
  }
}
```
