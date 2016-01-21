Known differences when migrating to .NET Native
===========

### Using libraries not targeted to .NET Core profile is unsupported
Trying to load assemblies that have references to Full .NET Framework framework assemblies (ex: System, 4.0.0.0, b77a5c561934e089) or APIs that only exist in the Full .NET Framework is not supported. Doing so will cause the .NET Native compiler to emit warnings.

{@TODO add sample warning}

### ArgumentException instead of TargetException
MethodInfo.Invoke and FieldInfo.Get/SetValue() on the Full .NET Framework throws TargetException when the "this" object is null or of the wrong type. 
The TargetException class does not exist in the Win8P surface area, thus ArgumentException will be thrown instead. ArgumentException was chosen because that is what Delegate.DynamicInvoke() throws for the analogous violation, making it possible for us to implement this efficiently.

### Mixed mode assemblies are not supported
Only pure IL assemblies are supported.

### AssemblyName parsing and generation are different
AssemblyName.FullName behaves differently

* On CoreCLR: `MyAssemblyName, Culture=neutral, PublicKeyToken=null`
* On Full .NET Framework: `MyAssemblyName`

{@TODO is this still true?}

### Delegate.BeginInvoke/EndInvoke is not supported 
This is not exposed from the .NET Core API surface.

### The [Serializable] attribute is not supported
This, and ISerializable is not exposed from the .NET Core API surface.

### Interop scenarios on CoreCLR
Supported interop scenarios are limited to those required for P/Invokes and WinRT support. However, since WinRT is built on classic COM, many classic COM scenarios are supported too. In particular, the following are not supported:

* Use of IDispatch
* VARIANT support

TLB consumption - Often TLBs will work fine if IDispatch/VARIANT isn't used. However, the entire scenario isn't completely supported

### Obfuscation is not supported
Obfuscators often rely on undocumented behavior of the runtime. .NET Native doesn't attempt to replicate undocumented behaviors of the CLR.

### Runtime exceptions vs compiler time errors
Some of the runtime exception can now be a .NET Native compiler error. This is because the code gets compiled ahead of time. For example, TypeLoad exception or JIT throw.

### Framework exception might not contain messages and parameter names
In order to save space for Release builds we remove the parameter names from public Framework exception and even the exception message itself might be stripped out for public framework exceptions. Compiling with .NET Native under Debug configuration will contain full exception messages.

### Modules are not supported
Modules are deprecated in .NET Native. TypeInfo.Module and Assembly.ManifestModule APIs return <Unknown>. Retrieving module level custom attributes will return an empty collection.

### Task.FromAsync overloads not particularly efficient 
They’re not particularly *efficient* - each call will end up consuming a whole ThreadPool thread – but they are functional.

### F# is not currently supported
Support for F# requires features that are not yet implemented in the .NET Native compiler.

### Types may have different private implementation details
In some cases, .NET Native provides different implementations of .NET Framework class libraries. An object returned from a method will always implement the members of the returned type. However, since its backing implementation is different, you may not be able to cast it to the same set of types as you could on other .NET Framework platforms.

1. For example, in some cases, you may not be able to cast the IEnumerable<T> interface object returned by methods such as TypeInfo.DeclaredMembers or TypeInfo.DeclaredProperties to T[].
2. To work around this problem, make sure to reference contracts instead of implementation assemblies (for example: System.Runtime instead of mscorlib). Binding against contracts will ensure that the type will always be resolved.

### Reference equality for Type objects
.Net Native allows the use of reference equality on Type objects to detect semantic equality, but not on other Reflection objects (such as TypeInfo and MemberInfo) 
Note that this was never a guaranteed semantic even on the Full .NET Framework but it worked often enough that people took dependencies on it anyway. (Reflection/Type System)

* To compare reflection objects such as MemberInfo objects for semantic equality, use the Equals() override, not “==”.
* Using “==” may work accidentally and sporadically due the framework caching and reusing Reflection object instances.
* However, this is not guaranteed to work 100%. Further, .NET Native’s Reflection implementation was designed with an increased emphasis on memory utilization and is less inclined to reuse object instances than the full CLR implementation. 

Because of this, the “==” error may more likely to manifest itself in the .Net Native implementation.

### EqualityComparer&lt;T&gt;.Default doesn't use the IEquatable&lt;T&gt; interface even if T implements it
This is a deliberate trade-off of strict compatibility vs. performance. Unlike the Full .NET Framework (which uses MakeGenericType() to turn the IEquatable&lt;T&gt; cast check to a one-time cost), we would have to do the IEquatable&lt;T&gt; cast on each call to Equals(). This would aggravate what's already a known sore spot when it comes to .NET Native performance vs. Full .NET Framework performance.

EqualityComparer&lt;T&gt;.Default will use the Object.Equals and Object.GetHashCode virtual methods.

In order to avoid compatibility problems, make sure your IEquatable&lt;T&gt;.Equals and IEquatable&lt;T&gt;.GetHashCode implementations match the Object.Equals and Object.GetHashCode behavior.

