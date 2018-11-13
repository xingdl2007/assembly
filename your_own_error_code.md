# Your own error code

Andrzej https://akrzemi1.wordpress.com/2017/07/12/your-own-error-code/

I was recently implementing the “classification of error conditions” in my application offered by the functionality behind *std::error_code*. In this post I want to share some of my experience and insight.

C++11 comes with a quite sophisticated mechanism for classifying error conditions. You may have encountered names like “error code”, “error condition”, "error category”, but figuring out what good they are, and how to use them is difficult. The only valuable source of information on the subject in the Internet is a series of blog posts by Christopher Kohlhoff, the author of Boost.Asio library:

- “[System error support in C++0x – part 1](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-1.html)”
- “[System error support in C++0x – part 2](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-2.html)”
- “[System error support in C++0x – part 3](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-3.html)”
- “[System error support in C++0x – part 4](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-4.html)”
- “[System error support in C++0x – part 5](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-5.html)”

And this was a really good start for me. But still, I believe it would be beneficial to have more than one source of information, and more than one way of explaining the subject. So here we go… 

The problem

First, why I need it. I have a service for looking for flight connections. You tell me where from and where to you want to go, and I will offer you concrete flights, and a price. In order to be able to do this, my service calls other services in turn:

- one for finding the (short) sequence of flights that will take you to your destination,
- one for checking if there is still seats available on these flights in the requested class of service (economy class, business class)

Each of these services can fail for a number of reasons. Reasons for failure — different for each service — can be enumerated. For instance the authors of these two services chose the following enumerations:

    enum class FlightsErrc
    {
        // no 0
        NonexistentLocations = 10, // requested airport doesn't exist
        DatesInThePast,            // booking flight for yesterday
        InvertedDates,             // returning before departure
        NoFlightsFound       = 20, // did not find any combination
        ProtocolViolation    = 30, // e.g., bad XML
        ConnectionError,           // could not connect to server
        ResourceError,             // service run short of resources
        Timeout,                   // did not respond in time
    };

    enum class SeatsErrc
    {
        // no 0
        InvalidRequest = 1,    // e.g., bad XML
        CouldNotConnect,       // could not connect to server
        InternalError,         // service run short of resources
        NoResponse,            // did not respond in time
        NonexistentClass,      // requested class does not exist
        NoSeatAvailable,       // all seats booked
    };

Some things to observe here. First, the reasons for failure are quite similar in either service, but they are assigned different names and different numeric values. This is because two services are developed independently by two different teams. This also means that the same numeric value can refer to two completely different conditions depending on which service reported it.

Second, as can be seen from the names, the causes for failure come from different sources:

- environment: internal problem in the service (e.g., with resources),
- miscommunication: between two services,
- user: providing incorrect data in request,
- just bad luck: no error actually, but no response can be returned to the user because e.g., all seats have been sold out.

Now, what do I really need those different error codes for? If any of these errors occurs we want to stop processing the current request from the user. When I can offer no flight connection to him, I only want to distinguish the following situations:

1. you gave us illogical request,
2. no airline we are aware of is able to offer you a trip,
3. there is some problem with the system which you will not understand but which prevents us from giving you the requested answer.

On the other hand, for the purpose of internal audit, or looking for bugs in the logs, we want a more detailed information to be put into logs, like which system reported the failure, and what actually happened. This can be encoded in an integer number. Any more details, like ports on which we tried to connect, or what database we tried to connect, are likely to be logged separately, so the data encoded in ints should be sufficient.

## The std::error_code

The Standard Library *std::error_code* is designed to hold exactly this type of information: a number representing the status, and a “domain” within which this number is assigned meaning. In other words, an *std::error_code* is a pair: *{int, domain}*. This is reflected in its interface:

    void inspect(std::error_code ec)
    {
        ec.value();    // the int value
        ec.category(); // the domain
    }

But you almost never want to inspect an error_code this way. As we already said, the two things we want to do is to log the state of the error_code as it was constructed (without being later transformed by higher layers of the application), and to use it to answer a specific question, like “is this error caused by the user providing data he knew was incorrect”.

In case you ask yourself, why use std::error_code instead of exceptions, let me clarify: the two things are not mutually exclusive. I want to report failures in my program through exceptions. It is that inside the exception, rather than storing and parsing strings, I just want to contain an error_code that I can easily inspect. std::error_code has nothing to do with avoiding exceptions. Also, in my use case I do not feel a compelling reason to have many different types of exceptions. I just need one: I will catch them only in one (or two) places and I will tell the different situations by inspecting the error_code object.

