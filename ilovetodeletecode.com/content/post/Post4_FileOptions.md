+++
date = "2014-11-16T14:03:50Z"
title = "FileOptions Enumeration, Windows Cache Manager and Memory-Mapped Files (C#) Part 1"
slug  = "FileOptions-Enumeration-Windows-Cache-Manager-and-Memory-Mapped-Files-(CSharp)-Part-1"
+++
Because working with files is such a common and potentially expensive operation, every operating system introduces numerous levels of indirection when serving I/O requests. This is necessary in order to execute I/O operations in a reasonable time frame. But managing complexity through abstraction of various hardware and software layers that interact during file operations also has its disadvantages: you get a false impression that you don't need to know what is really going on under the hood :)

I needed to optimize a part of my code that was writing data to random locations in a binary file. After realizing that [FileOptions.RandomAccess](http://msdn.microsoft.com/en-us/library/system.io.fileoptions(v=vs.110).aspx) does not automagically do the trick and make seeks (in a loop!) run noticeably faster, I started to dig into the FileOptions Enumeration. A couple of days later, bursting with knowledge on internal working of Windows Cache Manager and speeding up random writes using [Memory-Mapped Files](http://msdn.microsoft.com/en-us/library/dd997372%28v=vs.110%29.aspx) I decided to write a two part blog post about it.

<!--more-->

## FileOptions Enumeration & The 5 Cache Flags

The description for FileOptions.RandomAccess on MSDN is pretty vague:

>"FileOptions.RandomAccess indicates that the file is accessed randomly. The system can use this as a hint to optimize file caching."

Even a general search for FileOptions.RandomAccess keyword did not result in any resources that would describe the attribute in a more detailed manner. With a little help of [.NET Reference Source](http://referencesource.microsoft.com/#mscorlib/system/io/filestream.cs,76ef6c04de9d0ed8) you can quickly check that the FileOptions parameter you apply to FileStream's constructor maps to next to last parameter (dwFlagsAndAttributes) of the native [CreateFile function](http://msdn.microsoft.com/en-us/library/windows/desktop/aa363858(v=vs.85).aspx). Lets check the flags from the native point of view.

| .NET                       | Native                    |
|----------------------------|---------------------------|
| FileOptions.RandomAccess   | FILE_FLAG_RANDOM_ACCESS   |
| FileOptions.SequentialScan | FILE_FLAG_SEQUENTIAL_SCAN |
| FileOptions.WriteThrough   | FILE_FLAG_WRITE_THROUGH   |
| FileOptions.NoBuffering    | FILE_FLAG_NO_BUFFERING    |
| FileAttributes.Temporary   | FILE_ATTRIBUTE_TEMPORARY  |

{{< figure src="/img/Post4_cache_layers.png">}}

These flags affect caching - not the FileStream's internal buffer mechanism, but caching on the OS level (Cache Manager). The picture on your right (borrowed from this [research paper](http://research.microsoft.com/apps/pubs/default.aspx?id=64538)) clearly shows that caching is a two stage buffering process. Every time you are reading/writing in .NET you are actually copying bytes between the two buffers. With this in mind it is also easier to understand the [difference between Flush() and Flush(true)](http://stackoverflow.com/questions/4921498/whats-the-difference-between-filestream-flush-and-filestream-flushtrue).

## Windows Cache Manager & FILE_FLAG_NO_BUFFERING

Each time you write or read data to/from disk drives the OS caches data unless FILE_FLAG_NO_BUFFERING is specified. In latter case your application has total control over the buffering process which is known as unbuffered file I/O. When inspecting .NET Reference Source 4.5.2 you can see that the FileOptions.NoBuffering enum is currently commented out. But the guard clause that validates FileOptions flags before calling native CreateFile function allows the value 0x20000000 and this value relates to the native flag FILE_FLAG_NO_BUFFERING. So the following initialization of FileStream would be legal:

{{< highlight csharp "style=paraiso-dark" >}}
var f = new FileStream("c:\\file.txt", FileMode.Open, FileAccess.Read, FileShare.None, bufferSize, (FileOptions)0x20000000);
{{< /highlight >}}

Files opened with FILE_FLAG_NO_BUFFERING do not go through the Cache Manager (all of the cache flags are ignored) and working with such files requires from our application to follow some rules regarding buffer alignment criteria. This is an [advanced topic](http://msdn.microsoft.com/en-us/library/windows/desktop/cc644950(v=vs.85).aspx) and goes beyond this post. For each file that is accessed through the Cache Manager an array of internal data structures need to be created and managed. The bigger the file, the more resources it takes to successfully support caching. That is why it's advised to use FILE_FLAG_NO_BUFFERING when working with very large files.

## Cache Manager Routines

Normally you would open files with Cache Manager enabled so let us look at its main routines that help us reduce I/O operations and enable our application to work faster:
1. Reading ahead
2. Lazy writing

## (Intelligent) Read Ahead

Cache Manager, keeps a history of the last two read request, for each file handle being used. With such history in mind it tries to determine the following patterns:

1. User is doing sequential forward reads.
2. User is doing sequential backward reads.
3. User is doing reads in step sizes (offset 0 10 bytes, offset 1000 10 bytes, offset 2000 10 bytes....)

If a pattern corresponds with history read requests then the Cache Manager can anticipate which bytes you are going to read next and that is a big performance improvement.

### How do cache flags affect Read Ahead?

| Flag                       | Description                                                                                                                                                                              |
|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| FileOptions.None           | General (previously described) pattern detection algorithm is used when no cache flags are applied.                                                                                      |
| FileOptions.SequentialScan | Cache manager does not try to determine any patterns. More aggressive read ahead is used - two times as much data is prefetched than normally (Do not use it when doing backward reads). |
| FileOptions.RandomAccess   | If you are constantly doing seeks, you are wasting time and memory pre-emptively reading data that your application will probably not need. This option disables read-ahead.             |

## Lazy Writing (Write Back Cache)
In order to flush as much data as possible to disk at once, write operations are piled up in memory until the decision is made by the Cache Manager. The decision about, which dirty pages should be flushed to disk and when, is made by the Cache Manager's lazy writer function that executes on a separate system worker thread once per second.

### How do cache flags affect Lazy Writing?

| Flag                     | Description                                                                                                                                                                                                                                                                                                       |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| FileAttributes.Temporary | (Disable lazy writing) File system tries to keep the data in memory and flushes it only in case of memory shortage. This attribute is commonly combined withFileOptions.DeleteOnClose. Both of these flags combined are ideal for small temporary files since there will most likely be no hits to the hard disk. |
| FileOptions.WriteThrough | Cache manager still caches data but only for file-read operations. Write changes are written to disk instantly. This is useful for reducing the risk for data loss. Warning! A lot of [resources](http://blogs.msdn.com/b/oldnewthing/archive/2014/03/07/10505524.aspx) are stating that all SATA drives ignore this flag.                                                                 |


## Cache Usage Performance Counters

There are a lot of counters in the Cache section of Performance monitor that can help you evaluate cache usage. The two that relate to this blog post the most are Read Aheads/sec and Lazy Write Flushes/sec. The first one displays the rate at which the Cache Manager detects that a file is being accessed sequentially and the latter displays the rate of periodical flushes that are performed by the lazy writers thread. If you want to see all flushing activity use Data Flushes/sec counter.

## Summary

When you step out of your comfortable .NET bubble, wonderful things can happen. I discovered way more information when learning about native flags that I did with FileOptions Enumeration. If you, like myself, never did any native C++ Windows programming, reading about internal mechanisms like Cache Manager and how to control it really helps you to better understand and optimise your managed .NET application. 


