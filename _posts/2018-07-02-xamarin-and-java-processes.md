---
layout: post
title: Xamarin and Java Processes (root)
author: Tomasz Cielecki
comments: true
date: 2018-7-2 21:40:00 +0200
tags:
- Xamarin
- Xamarin.Android
- Java
- Process
- dotnet
---

I have seen this question pop up from time to time, where someone asks how to run a native process on Android.
This is usually because they want to run something with root permissions, which could allow you to do various things, such as reading files you are usually not allowed. This could be reading VPN state from `/data/misc/vpn/state`. Blocking Advertisements by adding domains to `/etc/hosts` and much more.

This blog post will show you how to start a Java process and read the exit code, and the output from the process.

> Note: If you want to try this yourself, you will need to have a rooted device. I like to run [LineageOS][los] on my devices. They support a large amount of popular devices.
On LineageOS you can install [SU by grabbing it from the Extras][lossu] web page and flashing using your recovery. This is up to the reader to do.

## Linux Processes
Linux is the operating system Android is based on. A standard process on Linux will use

* stdin to read input from the console (i.e. keyboard)
* stdout to write to the console
* stderr to write errors to the console

These are in fact file handles to `/proc/<pid>/fd/{0,1,2}`, where `<pid>` is your process id and 0, 1, 2 are stdin, stdout, stderr respectively. You should not worry about this as this is usually handled by the OS.

A process has an exit code. 0 means everything was OK, anything else than 0 means it probably failed.

## Starting a Process On Android
On Android you can start a native Process using with `Java.Lang.Runtime.GetRuntime().Exec(<command>)`, which will return a `Java.Lang.Process`. Alternatively (also preferred way) if you need more control you can use `Java.Lang.ProcessBuilder`, which allows you to redirect stdin, stdout, stderr. Its `Start()` method, similarly to `Exec()` gives you a `Java.Lang.Process`. This object reflects the Linux process and has streams for stdin, stdout and stderr. These are our way into reading what the process writes out.

[Xamarin][xam] has some nice helpers for us in the [`Java.Lang.Process`][proc] they expose. Instead of blocking when we want to wait for the process to finish, they have added `WaitForAsync()`, which is very nice. Also the streams for stdin, stdout and stderr are nice C# Streams that you can read and write to very easily. Namely `InputStream`, `OutputStream` and `ErrorStream`.

```csharp
var builder = new ProcessBuilder("echo", "hello");
var process = builder.Start();
var exitCode = await process.WaitForAsync();
```

The example above runs the `echo` command, which just echoes what follows after. In this case "hello". The `WaitForAsync()` method returns us a `Task<int>` with the exit code of the process.

> Keep in mind that `WaitForAsync()` also can throw exceptions such as `IOException` and `SecurityException` depending on what you are doing.

### Reading From The Process
When we start a Process, it should be considered that we start a process and execute some commands that redirect into this Process. Meaning most of the time, when we want to read the result from the commands we run, we will actually need to read from `InputStream`, rather than `OutputStream` which would be the usual stdout. However, since our Process is the one starting other Processes, then their output stream gets redirected into our input stream.

Anyways, reading the stream is super easy.

```csharp
using (var outputStreamReader = new StreamReader(process.InputStream))
{
    var output = await outputStreamReader.ReadToEndAsync();
}
```

Bingo! If we ran this on the `echo hello` process we created further up, `output` would be `hello` in this case. Neat!

## Running a Process as Root
Running a process as root is super easy. Just add `su` and `-c` to the commands you execute as follows.

```csharp
var builder = new ProcessBuilder("su", "-c", "echo hello");
```

`-c` means that the root process `su` (super user aka. root) should run a specific command. Notice that `echo` and `hello` became one argument. This is because we are redirecting into `su`'s stdin. Apart from that everything else stays the same.

Most Android OS's with root, will show a dialog when root is requested.

[![Root dialog][root]][root]

## Reading Large Files
Reading large files by using `cat` to our Process stdin might be inefficient. You could consider using the `ProcessBuilder` to redirect the stdin directly to a file at a destination you have better access to.

```csharp
var myFile = new File("/some/path");
builder.RedirectInput(myFile);
```

Alternatively you could do the same with the `InputStream` and read it to a file.

```csharp
using (var inputStream = process.InputStream)
using (var fileStream = File.Create("/some/path"))
{
    await inputStream.CopyToAsync(fileStream);
}
```

## Convenience Helper Method
Here is a little helper method for running simple commands and getting their result.

```csharp
async Task<(int exitCode, string result)> RunCommand(params string[] command)
{
    string result = null;
    var exitCode = -1;
    try
    {
        var builder = new ProcessBuilder(command);
        var process = builder.Start();
        exitCode = await process.WaitForAsync();

        if (exitCode == 0)
        {
            using (var inputStreamReader = new StreamReader(process.InputStream))
            {
                result = await inputStreamReader.ReadToEndAsync();
            }
        }
        else if (process.ErrorStream != null)
        {
            using (var errorStreamReader = new StreamReader(process.ErrorStream))
            {
                var error = await errorStreamReader.ReadToEndAsync();
                result = $"Error {error}";
            }
        }
    }
    catch (IOException ex)
    {
        result = $"Exception {ex.Message}";
    }

    return (exitCode, result);
}
```

Usage would be something like.

```csharp
var (exitCode, result) = await RunCommand("su", "-c", "cat /data/misc/vpn/state");
```

This would return with the `exitCode` 0 if everything went well, and the string `result` would contain something like.

```
ppp0
10.0.0.42/32
0.0.0.0/0
109.226.17.2 144.117.5.51
```

Nice! Now you should be an expert in native processes and how to launch them from your Android App. It is up to you the reader as an exercise to figure out how `OutputStream` works. Enjoy!

[los]: https://lineageos.org/ "LineageOS Android Distribution web site"
[lossu]: https://download.lineageos.org/extras "LineageOS Extras Downloads"
[root]: {{ site.url }}/assets/images/root.jpg "Root dialog"
[xam]: https://xamarin.com "Xamarin web site"
[proc]: https://developer.xamarin.com/api/type/Java.Lang.Process/ "Java.Lang.Process Xamarin documentation"
