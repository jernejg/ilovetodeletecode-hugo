+++
date = "2014-02-05T11:08:28Z"
title = "Refactoring a spaghetti C# serial communication app & asynchronous control flow modeling with async/await"
slug = "Refactoring-a-spaghetti-C-serial-communication-app-asynchronous-control-flow-modeling-with-asyncawait"
+++
A couple of months back I've got a task of implementing a protocol for an embedded device sending/receiving streams of data through serial communication. Together with this task I also inherited a Windows Forms C# application that already did some of the previous but in a different protocol with a different device. The more I struggled to understand and reuse some of the existing infrastructure, the more I wanted to rewrite the whole thing (sounds familiar?).

Some facts about existing code:

* Each scenario was implemented as a complex state machine with long switch statements.
* For each scenario a heavy weight enum with all possible states was defined.
* State machine transitions were implemented with a table of Action delegates paired with already mentioned enumerations in a key/value dictionary.
* Events were used to signal status of long running operations.

Such code was really hard to read and even harder to maintain. And if you don’t have the documentation for the device's programming interface it's all even harder.

In this post I'm trying to solve two things:

1. How to design a more robust flexible issue/response system used for communicating with the serial device.
2. How to model complex asynchronous control flows, without blocking, using Task Parallel Library and C# 5.0 async/await.

<!--more-->

##  Use the state pattern to get rid of if/switch statements

According to Gang of Four we should use the state pattern in either of the following cases:

* An object's behavior depends on its state, and it must change its behavior at run-time depending on that state.
* Operations have large, multipart conditional statements that depend on the object's state. This state is usually represented by one or more enumerated constants. Often, several operations will contain this same conditional structure. The State pattern puts each branch of the conditional in a separate class. This lets you treat the object's state as an object in its own right that can vary independently from other objects.

Using state pattern we can make a more object oriented state machine so that each behavior specific to one state is encapsulated into a separate object. Such separation of state specific logic simplifies our code as it reduces the need for large conditional statements and makes it flexible so that it is easy to add new states in the future.

#### Players in the state pattern:

| Gang of Four                                                                                                   | Our example                |
|----------------------------------------------------------------------------------------------------------------|----------------------------|
| Context                                                                                                        | EmbeddedDevice             |
| IState (defines an interface for encapsulating the behavior associated with a particular state of the Context) | ICommand                   |
| StateA : IState                                                                                                | OpenFileCommand : ICommand |


{{< highlight csharp "style=paraiso-dark" >}}
public interface ICommand
{
    IEnumerable<byte> CommandInstruction { get; }
    void Proccess(IEnumerable<byte> data);
    bool IsFinished { get; }
}
{{< /highlight >}}

## Collaborations

1. Context delegates state-specific requests to the current ConcreteState object.
EmbeddedDevice type delegates serial data to command that is currently being executed. Delegation is performed via state's method Process (byte [] data) defined in ICommand.
2. Context is the primary interface for clients. Clients can configure a context with State objects. Once a context is configured, its clients don't have to deal with the State objects directly.
Clients issue new commands by adding it to the EmbeddedDevice command queue. Once the command is queued the client needs to wait to be signaled with the result.
3. Either Context or the ConcreteState subclasses can decide which state succeeds another and under what circumstances.
EmbeddedDevice checks the state of the current command with the IsFinished property defined in ICommand. If necessary a transition is made simply by switching the CurrentCommand variable to the next command from command queue and signaling to the client that the command has finished executing. When the next command starts executing the first thing that EmbeddedDevice does is write to serial port bytes from the CommandInstruction property.

Using such partitioning adds another nice side effect. States are modeled to commands this means that logic specific to each command is implemented in its own class. This way only by looking the file structure of the project you can get familiar right away with the internal working of the device you are communicating with. Inspecting each command object instantly gives you information about how the command has to be issued and how the response for each command looks like.

{{< figure src="/img/Post2_structure.png">}}

## Achieve readable asynchronous control flow