## Plugging your enumeration

Now, we want to adapt the std::error_code so that it can store error situations from the Flights service described above:

    enum class FlightsErrc
    {
        // no 0
        NonexistentLocations = 10, // requested airport doesn't exist
        DatesInThePast,            // booking flight for yesterday
        InvertedDates,             // returning before departure
        NoFlightsFound       = 20, // did not find any combination
        ProtocolViolation    = 30, // e.g., bad XML
        ConnectionError,           // could not connect to server
        ResourceError,             // service run short of resources
        Timeout,                   // did not respond in time
    };

We should be able to convert from our enum to std::error_code:

    std::error_code ec = FlightsErrc::NonexistentLocations;

But our enumeration must meet one condition: numeric value 0 must not represent an error situation. 0 represents a success in any error domain (category). This expectation is later exploited when we are inspecting an std::error_code object:

    void inspect(std::error_code ec)
    {
        if (ec) // equivalent to: ec.value() != 0
            handle_failure(ec);
        else
            handle_success();
    }

In this sense the [mentioned blog post](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-4.html) has it incorrect that numeric value 200 indicates success.

So this is what we did: we did not start the enumeration of FlightsErrc with 0. This in turn implies that we can create the enumeration that does not correspond to any of the enumerated values:

    FlightsErrc fe {};

This is an important characteristic of enums in C++ (even the C++11 enum classes): you can create values from outside the enumerated range. It is for this reason that compilers issue a warning in switch-statement that “not all control paths return value” even though you have a case label for every enumeration.

Now, back to the conversion, std::error_code has a converting constructor template that looks more-less like this:

    template <class Errc>
        requires is_error_code<Errc>::value
    error_code(Errc e) noexcept
        : error_code{make_error_code(e)}
        {}

(Of course, I used yet non-existent concepts syntax, but you get the idea: this constructor is only visible when std::is_error_code<Errc>::value evaluates to true.)

This constructor is meant to be a customization hook for plugging custom error enumerations into the system. In order to plug FlightsErrc, we have to make sure that:

1. *std::is_error_code<Errc>::value* returns true,
2. Function *make_error_code* taking FlightsErrc is defined and accessible through argument-dependent lookup.

Regarding the first part, we need to specialize the standard type trait:

    namespace std
    {
        template <>
        struct is_error_code_enum<FlightsErrc> : true_type {};
    }

This is one of these situations where declaring something in namespace *std* is legal.

Regarding the second part, we just need to declare function overload make_error_code in the same namespace as enum FlightsErrc:

    enum class FlightsErrc;
    std::error_code make_error_code(FlightsErrc);

And this is all that other parts of the program/library need to see, and what we have to provide in the header file. The rest is the implementation of function make_error_code and we can put it in a separate translation unit (a .cpp file).

With this in place, we can make an impression that FlightsErrc is an error_code:

    std::error_code ec = FlightsErrc::NoFlightsFound;
    assert (ec == FlightsErrc::NoFlightsFound);
    assert (ec != FlightsErrc::InvertedDates);

## Defining error category

So far, I have only been saying that error_code is a pair: {number, domain}, where the first element uniquely identifies the particular error situation within the domain, and the second uniquely identifies the domain of errors across all possible error domains that will ever be conceived. But given that this domain ID should be stored in one machine word, how can we guarantee that it will be unique across all libraries currently in the market and those yet to come? We are hiding the domain ID as an implementation detail. If we are to use another third-party library with its own error enumeration, how can we guarantee that their domain ID will not be equal to ours?

The solution chosen for *std::error_code* relies on the observation that for every global object (or more formally: namespace-scope object) a unique address is assigned. No matter how many libraries are combined together, with how many globals, each global has a unique address — this is quite obvious.

In order to exploit this, we have to associate with every type that wants to be plugged into the error_code system a unique global object, and then use its address as an ID. Now, this implies using pointers for representing domains, and this is indeed what *std::error_code* is doing. But now, that we store some T* the question is what the T should be. The choice is quite clever: let’s use a type that can offer us additional benefits. The type T used is *std::error_category*, and the additional benefit is in its interface:

    class error_category {
    public:
        virtual const char* name() const noexcept = 0;
        virtual string message(int ev) const = 0;
        // other members ...
    };

I used name “domain”, the Standard Library uses name “error category” for the same purpose.

