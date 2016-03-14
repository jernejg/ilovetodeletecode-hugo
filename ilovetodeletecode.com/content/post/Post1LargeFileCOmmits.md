+++
date = "2013-10-31T12:53:22Z"
title = "Prevent large file commits with PowerShell and Mercurial hooks"
slug = "prevent-large-file-commits-with-powerShell-and-mercurial-hooks"
tags = [ "Development", "Go", "profiling" ]
topics = [ "Development", "Go" ]
+++

Because of Mercurial's immutable historical record of project files there is no way to make a committed file completely disappear from the history tree. Our only hope in this case it's not Obi-Wan but the hg rollback command if we act quickly. Since this command only works for one most recent operation [(How to keep a Mercurial repository small?)](http://stackoverflow.com/questions/3288865/how-to-keep-a-mercurial-repository-small/) it will not work if we already committed one or more changes after the change that we would like to completely remove [(Accidentally committed a large amount of raw data in Mercurial, how do I keep it from overloading my Bitbucket repository?)](http://stackoverflow.com/questions/8466669/accidentally-committed-a-large-amount-of-raw-data-in-mercurial-how-do-i-keep-it). Our problems escalate even faster if we already pushed or pulled changes to a remote repository and our bad commit propagated to other collaborators.

Two most common cases of brown paper bag commits [(a commit that is so bad you want to pull a brown paper bag over your head)](http://hgbook.red-bean.com/read/finding-and-fixing-mistakes.html#sec:undo:aaaiiieee) are accidentally committing sensitive information files (passwords, connection strings...) and large files. The latter case is less critical because it only increase s repository size and the time it takes to perform operations like pull or clone.

If you are interested in how to prevent yourself from committing large files and are more comfortable with PowerShell then please read on. If you are more of an indentation than squirrely brackets guy then you can check this similar solution in Python - [Mercurial hook to disallow committing large binary files](http://stackoverflow.com/questions/2551719/mercurial-hook-to-disallow-committing-large-binary-files).

<!--more-->

## Workflow

Our solution has a simple 3 step workflow

1.  On each commit retrieve files that have been added to the repo in current change set.
2. Get the number of files that have size greater than maximum allowed size..
3. Roll back or complete commit transaction based on the information from step 2.

## Controlling repository events with hooks

Triggers that allow you to perform custom actions in response to events that occur on a Mercurial repository are called hooks. Because we need to execute a custom script every time a commit occurs there are two hooks that might interest us.

1. Commit - runs after a new changeset has been created in the local repository.
2. Pretxncommit - runs after a new changeset has been created in the local repository, but before the transaction is completed.

For obvious reasons we will go with pretxncommit. This hook will not only call our custom PowerShell script every time a commit, but it will also pass parameters in form of environmental variables to it. Each Hook parameter gets prefixed with "HG_”. We can list all parameters that are available from Mercurial context applying a filter to PowerShell "Env" drive's content:

{{< highlight PowerShell "style=paraiso-dark" >}}
dir -path:"env:" |  Where-Object {$_.Name -like "HG_*"} |  Select -Property Name,Value
{{< /highlight >}}

Since pretxncommit is a controlling hook we can control the success of the operation using exit codes from our PowerShell script. Return a non-zero to roll back the transaction or 0 to complete it. We will exit our PowerShell script using the number of files greater than maximum size allowed.

## Defining the pretxncommit Hook as an external program

There are two ways in which a Hook gets executed. In our case the execution is "shell out" to another process. An alternative would be a Python function that is executed within the Mercurial process. In-process hooks are faster and have complete access to Mercurial API but you have to write code in Python.

Hooks are defined in the [hooks] section of your local (hg\hgrc) or global (%USERPROFILE%\mercurial.ini) settings file. Each hook is defined in a type, name and action like format. You can edit settings file directly or use TortoiseHg settings dialog:

{{< highlight Bash "style=paraiso-dark" >}}
[hooks] 
pretxncommit.fileSizeCheckHook = powershell.exe -File "C:\Users\Jernej\Documents\WindowsPowerShell\pretxncommit.ps1"
{{< /highlight >}}

{{< figure src="/img/Post1_Configure_Hook.png">}}

#### pretxcommit.ps1
{{< gist jernejg 7166587 >}}

As you can see I use a custom simple function called Get-HgAddedLargeFile to integrate Mercurial with PowerShell and to parse the output of hg status to PowerShell objects. You can also check out [posh-hg](https://github.com/JeremySkinner/posh-hg) from Jeremy Skinner that deals with Mercurial/PowerShell integration on a more advanced level.

{{< figure src="/img/Post1_hook_output.png">}}

Because our hook runs in a separate process the detailed error information we printed to the console window in our previous case doesn't get pulled to TortoiseHG log window instead just the default error is printed.

{{< figure src="/img/Post1_commit_abort.png">}}

We can solve this by calling a message box from PowerShell that will provide additional information about problematic files and in addition to that provide us with a dialog where we can decide on further action [pretxncommit_messagebox.ps1](https://gist.github.com/jernejg/7196705).

{{< figure src="/img/Post1_message_box.png">}}

## Hooks do not propagate

For the end let me remind you that hooks are not version controlled and do not propagate when you clone or pull from a repository. When collaborating with other people you can't make them to use the same hooks as you.

