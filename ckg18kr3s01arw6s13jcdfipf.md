## Troubleshooting Applications Running on Windows

Over the past few months, I’ve been troubleshooting hard problems that have appeared when running the [VAST Platform (VA Smalltalk)](https://www.instantiations.com/products/vasmalltalk/index.html) on Windows. Some of the problems were indeed bugs (like sockets leaking under a particular scenario) and some were just Windows or customer issues.

Regardless of where the problem was, I learned much about certain tools and tricks that could help me again in the future or possibly help others too. Hence, the reason for this post.

## Process Explorer

I have heard before about the “[Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer)” tool, but I had not used it until recently. This is a really small application (3MB) that allows you to analyze many aspects of the running processes: loaded DLLs, threads, opened file handlers, and many more.

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-08-at-10.51.29-AM.png?resize=748%2C645&ssl=1)

![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-08-at-10.52.37-AM.png?resize=748%2C653&ssl=1)

In my case, the two features I used the most were the ability to know:

(1) Which exact DLLs (with full path) were loaded in the process (for one of the problems a wrong version of the DLL was being loaded)

(2) The list of opened file handlers (another problem was a leak getting close to the max allowed file handlers per process)

> **PRO TIP:** You don’t even need to install “Process Explorer” (sometimes installing programs on large companies isn’t easy) as it comes in a kind of “portable” executable that you just double click.

## Dumpbin

A shared library could (and very likely will) depend on other shared libraries (dependencies).

One typical problem when you dynamically load shared libraries is that the load of the library fails because the operating system can’t find one or more of the required libraries.

In Linux, we would normally use something like `ldd mylib.so` and check for the not found. But for me, it wasn’t obvious how to do that on Windows. I knew about the “[Dependency Walker](https://www.dependencywalker.com/)” but I also understand that it does not support Windows 10 very well, nor is it officially supported by Microsoft.

Finally, I found the “[dumpbin](https://docs.microsoft.com/en-us/cpp/build/reference/dumpbin-reference?view=vs-2019)” tool which _is_ officially supported by Microsoft and allowed me to do what I needed…and more. If you have Microsoft Visual Studio, then you probably already have the tool (although it’s very very hidden). Otherwise, you can install the “[Windows 10 SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk/)“.

The feature I needed this time was `/dependents` as you can see in below example:

![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-08-at-11.18.33-AM.png?resize=748%2C382&ssl=1)

You can check the documentation for all the possible command line arguments.

## The Microsoft Error Lookup Tool

I am a little bit ashamed that I discovered this tool only just recently. A customer reported a Windows error in hexadecimal format. Sure, I can Google that exact hexa error code, but the [Microsoft Error Lookup Tool](https://docs.microsoft.com/en-us/windows/win32/debug/system-error-code-lookup-tool) is very handy. You can just pass the hexa as an argument, and I will show all the places Windows found for that error and also print in which C header file it’s defined.

Here is a simple example with the hexa code 0x80070583 I was investigating due to a COM initialization problem:

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-08-at-11.34.27-AM.png?resize=748%2C222&ssl=1)

And here is an example with more than 1 result:

![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-08-at-11.36.28-AM.png?resize=748%2C325&ssl=1)

## Forcing Windows to re-load manifest files 

VAST Platform 9.2 comes with a full native HiDPI support for Windows. For that, it needs some lines in the manifest files:

```xml
 <compatibility xmlns="urn:schemas-microsoft-com:compatibility.v1"> 
   <application> 
       <!-- Windows 10 -->
       <supportedOS Id="{8e0f7a12-bfb3-4fe8-b9a5-48fd50a15a9a}"/>
       <!-- Windows 8.1 -->
       <supportedOS Id="{1f676c76-80e1-4239-95bb-83d0f6d0da78}"/>
       <!-- Windows 8 -->
       <supportedOS Id="{4a2f28e3-53b9-4441-ba9c-d69d4a4a6e38}"/>
       <!-- Windows 7 -->
       <supportedOS Id="{35138b9a-5d96-4fbd-8e2d-a2440225f93a}"/>
       <!-- Windows Vista -->
       <supportedOS Id="{e2011457-1546-43c5-a5fe-008deee3d3f0}"/> 
   </application> 
 </compatibility>
 <application> 
   <windowsSettings> 
     <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitor</dpiAwareness> 
     <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">True/PM</dpiAware>
   </windowsSettings> 
 </application>
```

One problem we detected is that Windows seems to cache the data of the manifest file even if you edit it and reboot. We researched a lot for possible workarounds and a good one was found in [this post](http://csi-windows.com/blog/all/27-csi-news-general/245-find-out-why-your-external-manifest-is-being-ignored). However, we found an easier trick that worked for us. Example:

![](https://i0.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-08-at-11.52.57-AM.png?resize=748%2C218&ssl=1)

## Wrong Locale when running your program as a Windows Service

If you normally use the “US” region on Windows you may not have faced this issue. If you have custom Locale settings (ex. german-german) you may experience they are not being properly detected by your program when running as a Windows Service, but they are when run from a console.

When you run a program from a normal CMD console, it uses the normal Windows’ user logged in (ex. “Mariano” user). But when you register your application as a service, the user to log on, will probably be “Local System” (the default). You can confirm this by looking at your registered service and the column “Log On As”.

The problem is that the “Local System” settings for Locale are not necessary the same as your current Windows user (ex. “Mariano”). The solution to this problem is to specify the Locale information also to the “Local System” user. [Here](http://www.perennitysoft.com/support/kb/faq.php?id=86) is how you do it.

## OpenSSL for Windows

OpenSSL is becoming more and more critical every day and not only on HTTP but in SMTP, MQ, SFTP, SSH, just to name a few. As you probably know, [there is no “official” binary download for Windows](https://www.openssl.org/community/binaries.html). That means that there are [a few different 3rd party places](https://wiki.openssl.org/index.php/Binaries) from where you can download pre-compiled binaries. You can even compile it yourself. This, combined with the fact mentioned above that OpenSSL is become more and more used, could be nightmare if you are trying to load OpenSSL dynamically and relying on [Windows DLL lookup mechanism](https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order#search-order-for-desktop-applications).

The mentioned Process Explorer may help you realize which exact OpenSSL dll you are loading, but after that, it would be helpful to know more about that dll. Which version was it? Which parts were included when it was compiled? To help resolve those questions, we found this great website: [https://www.howsmyssl.com/](https://www.howsmyssl.com/). You can do a normal HTTP GET with your favorite client (curl, wget, a web browser, etc) and it will answer a nice HTML response with a lot of info. Example:

![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-08-at-12.11.58-PM.png?resize=748%2C448&ssl=1)

## Conclusion

This will probably just be a reminder for “future me” when I encounter these problems again. But in the meantime, I hope this was helpful for others too.

Thanks for reading.