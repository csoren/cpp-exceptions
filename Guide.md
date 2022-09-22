# Introduction

This document is the author's personal view and rules of thumb for using C++ exceptions effectively. There may be times when deviating from these is beneficial, that is up to the reader.

# 1. Throwing exceptions
The most commonly used C++ compilers today all implement zero-cost exceptions - exceptions are free (in execution time) when they are not thrown. When they are thrown, they are potentially more expensive than returning an error code. Exceptions should thus only be thrown in exceptional circumstances - when something unexpected happens.

Great opportunities for exceptions include invalid tokens when parsing, checksum errors, servers not responding, insufficient access privileges - all examples of a rare and unexpected code path that deviate from the happy path.

Exceptions should never be used to indicate an expected state. An stream representing a file on disk is expected to end at some point. Do not use an exception to indicate that the stream has ended, but do throw an exception if an attempt to actually read past the end was performed by the user. That's a programmer error and should be flagged as such.

Never use exceptions to carry data or information down the call stack, other than information describing the exceptional event.

## 1.1. Use the STL exception classes
C++ exceptions give you, like most of the language's features, plenty of rope with which to shoot yourself in the foot. Don't give into the temptation of throwing a second rate `char *` or `int`, you can do much better. The STL defines a number of [standard exception classes](https://cplusplus.com/reference/stdexcept/) that can be used to indicate runtime errors and logic errors. Use these whenever possible, a `char *` or `int` is not distinguishable as a class of error and makes it impossible for the user to choose an appropriate course of action without assigning a meaning to the value.

Don't:

```
void MyClass::ToString() const {
  // Don't throw char*
  throw "Not implemented";
}
```

Do:
```
void MyClass::ToString() const {
  // Throw a proper exception
  throw std::runtime_error("Not implemented");
}
```

## 1.1.2. Derive from the standard exception classes whenever possible

Derive from the STL hierarchy of exception classes when you want your exception to be particularly distinguishable in a catch block. Do *not* invent your own exception base class.

## 1.1.3. Use built-in data types exclusively when designing exception classes

When you design your own exception classes, member variables should be PODs (plain old datatypes such as `int` or `char *`) or classes behaving as PODs. Do not aggregate expensive objects such as `std::string` or STL containers in your exception classes. These require initialization and heap allocations, which might throw exceptions.

## 1.1.4. Don't throw exceptions when constructing or destroying exception objects
Exception class constructors *should* be strictly `noexcept`, exception class destructors must *never* throw an exception. Throwing an exception while `delete`'ing an exception object as part of a stack unwind will cause an unrecoverable error and a call to `std::terminate`.

There are legitimate uses of throwing an exception in constructors, for instance when a memory allocation fails. There is no other way to communicate this error.

STL exceptions' constructors, destructors and assignment operators are marked `noexcept`, try to follow this pattern.

## 1.1.5. Exception classes should be immutable
While there's nothing technically wrong with modifying an exception and rethrowing it, constructing a new one will take up fewer lines of code and be more readable. There’s no good reason for a exception handler to be able to change the exception after it’s been thrown. Throw a new exception (but take special care not to mask the source of the problem.)

Don't:
```
catch (MutableException& ex) {
  // Don't allow this
  ex.SetMessage("Now for something completely different");
  throw;
}
```

## 1.2. Be descriptive
Resist the temptation to provide the empty string as an exception description. After all, if you went through all the trouble of detecting an anomalous situation, at least describe what went wrong. Not only are you helping the user of your class, but the code also becomes self-documenting.

Don't:
```
void MyFunc(int n) {
  // Silly bit trick
  if (n & -n != n)
    throw std::logic_error("");
  // ...
}
```

Do:
```
void MyFunc(int n) {
  // Silly bit trick
  if (n & -n != n)
    throw std::invalid_argument("Parameter n must be a power of two");
  // ...
}
```

## 1.3. Throw by value
Always throw an exception by value. Never throw a naked pointer. Instantiating an exception and throwing a pointer to it requires whoever catches it to dispose of it (also, instantiating the exception might throw an `std::bad_alloc` exception, which is not what you wanted to throw at all.) The exception handler will have to remember to dispose of the exception and even guess how to do this properly. Their best bet is `delete` but who knows for sure how you instantiated the object. Throwing pointers also don’t mix well with `catch(…)` constructs. There's no way to dispose of the exception as the type is lost.

Leaks are pretty much guaranteed to happen when throwing naked pointers.

As a contrived example, *don't* throw by pointer (and definitely don't use randomly selected allocation methods):