It has pure virtual member functions, which already suggests something: we will be storing pointers to classes derived from std::error_category: each new error enum requires a new corresponding class to be derived from std::error_category. Usually having pure virtual functions implies allocating objects on the heap, but we will do no such things. We will be creating global objects and pointing to them.

There are more virtual member functions in std::error_category that on other occasions need to be customized, but we will not have to do it for the purpose of plugging our FlightsErrc.

Now, for each custom error “domain” represented by a class derived from std::error_category, we need to override two member functions. Function *name* is expected to return a short mnemonic-like descriptive name of this error category (domain). Function *message* assigns a text description for every numeric error value in this domain. In order to illustrate it better, let’s define an error category for our enum FlightsErrc. Remember, this class only needs to be visible in one translation unit. In other files we will just be using an address to its instance.

    namespace { // anonymous namespace
    
    struct FlightsErrCategory : std::error_category
    {
        const char* name() const noexcept override;
        std::string message(int ev) const override;
    };
    
    const char* FlightsErrCategory::name() const noexcept
    {
        return "flights";
    }
    
    std::string FlightsErrCategory::message(int ev) const
    {
        switch (static_cast<FlightsErrc>(ev))
        {
        case FlightsErrc::NonexistentLocations:
            return "nonexistent airport name in request";
        
        case FlightsErrc::DatesInThePast:
            return "request for a date from the past";
        
        case FlightsErrc::InvertedDates:
            return "requested flight return date before departure date";
        
        case FlightsErrc::NoFlightsFound:
            return "no filight combination found";
        
        case FlightsErrc::ProtocolViolation:
            return "received malformed request";
        
        case FlightsErrc::ConnectionError:
            return "could not connect to server";
        
        case FlightsErrc::ResourceError:
            return "insufficient resources";
        
        case FlightsErrc::Timeout:
            return "processing timed out";
        
        default:
            return "(unrecognized error)";
        }
    }
    
    const FlightsErrCategory theFlightsErrCategory {};
    
    }

Function name provides a short text that is used while streaming out an std::error_code into things like logs: it can help you identify the cause of an error. It does not have to be unique across all different error enums: in the worst case log entries will be ambiguous.

Function message provides a descriptive message for any numeric value representing an error in our category. This can be helpful in debugging or browsing logs; but you would probably not want to give this text to the users unprocessed. These messages are close to the comments I initially put in the definition of FlightsErrc.

This function is usually called indirectly. The callers cannot know that the numeric error value is a FlightsErrc, so we have to explicitly cast it back to FlightsErrc. I believe the example in the [aforementioned article](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-4.html) does not compile due to the omitted static_cast. Now after the cast, there is a risk that we will be inspecting a value from outside the enumerated set: therefore we need the default label. (Interestingly, whenever I decide to use enum class in my programs, I immediately find myself in need of static_casting it either to or from int.)

Finally, note that we have initialized a global object of our type FlightsErrCategory. This will be the only object of this type in the program. We will need its address (to tell error_codes from different domains), but also we will use its polymorphic properties.

Although class std::error_category is not a literal type, it has a constexpr default constructor. The implicitly declared default constructor of our class FlightsErrCategory inherits this constexpr-ness. Thus, we are guaranteed that the initialization of our global object is performed during constant initialization, as described in this [post](https://akrzemi1.wordpress.com/2012/05/27/constant-initialization/), and is therefore free from any [static initialization order fiasco](https://isocpp.org/wiki/faq/ctors#static-init-order).

Now, the last missing part is implementing *make_error_code*:

    std::error_code make_error_code(FlightsErrc e)
    {
        return {static_cast<int>(e), theFlightsErrCategory};
    }

And we are done. Our FlightsErrc can be used as if it was an std::error_code:

    int main()
    {
        std::error_code ec = FlightsErrc::NoFlightsFound;
        std::cout << ec << std::endl;
    }

The following program will output:

    flights:20

A full working example illustrating all the above can be found [here](https://akrzemi1.wordpress.com/examples/error_code-example/).

And that’s it for today. What we have still not covered is how to make useful queries on std::error_code objects, but this will be the subject of another post.

## Acknowledgements

I am grateful to Tomasz Kamiński for explaining to me the idea behind std::error_code. Apart form Christopher Kohlhoff’s series of posts, I was also able to learn about std::error_code form the documentation of Niall Douglas’s [Outcome library](https://github.com/ned14/outcome), [here](https://ned14.github.io/outcome/tutorial/error_code/).