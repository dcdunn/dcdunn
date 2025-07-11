# C++ I have done
C++ has been my bread-and-butter for over 20 years. Here are some unorganised notes. 

## Early experience
I started with C++ in the early 2000s, when the STL was young.  I was fortunate
enough to work with some very talented engineers at D-Cubed and DisplayLink, who
understood the principles of good design. I was directed to Scott Meyers and
Herb Sutter to learn how to do it properly. 

One frustration was the lack of language support for concurrency. We built core
libraries of our own, using Win32 and posix primitives to create thread classes
and synchronisation primitives. Similarly, we had *Time*, *Clock* and
*StopWatch* classes, which had to use Win32 or posix primitives under-the-hood.

We _really_ wanted to use Resource Allocation Is Initialisation (RAII) properly,
but the old *auto_ptr* was had design flaws.
 * Transfer of ownership didn't work safely, and could lead to double-deletion.
 * Unable to use in STL containers.
 When C++11 introduce move semantics, we had a very large codebase that was strewn with *auto_ptr*. We had to fix it up, and in fact we had to fix it to move to C++17. 

## C++11
There is almost no part of what came with C++11 that I don't use fully, to the
extent that it's difficult to imagine the language being otherwise.
 * The thread library. No more bespoke threads or synchronisation primitives.
   Promise, future, mutex, condition variable and thread itself was one of the
   most useful improvements.
 * Move semantics: express ownership and transfer of it correctly.
 * *default*, *delete*, *override*, *final*. Make your intentions in class design
   clear and semantic.
 * The *functional* library. Much easier to create bindings and pass function pointers around. 
 * Lambdas. Much easier to create readable calls to STL algorithms; narrow scope
   of function definitions.
 * *chrono* library. It is not always as easy as it promises to be, but in most
   applications I've worked in, it's important to get time units right. 
 * Auto-type deduction. 
 * Brace initialisation, range-base loops, nullptr, 


## Coding standards
It was [The Elements of C++ Style](https://www.amazon.co.uk/Elements-Style-SIGS-Reference-Library/dp/0521893089) back then. These days the [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/) are better. 

## RAII and C libraries.
Most C++ programmers will use smart-pointers for managing memory. The idiom is
also very useful for interfacing with C libraries. For example to create an HTTP
request using libcurl requires something like this (from [here](https://curl.se/libcurl/c/curl_easy_init.html)):
```C
int main(void)
{
  CURL *curl = curl_easy_init();
  if(curl) {
    CURLcode res;
    curl_easy_setopt(curl, CURLOPT_URL, "https://example.com");
    res = curl_easy_perform(curl);
    curl_easy_cleanup(curl);
  }
}
```

In C++, it is safer to manage the *CURL* resource using a smart pointer:

```C++
class HttpRequest
{
private:
  using CurlPtr = std::unique_ptr<CURL, std::function<void(CURL*)>>;
  CurlPtr m_curl;

public
  HttpRequest(const std::string& url, ...)
  : m_curl(curl_easy_init(), [](CURL* c) { curl_easy_cleanup(c); })
  {
    if (!m_curl.get()) {
        // throw
    }
  }

  CURLcode send() { return curl_easy_perform(m_curl.get()); }
}

```

Not only does this make for safer resource management, but it helps  when
building abstractions. In the case of curl, you build the header list using a
similar resource acquisition, and you can equally well build an abstraction of
the headers. 

_Nota Bene_ thinking about security would tell you not use a *std::string* to
represent a URL, but instead create a URL class that throws in its constructor
if you try to create an invalid one. 

## Functional styles
C++11 gave rise to the possibility of using some paradigms from functional
programming. For example, we can pass around behaviour, using the *functional*
library. The following uses this approach for an asynchronous HTTP request, and
frees us from creating a new object-oriented inteface. 

```C++
class SendHttpRequest
{
public:
  using OnRequestComplete = std::function<void(const HttpResponse&)>

  virtual ~HttpService() = default;
  // Send an HTTP request asynchronously
  virtual void get(const Url&, OnRequestComplete);
};

class Weather
{
  SendHttpRequest& m_request;

  void onNewReport(const HttpResponse& response) 
  {
    // parse and inform observers
  }

public:
  Weather(SendHttpRequest& request)
  : m_request(request)
  {}
   
  WeatherReport latest()
  {
    m_request.get(URL, [this](const HttpResponse& response) {
      this->onNewReport(response);
    })
  }
}

```


## Interface segregation
Multiple inheritance is usually considered a bad idea. I think that this is true if you inherit concrete classes (which I argue you should not do very often). However, it is often the case that a concrete class might implement several interfaces. For example:

```C++

class DataPersist
{
public:
  virtual void save(const Type&) = 0;
};

class DataRetrieve
{
public:
  virtual Type retrieve() = 0;
};

class DatabaseAccess: public DataPersist, public DataRetrieve
{

};

```
Then you can pass a pointer or reference to the concrete type as a narrower
interface to parts of you system that need to save or retrieve data.


A technique that I like to use is to make the concrete implementation of abstract
functions *private*:

```C++

class MyInterface
{
public:
  virtual MyInterface() = default;
  virtual void method() const = 0;
};

class MyImplementation: public MyInterface
{
private:
  void method() const override;
};
```
This sometimes surprises engineers, but not only is it possible, it is
semantically correct. We will be unable to access the method except through a
pointer or reference to type *MyInterface*, which supports the paradigm of coding to interfaces. Apart from some creation method that needs to know the concrete types, we parameterise the system with a reference or pointer to the interface.

```C++

void f(const MyInterface& if)
{
  // acceptable
  if.method();
}


MyImplementation obj;
// This is not allowed:
obj.method();
```

There is one downside, which is unit testing. Here we need to create and test the concrete-type. The extra cost of either creating the subject-under-test on the heap, or some other way of indirecting in test, is worth bearing for safer production code.


## Auto type detection
A lot of C++ developers are uneasy about auto type detection, and usually say
they need to know the type to read code accurately. I'm not sure that this is
generally true, and there does not seem to be the same wariness of *using* or
*typedef* to make code more readable by introducing semantic naming.


The [C++ Guideline ES.11](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Res-auto) says _use auto to avoid redundant repetition of type names_. I think this is a strong guideline. For example, 

The compiler already knows the type - why go on a merry dance to work it out again? 