```
try {
  static std::exception ex1("Throwing a pointer to a static exception instance");

  if (GetRandomNumber() & 1)
    throw new std::exception("Throwing a pointer to a new exception");
  else
    throw &ex1;
}
catch (std::exception* ex) {
  // What am I supposed to do now?
}
```

Throwing a pointer to an exception will also be slower than throwing it by value. Instantiating the exception requires allocating memory from the heap and disposing of it again. Throwing a temporary object is subject to the [return value optimization](http://en.wikipedia.org/wiki/Return_value_optimization) and *will* construct the exception directly on the exception frame, no dynamic allocations or copying necessary.

Don't:
```
try {
  // Allocate an exception on the heap and throw a pointer to it
  throw new std::exception("A new exception");
}
catch (std::exception* ex) {
  Log::Info("Exception " + ex->what());
  // Heap deallocation
  delete ex;
}
```

Do:
```
try {
  throw std::exception("Throwing a temporary object, instantiated directly on the exception frame");
}
catch (std::exception& ex) {
  Log::Info("Exception" + ex.what());
}
```

Constructing the exception on the stack and throwing a local variable performs slightly worse than throwing a temporary object, as the copy constructor is possibly invoked to copy the exception onto the stack frame and the destructor called for the temporary object. The standard allows for the move constructor to be used but some compilers seem to use the copy constructor anyway.

```
try {
  std::exception ex("Throwing an exception instantiated on the stack");
  throw ex; // Now copied onto the exception frame
}
catch (std::exception& ex) {
  Log::Info("Exception " + ex.what());
}
```

## 1.4. Throw by reference - does not compute
There's no such thing as throwing by reference. If you throw a reference type, the object will be copied onto the exception frame. You don't have to copy it explicitly or worry if memory has gone away when the exception handler gets hold of the exception.

Don't
```
void MyThrow(const std::exception& ex) {
  throw ex; // ex copied onto exception frame
}
```

Note, however, that you're throwing an `std::exception` - not an instance of ex's dynamic type. Yes, this is indeed the well-known [C++ slicing problem](http://en.wikipedia.org/wiki/Object_slicing). If `ex` is really an `std::runtime_error`, the object will be sliced so only the `std::exception` portion is left, discarding any data or overridden methods in `std::runtime_error` - including `what()`. Be very, very careful when throwing references to exceptions. You rarely get what you want.

## 1.5. Re-throw without arguments
If you have to re-throw a caught exception, `throw` without arguments is almost always what you want to do. If you do supply an argument, the object will be sliced. `throw `without arguments will re-throw the exception polymorphically, reusing the exception frame.

Don't:
```
try {
  throw NetworkDnsException("Unable to look up hostname");
}
catch (std::exception& ex) {
  Log::Info("Saw exception " + ex.what());
  throw ex; // Wrong - will slice, copy and throw an std::exception
}
```

Do:
```
try {
  throw NetworkDnsException("Unable to look up hostname");
}
catch (std::exception& ex) {
  Log::Info("Saw exception " + ex.what());
  throw; // Correct - will re-throw the original exception, no copying or slicing
}
```

# 2. Catching exceptions
In general, exceptions should be caught when you can reasonably recover from the indicated error. If you cannot recover from the exception, don't catch it or attempt to handle it.

Don't:
```
XmlDomDocument XmlDomDocument::LoadFrom(Stream& stream) {
  try {
    while (boost::optional<wchar_t> nextChar = stream.ReadCharacter()) {
      // ...
    }
  }
  catch (NetworkTimeoutException& ex) {
    // The wrong place to handle networking errors.
    // We don't know anything about the network here and have no meaningful
    // way to recover.
  }
}
```

Do:
```
try {
  Stream stream = Http::Get("http://....");
  XmlDomDocument doc = XmlDomDocument::LoadFrom(stream);
  // ...
}
catch (NetworkException& ex) {
  // The right place for handling networking errors.
  // We can retry the connection or report something meaningful to the user.
}
```

## 2.2. Avoid blanket catch clauses
Catching anything (`...`)  or `std::exception` and other base classes should generally be avoided. Rarely does the network stack and GUI need the same recovery code. However, it may be appropriate to catch and handle a generic `NetworkException` from which `NetworkTimeoutException`, `NetworkDnsException` etc. derives.

Don't:
```
XmlDomDocument XmlDomDocument::LoadFrom(Stream& stream) {
  try {
    while (boost::optional<wchar_t> nextChar = stream.ReadCharacter()) {
      // ...
    }
  }
  catch (...) {
    // Now we're just hiding information,
    // giving the user of the class no way to recover.
    return XmlDomDocument::Empty;
  }
}
```

## 2.2.1. Embrace RAII when possible
Blanket catch clauses do have their place. They are, for instance, appropriate in job management classes and other top level code responsible for delegating work. They can also be useful as a "poor man's not-quite-finally" for disposing of unmanaged resources in case of a fatal error.

Handling unmanaged resources:

```
char* aBuffer = new char[64];
try {
  Socket s("localhost", 1919);
  s.ReadBytes(&aBuffer[0], 64);
}
catch (SocketException& e) {
  // ...
}
catch (...) {
  delete[] aBuffer;
  throw;
}
delete[] aBuffer;
```

C++ does not have `finally` blocks, the language doesn't need them. A much better solution is to embrace [the RAII principle](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) and use smart pointers to make sure no resources leak when an exception is thrown.

A better solution:
```
std::unique_ptr<char> aBuffer(new char[64]);
try {
  Socket s("localhost", 1919);
  s.ReadBytes(&aBuffer[0], 64);
}
catch (SocketException& e) {
  // ...
}
```

## 2.3. Use exception translation judiciously

Avoid `catch(...)` if possible and resist masking exceptions as other exceptions - do not throw an `XmlParserException` in a `catch(...)` block. You’re hiding the source of the actual problem and taking full responsibility instead.

Don't:
```
XmlDomDocument XmlDomDocument::LoadFrom(Stream& stream) {
  try {
    while (boost::optional<wchar_t> nextChar = stream.ReadCharacter()) {
      // ...
    }
  }
  catch (...) {
    // At least the user now knows something went wrong, but not what.
    throw XmlParserException("Something went wrong");
  }
}
```

There are legitimate uses for exception translation, however. If you can meaningfully translate an exception into an exception in your domain, by all means do so. For instance in a parser, it *may* be appropriate to translate an `EndOfFileException` into an `ExpectedTokenException` (and maybe include a line number if possible).

Do:
```
try {
  wchar_t ch = stream.ReadNextChar();
  // ...
}
catch (EndOfFileException& e) {
  throw ExpectedTokenException("Expected a valid token but encountered end of file");
}
```

## 2.4. Avoid empty catch bodies

Rarely is "do nothing" an appropriate response to a problem. Avoid empty catch bodies, the next developer will have no chance of understanding why the exception is being swallowed. If no recovery code is appropriate, log the exception and leave a comment explaining why there's no recovery code.

If you find yourself wanting to catch an exception but end up with an empty catch clause, it may be an indicator that you should not be handling the exception here. Choose wisely.

Don't:
```
void ThreadPool::ExecuteTasks() {
  while (boost::optional<Task> task = NextTask()) {
    try {
      task->Run();
    }
    catch (...) {
    }
  }
}
```

Do:
```
void ThreadPool::ExecuteTasks() {
  while (boost::optional<Task> task = NextTask()) {
    try {
      task->Run();
    }
    catch (std::exception& ex) {
      // We can't recover from this but want to keep running our tasks.
      Log::Info("Task exited due to exception " + ex.what());
    }
    catch (...) {
      // We can't recover from this but want to keep running our tasks.
      Log::Info("Task exited due to unknown exception");
    }
  }
}
```

## 2.5. Do not catch by value

*Do not* catch by value. Catching by value copies the exception from the exception frame onto the stack. Even worse - the object is sliced (again!)

Don't:
```
try {
  throw std::runtime_error("Something bad");
}
catch (std::exception ex) {
  // Exception copied and sliced.
  Log::Info("Guess what I'm printing? " + ex.what());
}
```

## 2.6. Catch by reference
Always catch by reference. Catching by reference avoids copying the exception from the exception frame onto the stack. You can catch by const reference if you wish - it’s very nice of you to tell the world about the constness fact, but really - nobody’s listening. In most cases const is just noise as exception classes should be immutable, but add it if that's your style.

Do:
```
try {
  throw std::runtime_error("Something bad");
}
catch (std::exception& ex) {
  // ex is a reference to the exception on the exception frame
  Log::Info("Now we know what we're printing " + ex.what());
}
```

## 2.7. Do not catch by pointer
You’re not throwing anything by pointer so don’t attempt to catch anything by pointer (broken MFC exceptions aside - do not learn from MFC, MFC existed before the language was finalized and MFC is no longer idiomatic C++)
