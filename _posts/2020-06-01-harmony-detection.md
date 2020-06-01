---
title: "Prevent and understand managed hooking with Harmony"
categories:
	- C#
	- Hooking
	- Anti-RE
tags:
	- c#
	- hooking
	- managed
	- harmony
	- monomod
	- anti-re
	- patching
---

## Introduction

Hooking is a widely known technique which can be used for countless different purposes.

But how many people know that you can patch any .Net method at runtime and hook them to modify parameters, return values or even supress calls?

Hooking can be used to bypass DRM and other licensing methods within every .Net assembly.
Web requests can be sniffed and modified regardless of SSL encryption.
And thats just what came into my head when i first thought about it.

In order to prevent people analysing or modifying our application we need to know how armony and the underlying MonoMod Framework work.
With the gained knowledge we will try to address several ways on how to detect the presence of a hooked (or patched) method at runtime.

### What is Harmony

Harmony is an easy to use open source library with the ability to alter and extend the functionality of all available assemblies in a managed application.

It is based upon the MonoMod Framework which basically is the "swiss army knife" when it comes to modding a .Net assembly.

You can read more about those libraries her:

- [Harmony]("https://github.com/pardeike/Harmony")
- [MonoMod]("https://github.com/MonoMod/MonoMod")

### How does it work

Basically, harmony takes any method and copies all the opcodes into a new dynamicly created method which will then be used as the new "original" method.

This is done because the inital method will no longer be usable after the patching is done.
The first bytes of the initial method will be overriden with a jump to a newly created wrapper.

The wrapper method will then call a `prefix` method right before the original method will be called and a `postfix` method right after.

The `prefix` and `postfix` methods are the ones implemented by the user of the library which of course can alter and use the parameters and return values provided to the initial value.

---

### Detecting a hooked method

To detect a hooked method we would first need to optain a pointer to the compiled code.

Most of us probably know the highly useful `Marshal` class when you have dealt with unmanaged code or structures already.

It contains the [Marshal.GetFunctionPointerForDelegate]("https://docs.microsoft.com/de-de/dotnet/api/system.runtime.interopservices.marshal.getfunctionpointerfordelegate?view=netcore-3.1") method which **converts** a given delegate to a function pointer.

When you read the above documentation it may be clear that a conversion is not what we want.
The returned function pointer isn't of any use for us because it is a pointer to a P/Invoke wrapper for the given delegate which can be used with platform interop.

What we need is a pointer to the compiled managed method.
We want to use the same pointer `MonoMod` used to patch the method.

To get what we need we can use the [RuntimeMethodHandle.GetFunctionPointer]("https://docs.microsoft.com/en-us/dotnet/api/system.runtimemethodhandle.getfunctionpointer?view=netcore-3.1") method.

```cs
private static IntPtr GetMethodStart<T>(T target) where T : Delegate
{
    var method = target.Method;

    RuntimeHelpers.PrepareMethod(method.MethodHandle);

    return method.MethodHandle.GetFunctionPointer();
}
```

#### Edge cases

MonoMod handles some other cases within their `GetMethodStart` function which i do not want to explain in this post.

To implement the same behavior we need to use the `Ldftn` IL-opcode on some methods.

I used a simple cache and a `DynamicMethod` to retreive the function pointer on those special ones.

```cs
using System;
using System.Collections.Generic;
using System.Reflection;
using System.Reflection.Emit;
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;
using System.Threading;

private static IntPtr GetMethodStart<T>(T target) where T : Delegate
{
    var method = target.Method;

    if (method.IsVirtual && (method.DeclaringType?.IsValueType ?? false))
    {
        RuntimeHelpers.PrepareDelegate(target);

        var dm = DynamicHelper.GetDynamicLdftnMethodDelegate(method);

        return dm();
    }
    else
    {
        RuntimeHelpers.PrepareMethod(method.MethodHandle);

        return method.MethodHandle.GetFunctionPointer();
    }
}

private static class DynamicHelper
{
    private static readonly Dictionary<MethodInfo, Func<IntPtr>> _cache = new Dictionary<MethodInfo, Func<IntPtr>>();
    private static readonly object _lock = new object();

    public static Func<IntPtr> GetDynamicLdftnMethodDelegate(MethodInfo info)
    {
        if (info == null) throw new ArgumentNullException(nameof(info));

        lock (_lock)
        {
            if (_cache.ContainsKey(info))
            {
                return _cache[info];
            }

            // setting the owner type and skipVisibility is very important!
            var dm = new DynamicMethod(string.Empty, typeof(IntPtr), Type.EmptyTypes, typeof(DynamicHelper), true);

            var gen = dm.GetILGenerator();
            gen.Emit(OpCodes.Ldftn, info);
            gen.Emit(OpCodes.Ret);

            _cache[info] = (Func<IntPtr>)dm.CreateDelegate(typeof(Func<IntPtr>));

            return _cache[info];
        }
    }
}
```

### Checking a single method

With the information we got we can now check if a given method is altered by the MonoMod framework.

To do so we simply need to copy the first bytes of the given method and check them against the used `JMP` assembler opcodes.

This is pretty straightforward so i only leave the code here for you.

> This will also return true when the app is started within visual studio!
{: .notice--danger}

```cs
public static bool IsPatched<T>(T target) where T : Delegate
{
    var address = GetMethodStart(target);

    // We only need the first 5 bytes to perform our check
    var buffer = new byte[5];

    Marshal.Copy(address, buffer, 0, buffer.Length);

    // Those 3 cases represent the different opcodes used by MonoMod to place a jump
    return buffer[0] == 0xE9
        || (buffer[0] == 0x68 && buffer[4] == 0xC3)
        || (buffer[0] == 0xFF && buffer[1] == 0x25);
}
```

---

### Detecting strings

Another simple approach to detect the usage of MonoMod or Harmony is to detect text within the loaded modules in our application.

I came up with the following lists of strings which were the most obvious candidates for me.

#### Module names

- 0Harmony
- HarmonySharedState
- MonoMod.Utils.Cil.ILGeneratorProxy
- MonoMod.RuntimeDetour

#### Namespaces

- HarmonyLib
- MonoMod

#### Types

- MethodPatcher
- NativeDetourData
- ILGeneratorProxy

They can be detected by using the `Assembly` type and the `Reflection` namespace but that is up to you.

#### Environment Variables

MonoMod can be configured globally on any system using `Environment Variables`.

The presence may not be enough to be sure it is actually used within the application.

If you are paranoid you may want to search all variables for string starting with `MONOMOD`.

Since retrieving all environment variables at once isn't quite easy for most people i'll leave it here for you to use!

[EnvironmentEx.cs]("https://gist.github.com/michel-pi/af478c482404b1ae45ab275a238582ca")

---

### Conclusion

It doesn't matter if it's about protecting critical parts of your own application or analysing another assembly.
Knowing about this technique and frameworks will not only help you but also save your valuable time.

Make sure to remember what you have learned today on your next project.
