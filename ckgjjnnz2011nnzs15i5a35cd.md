## “DLL Hell”: Tips & Tricks to Avoid it in VAST

Delegating tasks from high-level languages like the [VAST Platform (VA Smalltalk)](https://www.instantiations.com/products/vasmalltalk/index.html) to languages like C, C++, Rust via some kind of FFI (Foreign Function Interface) is becoming more and more common.

Ideally, you would like to have everything implemented in your preferred high-level language, but I believe in using the appropriate tool for each problem. Sometimes you need more performance, or sometimes you want to save yourself the costs of implementing and maintaining a library that’s already implemented in another language. The reasons could be many, but in this case, you will end up making a thin FFI binding (a.k.a wrapper) against that library.

Then, when you are in your high-level language and you want to call, for example, a function from a C library compiled into a DLL, you need to search for that DLL (Dynamic-link library) and load it into your applications process space.

That may sound like an easy step, but it can be one of those situations where developers spend a lot of time trying to figure out what is wrong: why a DLL couldn’t be loaded, why another DLL (with a different path) was loaded instead of the expected one, and so forth. This is a frequent problem that is widely known as “[DLL Hell](https://en.wikipedia.org/wiki/DLL_Hell)“.

[![](https://i0.wp.com/marianopeck.blog/wp-content/uploads/2020/10/1_DMv_Km8Ml49BuRXwWHydpg.png?resize=335%2C335&ssl=1)](https://i0.wp.com/marianopeck.blog/wp-content/uploads/2020/10/1_DMv_Km8Ml49BuRXwWHydpg.png?ssl=1)

In this post I’ll show some tips, tricks, and scripts to help you get out of this situation when loading DLLs from the VAST Platform.

## Troubleshooting VAST .ini specifications

In VAST, most of the time, the library filename of the different wrappers are specified in the main `.ini` file under the section `[PlatformLibrary Name Mappings]`.

For example, below is an extract of it:

```

[PlatformLibrary Name Mappings]
; Almost all .DLL files referenced by VA Smalltalk have aliases (logical names) that are mapped here.
;
; The keywords (logical names) are case sensitive; the values (dll names) are not.
Abt_Primitives=
AbtNativeQ=abtvof40
AbtNativeV=abtvmf40
BROTLI_LIB=esbrotli40
CgImageSupport=escgi40
CRYPTO_LIB=libcrypto

```

That way you customize the “physical name” (ex. `libcrypto`) of a given “logical name” (ex. `CRYPTO_LIB`). You can use different names or even full paths…for example:

```

CRYPTO_LIB=my-special-libcrypto
CRYPTO_LIB=c:\Users\mpeck\Documents\Instantiations\libs\OpenSSL_x64\libcrypto

```

So…let’s see the typical issues and possible solutions.

### Be certain of which .ini file VAST is reading

Before you start digging into why a certain DLL didn’t load properly and you start randomly editing `.ini` files… just be sure you are editing the correct one.

This may be obvious, but I have seen cases where VAST is started from a chain of `.bat` files (where it’s hard to get to the exact line that ends up executing VAST) or cases where there are also many custom `.ini files` (read by end-user applications). In these situations, it’s not always easy to “know” which is the exact `.ini` file being read by VAST itself at startup to map logical to physical names.

The first solution here is to ask the system which exact `.ini` file was read at startup:

```smalltalk

Transcript 
    cr; 
    show: 'INI File: ';
    show: System primitiveIniFileFullPathAndName asString.

```

This will only help if you are having the problem on a development image or on a runtime one that you are able to modify and re-package it to print this information.

However, if the problem is in a runtime image you can’t re-package, then it will help to know the exact algorithm the VM does to look for the primary `.ini` file:

```C

 /**
 * Finds the .INI filename.
 * This is called AFTER EsFindImageFileName().
 * Algorithm:
 * 1. If globalInfo->iniName has a name in it (came from -ini: commandline switch),
 * look for (globalInfo->iniName).ini file, answer TRUE if present; else answer FALSE.
 * 2. Look for (globalInfo->imagePath)(globalInfo->imageName).ini, answer TRUE if present.
 * 3. Look for <exePath><exeName>.ini, answer TRUE if present.
 * 4. Look for es.ini, answer TRUE if present.
 * 5. Answer FALSE.
 */
BOOLEAN EsFindIniFileName(EsGlobalInfo *globalInfo)

```

This way, you can manually check for the existence of these files step-by-step until you find which one is being used.

> BTW: VAST 2021 (v10.x.x) is including this information as part of the default walkback header.

### Be sure your .ini edits are being taken into account

Again, this sounds obvious, but I’ve seen end-user applications that override the default VAST startup mechanism, have their own `.ini` reading, hardcoded settings, just to name a few…

So… once you start modifying the correct `.ini` file to troubleshoot the problem, be sure that those changes are being taken into account. The below code queries VAST to know exactly what have `CRYPTO_LIB` been mapped to after startup:

```Smalltalk

| logicalName aliases sharedLibs lib |
logicalName := 'CRYPTO_LIB'.
aliases := PlatformLibrary classPool at: 'Aliases'.
sharedLibs := PlatformLibrary classPool at: 'SharedLibraries'.
Transcript show: 'Alias Key: ', logicalName, ' Value: ', (aliases at: logicalName); cr.
lib := sharedLibs detect: [:each | each logicalName= logicalName] ifNone: [nil].
Transcript show: 'SharedLibraries LogicalName: ', lib logicalName, ' PhysicalName: ', lib physicalName. 

```

So for example, if I edited my `.ini` file with this line:

```

CRYPTO_LIB=c:\Users\mpeck\Documents\Instantiations\libs\OpenSSL_x64\libcrypto

```

But above code prints `libcrypto` then it means that for some reason my changes are ignored. So….be sure that above code prints what you specified in the file.

## Troubleshooting DLL load failure

Assuming you are sure which main `.ini` VAST is reading and that your changes would be taken into account, you could now start troubleshooting why a DLL failed to load or why a wrong one has been.

### Windows DLL lookup order

The most common problem when loading a DLL is that either Windows couldn’t find it through its lookup mechanism or it did but it’s a different version than the one you would expect (which could be problematic).

If you look at the official [documentation about the lookup mechanism](https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order#search-order-for-desktop-applications) you will see that there are many factors that could affect it: registry keys such as `KnownDLLs or SafeDllSearchMode`, custom load via `LOAD_WITH_ALTERED_SEARCH_PATH`, etc. However…. most of the time you end up using the “default” behavior.

Frequently, it’s hard to identify the working directory, what/where the VAST executable binary is, what directory is `GetSystemDirectory` or `GetWindowsDirectory`, etc.

The below script tries to mimic the Windows lookup order and displays what would be the directory for each step:

```Smalltalk

| string m p | 
string := String new: 256.

m := OSHmodule getModuleHandle: nil.
m isNull ifFalse: [
  p := String new: 256.
  m getModuleFileName: p cbFileName: p size.
  Transcript show: '1: The directory containing the EXE file: ', p trimNull ; cr.
] ifTrue: [Transcript show: 'Can''t find module!'; cr.].

OSCall new getSystemDirectory: string cbSysPath: 256.
Transcript show: '2: GetSystemDirectory : ', string; cr.

OSCall new getWindowsDirectory: string cbSysPath: 256.
Transcript show: '4: GetWindowsDirectory : ', string; cr.

Transcript show: '5: Current Directory : ', CfsDirectoryDescriptor getcwd; cr.

Transcript show: '6: $Path : ', 'Path' abtScanEnv; cr.

```

And that script could print something like this:

```

1: The directory containing the EXE file: Z:\Common\Development\VAST\10.0.0x64-b466\10.0.0x64\abt.exe
2: GetSystemDirectory : C:\WINDOWS\system32
4: GetWindowsDirectory : C:\WINDOWS
5: Current Directory : Z:\Common\Development\Images\10.0.0.x64-b466-dev
6: $Path : Z:\Common\Development\VAST\10.0.0x64-b466\10.0.0x64\;Z:\Common\Development\VAST\10.0.0x64-b466\10.0.0x64;C:\Python27\;C:\Python27\Scripts;C:\WINDOWS\system32;C:\WINDOWS;C:\WINDOWS\System32\Wbem;C:\WINDOWS\System32\WindowsPowerShell\v1.0\;C:\Program Files\PuTTY\;C:\WINDOWS\System32\OpenSSH\;C:\Program Files (x86)\Git\cmd;C:\Program Files\LLVM\bin;C:\Program Files\CMake\bin;C:\Users\mpeck\AppData\Local\Programs\Python\Python37\Scripts\;C:\Users\mpeck\AppData\Local\Programs\Python\Python37\;C:\Users\mpeck\AppData\Local\Microsoft\WindowsApps;C:\Program Files\PuTTY;C:\Users\mpeck\AppData\Local\Microsoft\WindowsApps;

```

Now you know which directories Windows will look for and in which order.

> Pro Tip: Are you running VAST 32-bit on Windows 64-bit? If so, bear in mind that the function `GetSystemDirectory` will answer `C:\WINDOWS\system32` even though the real physical directory mapped to it is `C:\Windows\SysWOW64\`. You can read [this post](https://www.howtogeek.com/326509/whats-the-difference-between-the-system32-and-syswow64-folders-in-windows/) for more details.

Oh, by the way, do you also want to grab any of the involved Windows registry keys that could affect the load order? You can do that from VAST too!

The below example gets the value of `SafeDllSearchMode`

```smalltalk

"Registry Hive: HKEY_LOCAL_MACHINE
Registry Path: \System\CurrentControlSet\Control\Session Manager\

Value Name: SafeDllSearchMode

Value Type: REG_DWORD
Value: 1
"

| subKey buffer key answer valueName |

subKey := ByteArray new: 4.
buffer := String new: 256.
key := 'System\CurrentControlSet\Control\Session Manager'.
valueName := 'SafeDllSearchMode'.

 answer := (OSHkey immediate: PlatformConstants::HkeyLocalMachine)
    regOpenKeyEx: key asParameter
    ulOptions: 0
    samDesired: PlatformConstants::KeyQueryValue
    phkResult: subKey.

 answer = PlatformConstants::ErrorSuccess
  ifTrue: [
    subKey := (OSHkey immediate: subKey abtAsInteger )
        regQueryValueEx: valueName
        lpReserved: 0
        lpType: nil
        lpData: buffer asParameter
        lpcbData: (OSUInt32 new uint32At: 0 put: buffer size; yourself)
asParameter.

  (buffer asByteArray uint64At: 0) inspect.
  ]. 

```

### What’s the path of my loaded DLL?

Another typical problem when dealing with DLLs is that for certain VAST images or machines it works, but for another it doesn’t. If you didn’t specify a full path for that DLL in the `.ini` file, then it would be useful to know which exact DLL (with path) is being picked-up by Windows on both cases.

For that, you can use below VAST script:

```smalltalk

"Disclaimer: this assumes you already loaded the dll into the running process"
 | m p |
 m := OSHmodule getModuleHandle: 'libeay32.dll' asPSZ.
m isNull ifFalse: [
  p := String new: 256.
  m getModuleFileName: p cbFileName: p size.
  Transcript show: 'Loaded DLL: ', p trimNull.
] ifTrue: [Transcript show: 'Can''t find module!'; cr.] 

```

Which prints something like this:

```

Loaded DLL: z:\Common\Development\Images\10.0.0.x64-b466-dev\libeay32.dll

```

Another way to do this (outside of VAST) is using the “Process Explorer” tool as explained in [this previous post](https://martinezpeck.hashnode.dev/troubleshooting-applications-running-on-windows-ckg18kr3s01arw6s13jcdfipf).

### Check DLL dependencies

Many times I think “I swear the DLL is there but VAST fails to load it”. It could be that the DLL you are trying to load is in the correct place and Windows is indeed trying to load it, but Windows may be failing to load it because a **dependency** of that DLL couldn’t be found.

For example…say we tried to load `ssleay32.dll` and that failed. [As shown in a previous post](https://martinezpeck.hashnode.dev/troubleshooting-applications-running-on-windows-ckg18kr3s01arw6s13jcdfipf), we can use the `dumpbin` tool to identify the dependencies of that DLL and be sure that we are not missing anything:

[![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-16-at-4.33.55-PM.png?resize=748%2C327&ssl=1)](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2020/10/Screen-Shot-2020-10-16-at-4.33.55-PM.png?ssl=1)

As you can see, in this example, `ssleay32.dll` depends on `libeay32.dll` (as well as others). So… if `libeay32.dll` or any other of the dependencies cannot be found, the load of the root DLL (`ssleay32` in this example) will just fail.

Always check dependencies.

### Verify the bitness of your executable and the DLL

A variation from the previous problem: “I swear the DLL is there but VAST fails to load it”. If you use both 32 and 64-bit programs you may eventually face this issue. When you are looking at a DLL and thinking Windows should have been able to load it, but it couldn’t even though the dependencies are fine, the other thing to check is the bitness. VAST 64-bit is only able to load 64-bit DLLs…and same for goes for 32-bit.

For this, I don’t use a VAST script but rather a [simple technique using a text editor](https://superuser.com/a/889267). Very useful to keep in mind.

## Conclusions

Hopefully these tips and tricks are useful for you! Do you have another trick to share? I would love to hear about it.