See "Notes to implementers" section here: https://msdn.microsoft.com/en-us/library/ms131190(v=vs.110).aspx

### Arrays
* Multi-dimensional arrays will not be supported for V1 beyond four dimensions.
* Arrays with non-zero lower bounds are not supported (Array.CreateInstance(type, int,int)). 
* Arrays of unmanaged pointers are not supported (though arrays of IntPtr will work just fine).
* Array.Copy does not do type conversions (copying byte[] into an int[] will throw an exception))

### Generics
Reflection will no longer unify generic types closed over its own formal parameters with the generic type definition.

{@TODO: You can't use reflection to get or set a pointer field. ?}

### Types can be loaded even if there is a cycle in generic initialization 
On Full .NET Framework the following code will throw TypeLoadException at runtime. This code will correctly compile with .NET Native.
```csharp
struct N<T> {} 
struct X { N<X> x;}
```

### Constraint violations don't yield errors in the compiler
Some invalid IL that fails to compile on Full .NET Framework might not result in compiler errors in .NET Native. To save validation time, the compiler assumes the IL was produced by a ECMA-335 compliant compiler.

### System.Type does not inherit from MemberInfo 
Some existing libraries that were not compiled against the .NET Core profile might attempt to cast System.Type to MemberInfo, or attempt to access MemberInfo members.

### Value types cannot declare a default constructor 
It's not possible to create such types in C# or VB, but the IL format supports this. The .NET Native compiler will fail to compile such types. 

### BigInteger
* BigInteger doesn't implement CompareTo(System.Object) implicitly.
* BigInteger.ToString ("E") is correctly rounded in .NET Native. In some versions of the CLR, the result string is truncated instead of rounded.

### No support for C++-style varargs
The undocumented keyword __arglist is not supported in C#.

### P/Invoke methods cannot be reflection invoked
To work around the problem, you can introduce a managed method to wrap the call to the P/invoke and reflection-invoke the wrapper.

### Application package file layout has changed
The layout of binaries inside of your .appx is significantly different when using .NET Native vs. previous versions of .NET. The most obvious one is that all of the managed code in an application will be rolled into a single large .dll. As a result, enumerating and loading files in a .appx behaves differently. Code that relies on seeing the managed *.dll files in the application package will not see them.

### Inconsistencies for GetRuntime*() methods
{@TODO make a bulleted list of the APIs}
On the Full .NET Framework, GetRuntimeProperties() and GetRuntimeEvents() behave inconsistently from the other GetRuntime*() APIs. Unlike the other GetRuntime*() APIs, they suppress hidden instance properties and events defined inside base classes.

On .NET Native, the GetRuntime*() APIs all behave consistently. That is, the results include hidden instance members from base classes.

### RuntimeReflectionExtensions.GetRuntimeInterfaceMap is not supported
This is not currently supported.

### RuntimeReflectionsExtensions.GetRuntimeMethods() doesn't include hidden members in base classes
On the Full .NET Framework, RuntimeReflectionsExtensions.GetRuntimeMethods()'s output include hidden members in base classes, but GetRuntimeProperties() and GetRuntimeEvents() does not.

### NullableComparer not supported
System.Collections.Generics.Comparer.Default&lt;Nullable&lt;T&gt;&gt; will produce a usable comparer only if T implements the non-generic IComparable interface.

### System.Reflection.Context not supported
This is not currently supported.

### ArgumentException instead of TargetParameterCountException
Delegate.DynamicInvoke throws ArgumentException rather than TargetParameterCountException if both the argument count and parameter types are wrong.

This also affects MethodBase.Invoke, PropertyInfo.Get/Set and any other APIs that are based on dynamic invocation.

### XmlDocument line endings now conform with standard
XmlDocument replaces `\r\n` -> `\n` and `\r` -> `\n` where it doesn’t do that on Full .NET Framework (even when you tell it to preserve white spaces). The new behavior conforms the XML standard.

### Aggressive late global optimizations may remove code with side effects
For example:
* An access that would cause a null reference exception may be optimized away if the accessed object is not used afterwards.
* Unused casts won't throw InvalidCastException

### Debugging in Release mode results in a non-optimal developer experience
To debug .NET Native specific issues, enable .NET Native compiler in the Debug build of your application. {@TODO add link with pictures and instructions}

### Infinite loops on any thread may lead to GC starvation
Imagine simple case of having only two threads and one is in infinite thread, other is waiting for GC collection, GC can’t start waiting for loop thread to stop, which will cause the starvation.

### Tail calls are currently not supported
The tail. prefix is just a hint and it won't be respected by the .NET Native compiler.

### Types that have duplicate members are not supported 
Duplicate members are not allowed as per the ECMA 335 specification. .NET Native is more strict about correctness of the inputs.

### Types with 256 or more generic type arguments will be rejected
This may also affect deeply nested generic types where the C# compiler introduces new generic type parameters behind the scenes.

{@TODO Interop}
{@TODO Reflection - RD.XML and stuff}
{@TODO Glob/Loc}
{@TODO Serialization}
{@TODO Networking}
{@TODO WCF}