The best way to represent future I/O bound work that yet has not finished is through the [TPL (Task Parallel Library)](http://msdn.microsoft.com/en-us/library/dd460717(v=vs.110).aspx). TPL is available since C# 4.0 and is replacing obsolete asynchronous patterns like [APM (Asynchronous Programming Model - BeginMethod, EndMethod, and IAsyncResult)](http://msdn.microsoft.com/en-us/library/ms228963(v=vs.110).aspx) and [EAP (Event Based Asynchronous Pattern - signaling with events)](http://msdn.microsoft.com/en-us/library/wewwczdw(v=vs.110).aspx).

In the heart of TPL are the Task type and its generic subclass Task<TResult>. Task represents an asynchronous operation that does some initialization, maybe spawns a new thread (or maybe not) and then immediately returns without blocking the calling thread. Tasks in our case (serial communication) are all I/O bound, so there is just a lot of waiting (no new threads need to be created). 
The second most important class in TPL is TaskCompletionSource. It enables us to take any operation (that doesn't yet follow TPL) and expose them as a Task. Exposing an asynchronous operation as a Task gives us:

* Monitor/callback mechanism.
* Easily retrieve a return value.
* Failure mechanism - exception propagation.
* Composition - ability to chain Tasks together.
* Separation of concerns - concurrency logic is not mixed with process logic.

Lets see how we wrap TaskCompletionSource around our external asynchronous operation. We add the command to the execution queue together with the corresponding TaskCompletionSource object. TaskCompletionSource exposes a Task object that is completely controlled by its parent. We return this Task to the caller. Providing a TaskCompletionSource for each command through the process of execution gives us a mechanism to signal back to the caller the state of the ongoing command execution (SetCanceled, SetException, SetResult). Returning a Task from the AddToExecutionQueue function enables the caller to monitor that state and utilize a continuation (callback) that executes after completion.

{{< highlight csharp "style=paraiso-dark" >}}
public Task<TReturn> AddToExecutionQueue<TReturn>(TReturn command)
{
    var tcs = new TaskCompletionSource<TReturn>();
    _commandQueue.Enqueue(new CommandTaskPair(command,tsc));
    return tcs.Task;
}
{{< /highlight >}}

## Structuring asynchronous flow of control

Lets look at a more complex asynchronous workflow example. The process of downloading a file from the device is a nice example to start with. It consists of retrieving file information, reading error handling and then cleaning up the resources at the end. These are all long-running processes that together form a more complex asynchronous flow of control which we'll try to model using 3 different approaches:

1. Nesting continuations using lambda expressions.
2. Writing separate functions.
3. Using C# 5 language constructs async/await.

{{< figure src="/img/Post2_workflow.png">}}

## Approach 1: Nesting continuations using lambda expressions

Each next step needs to be invoked from the previously completed one. Such complex sequence of steps result in nested spaghetti continuations which are hard to follow and maintain. Next problem is that we are downloading file chunk by chunk so we need a looping construct, but looping constructs don't go hand in hand with continuations. Because the next iteration should be triggered from the continuation itself we would have to use recursion. The only advantage with nested lambdas are closures which allow you to pass local state to the function without the need for parameters, but beware of closures if you don’t understand them fully, because funny things can happen :).

{{< highlight csharp "style=paraiso-dark" >}}
public Task<long> Download()
{
            long bytesTransfered = 0;
            const int blockSize = 32 * 1024;
 
            var tcs = new TaskCompletionSource<long>();
 
            var openFileCmd = new OpenFileCmd(_fileInfo.Name, FileAccess.Read);
            Task<OpenFileCmd> fileHandleTask = _embeddedDevice.AddToExecutionQueue(openFileCmd);
 
            fileHandleTask.ContinueWith(fileHandleFinished =>
            {
                int fileHandle = fileHandleFinished.Result.SuccessResult;
                var readBlockCmd = new ReadBlockCmd(fileHandle, blockSize);
                Task<ReadBlockCmd> readBlockTask = _embeddedDevice.AddToExecutionQueue(readBlockCmd);
 
                Action<Task<ReadBlockCmd>> recursiveDownload = null;
 
                recursiveDownload = readBlockFInished =>
                {
                    byte[] bytesRead = readBlockFInished.Result.SuccessResult.ActualData.ToArray();
                    bytesTransfered += bytesRead.Count();
                    Proccess(bytesRead);
 
                    if (bytesTransfered < _fileInfo.Length)
                    {
                        readBlockCmd = new ReadBlockCmd(fileHandle, blockSize);
                        Task<ReadBlockCmd> nextBlock = _embeddedDevice.AddToExecutionQueue(readBlockCmd);
                        nextBlock.ContinueWith(recursiveDownload);
                    }
                    else
                    {
                        var closeFileCmd = new CloseFileCmd(fileHandle);
                        Task<CloseFileCmd> closeHandle = _embeddedDevice.AddToExecutionQueue(closeFileCmd);
                        closeHandle.ContinueWith(closedHandle => tcs.SetResult(bytesTransfered));
                    }
                };
                readBlockTask.ContinueWith(recursiveDownload);
            });
            return tcs.Task;
}
{{< /highlight >}}

## Approach 2: Writing separate functions

Splitting callbacks into separate functions gives us a more readable control flow and also an easier way to achieve looping behavior. One drawback is that we have to pass parameters or use member variables in order to preserve context. Compared with lambda expressions, we are better off splitting functions but at the end of the day nothing can bring us a more natural control flow than approach 3.

{{< highlight csharp "style=paraiso-dark" >}}
   public class DownloadFile
   {
           private readonly EmbeddedDevice _embeddedDevice;
           private  DeviceFileInfo _fileInfo;
           private TaskCompletionSource<long> _tcs;
           long _bytesTransfered;
           private int _fileHandle;
           const int BlockSize = 32 * 1024;
    
           public DownloadFile(EmbeddedDevice embeddedDevice)
           {
               _embeddedDevice = embeddedDevice;
           }
    
    
           public Task Download(DeviceFileInfo fileInfo)
           {
               _tcs = new TaskCompletionSource<long>();
               _bytesTransfered = 0;
               _fileInfo = fileInfo;
               _fileHandle = 0;
    
               var openFileCmd = new OpenFileCmd(_fileInfo.Name, FileAccess.Read);
               Task<OpenFileCmd> fileHandleTask = _embeddedDevice.AddToExecutionQueue(openFileCmd);
    
               fileHandleTask.ContinueWith(OnFilehandleReceived);
    
               return _tcs.Task;
           }
    
           private void OnFilehandleReceived(Task<OpenFileCmd> fileHandleTask)
           {
               _fileHandle = fileHandleTask.Result.SuccessResult;
               var readBlockCmd = new ReadBlockCmd(_fileHandle, BlockSize);
               Task<ReadBlockCmd> firstBlock = _embeddedDevice.AddToExecutionQueue(readBlockCmd);
    
               firstBlock.ContinueWith(OnBlockReceived);
           }
    
           private void OnBlockReceived(Task<ReadBlockCmd> readBlockTask)
           {
               IEnumerable<byte> data = readBlockTask.Result.SuccessResult.ActualData;
               ProccessData(data);
               _bytesTransfered += data.Count();
    
               if (_bytesTransfered < _fileInfo.Length)
               {
                   var readBlockCmd = new ReadBlockCmd(_fileHandle, BlockSize);
                   Task<ReadBlockCmd> nextBlock = _embeddedDevice.AddToExecutionQueue(readBlockCmd);
                   nextBlock.ContinueWith(OnBlockReceived);
               }
               else
               {
                   var closeFileCmd = new CloseFileCmd(_fileHandle);
                   Task<CloseFileCmd> closeHandle = _embeddedDevice.AddToExecutionQueue(closeFileCmd);
    
                   closeHandle.ContinueWith(OnResourcesCleaned);
               }
           }
    
           private void OnResourcesCleaned(Task<CloseFileCmd> closedHandle)
           {
               _tcs.SetResult(_bytesTransfered);
           }
    }
{{< /highlight >}}

## Approach 3: Using C# 5.0 language constructs async/await

Async/await enables us to program asynchronous workflows in a linear way, that is, we can write our program as we would synchronously but without tying up a thread. If a method, called with await, has not yet completed, the execution immediately returns to the caller. When the method completes, the execution jumps back to the method where it left of, preserving all local state. What compiler does under the covers when it stumbles upon an await operator is very similar to yield return in an iterator, that is, a lot of boiler plate code gets generated in order to achieve pause/resume functionality using state machines.

{{< highlight csharp "style=paraiso-dark" >}}
  public async Task<long> Download(DeviceFileInfo fileInfo;)
  {
              long bytesTransfered = 0;
              const int blockSize = 32 * 1024;
   
              var openFileCmd = new OpenFileCmd(fileInfo.Name, FileAccess.Read);
              OpenFileCmd fileHandleTask = await _embeddedDevice.AddToExecutionQueue(openFileCmd);
              int fileHandle = fileHandleTask.SuccessResult;
   
              while (bytesTransfered < fileInfo.Length)
              {
                  var readBlockCmd = new ReadBlockCmd(fileHandle, blockSize);
                  ReadBlockCmd readBlockTask = await _embeddedDevice.AddToExecutionQueue(readBlockCmd);
                  byte[] bytesRead = readBlockTask.SuccessResult.ActualData.ToArray();
                  Proccess(bytesRead);
                  bytesTransfered += bytesRead.Count();
              }
   
              var closeFileCmd = new CloseFileCmd(fileHandle);
              await _embeddedDevice.AddToExecutionQueue(closeFileCmd);
              return bytesTransfered;
  }
  {{< /highlight >}}

  In summary, hiding callbacks with async/await language asynchrony support in C# 5.0 enables us to write code like we are used to and prevents complex asynchronous workflows (completion of multiple tasks, error handling…) from becoming unreadable and difficult to maintain. Even when targeting .NET 4 there is a [package](https://www.nuget.org/packages/Microsoft.Bcl.Async/1.0.16) on NuGet Microsoft.Bcl.Async which enables async/await keywords and includes the new Task based extension methods.