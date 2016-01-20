Known differences when migrating from Windows Store apps (aka Desktop) to UWP (aka NetCoreForCoreCLR)
===========

Using desktop assemblies in CoreCLR apps is not supported
-----------
Trying to load assemblies that have references to desktop framework assemblies (ex: System, 4.0.0.0, b77a5c561934e089) is not supported.

You might get warnings if you're trying to use assemblies that are not compiled against the .NET Core surface area.

{@TODO add sample warning}

ArgumentException instead of TargetException
-----------
MethodInfo.Invoke and FieldInfo.Get/SetValue() on the desktop throws TargetException when the "this" object is null or of the wrong type. 
The TargetException class does not exist in the Win8P surface area, thus ArgumentException will be thrown instead. ArgumentException was chosen because that is what Delegate.DynamicInvoke() throws for the analogous violation, making it possible for us to implement this efficiently.

Mixed mode assemblies are not supported
-----------
Only pure IL assemblies are supported.

{@TODO glob/loc issues from the doc}

AssemblyName parsing and generation are different
----------
AssemblyName.FullName behaves differently between CoreCLR and Desktop.

* On CoreCLR: `MyAssemblyName, Culture=neutral, PublicKeyToken=null`
* On Desktop: `MyAssemblyName`

Delegate.BeginInvoke/EndInvoke is not supported 
---------
This is not exposed from the .NET Core API surface.

The [Serializable] attribute is not supported
---------
This, and ISerializable is not exposed from the .NET Core API surface.

Interop scenarios on CoreCLR
---------
Supported interop scenarios are limited to those required for P/Invokes and WinRT support. However, since WinRT is built on classic COM, many classic COM scenarios are supported too. In particular, the following are not supported:

1. Use of IDispatch
2. VARIANT support

TLB consumption - Often TLBs will work fine if IDispatch/VARIANT isn't used. However, the entire scenario isn't completely supported

{@TODO Networking}

{@TODO WCF}

Obfuscated apps are not supported
------------
Obfuscators often rely on undocumented behaviors of the runtime. .NET Native doesn't attemt to replicate undocumented behaviors of the CLR.

Runtime exceptions vs compiler time errors
------------
Some of the runtime exception can now be a .NET Native compiler error. This is because the code gets compiled ahead of time. For example, TypeLoad exception or JIT throw.

Framework exception might not contain messages and parameter names
------------
In order to save space for Release builds we remove the parameter names from public Framework exception and even the exception message itself might be stripped out for public framework exceptions. Compiling with .NET Native under Debug configuration will contain full exception messages.

Modules are not supported
------------
Modules are deprecated in ProjectN and our metadata format has no information about modules. TypeInfo.Module and Assembly.ManifestModule APIs return <Unknown>. Retrieving module level custom attributes will return an empty collection.

Task.FromAsync overloads not particularly efficient 
------------
They’re not particularly *efficient* - each call will end up consuming a whole ThreadPool thread – but they are functional.

F# is not currently supported
------------
Support for F# requires features that are not yet implemented in the .NET Native compiler.

Types may have different private implementation details
------------
In some cases, .NET Native provides different implementations of .NET Framework class libraries. An object returned from a method will always implement the members of the returned type. However, since its backing implementation is different, you may not be able to cast it to the same set of types as you could on other .NET Framework platforms.

1. For example, in some cases, you may not be able to cast the IEnumerable<T> interface object returned by methods such as TypeInfo.DeclaredMembers or TypeInfo.DeclaredProperties to T[].
2. To work around this problem, make sure to reference contracts instead of implementation assemblies (for example: System.Runtime instead of mscorlib). Binding against contracts will ensure that the type will always be resolved.

Reference equality for Type objects
-----------
Project N allows the use of reference equality on Type objects to detect semantic equality, but not on other Reflection objects (such as TypeInfo and MemberInfo) 
Note that this was never a guaranteed semantic even on the desktop but it worked often enough that people took dependencies on it anyway. (Reflection/Type System)

* To compare reflection objects such as MemberInfo objects for semantic equality, use the Equals() override, not “==”.
* Using “==” may work accidentally and sporadically due the framework caching and reusing Reflection object instances.
* However, this is not guaranteed to work 100%. Furthermore, .NET Native’s Reflection implementation was designed with an increased emphasis on memory utilization and is less inclined to reuse object instances than the full CLR implementation. 

Because of this, the “==” error may more likely to manifest itself in the .Net Native implementation.

EqualityComparer&lt;T&gt;.Default doesn't use the IEquatable&lt;T&gt; interface even if T implements it
----------
This is a deliberate tradeoff of strict compatibility vs. performance. Unlike the desktop (which uses MakeGenericType() to turn the IEquatable&lt;T&gt; cast check to a one-time cost), we would have to do the IEquatable&lt;T&gt; cast on each call to Equals(). This would aggravate what's already a known sore spot when it comes to .NET Native performance vs. desktop performance.

EqualityComparer&lt;T&gt;.Default will use the Object.Equals and Object.GetHashCode virtual methods.

In order to avoid compatibility problems, make sure your IEquatable&lt;T&gt;.Equals and IEquatable&lt;T&gt;.GetHashCode implementations match the Object.Equals and Object.GetHashCode behavior.

See "Notes to implementers" section here: https://msdn.microsoft.com/en-us/library/ms131190(v=vs.110).aspx

