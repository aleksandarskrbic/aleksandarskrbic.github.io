---
title: "A Taste of Functional Java"
date: 2020-05-24
header:
  image: "/images/java/fp-in-java.jpg"
tags: [java, jvm, fp]
excerpt: "Java, JVM, Functional Programming"
---

{: .text-left}

There are lots of different definitions of what functional programming is. It can be challenging for people new to it to understand it since there are many materials related to it, and you will rarely find one that will present the basics and give you a broader picture. The result of that is that I met many developers who think that Java Stream API is functional programming, or even worse, some of them identify AWS Lambda with functional programming. Yes, Java Stream API has something to do with functional programming, but it is just a tiny part. My goal is to give you a high-level overview of pure functional programming and how it is usually referenced among functional programmers.

If someone asks me should Java be used for functional programming, my answer will be a no. Usually, as a (junior) developer, you don't have a choice of technology, but you can make that Java code better by making it more functional.
Java is not a friendly language for functional programming like Scala or Haskell are, but introducing some functional structures and concepts into your Java code will improve the maintainability, safety, readability, and general quality of your codebase. Even if we are going to use only a few functional concepts in our Java applications, we can’t say that we are writing purely functional code because some rules of functional programming will be violated. First, let's review some core functional programming concepts and see why it can be tough to satisfy all those rules, especially in Java.

## Really short definition of FP

Functional programming is programming with functions. This sounds pretty simple, and you can say that you are already doing it. But there is a trick here, of course. Function in functional programming is treated as the one in mathematics: *binary relation that map arguments to result*.

These functions are often referred to as *pure functions* and cannot perform any side effects. So now, when you know that side effects like mutations, exceptions, and I/O operations are forbidden in a purely functional world, it seems that it's impossible to write purely functional software. Once you start digging deeper into the functional world, you will realize that is possible, but it's just not worth it in Java. So instead of going fully functional, we will explore some functional structures and use them to solve our day-to-day problems.

<figure>
  <img src="{{ site.url }}{{ site.baseurl }}/images/fp/pure-functions.png" class="center-image ">
    <figcaption>Image source: <a href="https://impurepics.com/">impurepics.com</a></figcaption>
</figure>

Now I have to tell you that we will not talk about pure functional programming anymore. The goal of this article is to equip you with a few recipes that you can apply in the real world, even in your next pull/merge request. I plan to walk you through abstracting over control structures, dealing with missing values, and exception handling in a functional style.

## Motivating Example

Imagine developing an application that has information about all training centers in our city. A data model is simple, and you have a collection of training centers. Every training center has a schedule, where a schedule is represented as a collection of time slots. Every time slot is identified by the day of the week and optionally can contain a training session.
You, as a developer, have a task to find all training sessions for a given day. Here is a solution that I think most Java developers will provide.

```java
public List<TrainingSession> findSessionsByDay(final DayOfWeek day) {
    final List<TrainingCenter> trainingCenters = repository.findAll();
    final List<TrainingSession> sessions = new ArrayList<>();

    for (final TrainingCenter trainingCenter : trainingCenters) {
        final List<TimeSlot> schedule = trainingCenter.schedule();
        for (final TimeSlot slot : schedule) {
            if (slot.day() == day) {
                final Optional<TrainingSession> session = slot.session();
                if (session.isPresent()) {
                    sessions.add(session.get());
                }
            }
        }
    }

    return sessions;
}
```

I can't say that this solution is wrong, but every time I see something similar to this, I start asking myself why we are repeating ourselves over and over again while writing loops. It's a boilerplate, and it can be avoided by abstracting over it. We should describe what we want our program to do, not waste time writing the same loops and worry about loop index and temporary variables that will help us finish our task.

Let's review another solution probably written by a developer who just heard about Stream API and how cool it is.

```java
public List<TrainingSession> findSessionsByDay(final DayOfWeek day) {
    final List<TrainingSession> sessions = new ArrayList<>();
    repository.findAll().forEach(trainingCenter ->
                    trainingCenter.schedule().stream()
                            .filter(timeSlot -> timeSlot.day() == day)
                            .map(timeSlot -> timeSlot.session())
                            .forEach(session -> {
                                if (session.isPresent()) {
                                    sessions.add(session.get());
                                }
                            })
            );
    return sessions;
}
```

Wow, looks cool, looks declarative, but I think it's worse than the first solution. It demonstrates insufficient knowledge and understanding of Stream API and the whole point about it. The first mistake is using *forEach* since it can be replaced with some other operation, and the second one is *if* expression in a *forEach* instead of applying *filter* on *Optional*. People often get confused when dealing with nested structures like *List of Optional* or *Stream of Stream*. So, let's see how we should solve this problem.

```java
public List<TrainingSession> findSessionsByDay(final DayOfWeek day) {
    return repository.findAll().stream()
          .flatMap(trainingCenter -> trainingCenter.schedule().stream())
          .filter(timeSlot -> timeSlot.day() == day)
          .map(TimeSlot::session)
          .filter(Optional::isPresent)
          .map(Optional::get)
          .collect(Collectors.toList());
}
```

I hope you can capture the difference in this approach. We see that one *flatMap* solves our complex problem of nested structures. Maybe the easiest way to think about *flatMap* is that it allows you to remove one level of nesting or unwraps inner structure.

```java
public List<TrainingSession> findSessionsByDay(final DayOfWeek day) {
    return repository.findAll().stream()
            .flatMap(tc -> tc.schedule().stream())
            .filter(ts -> ts.day() == day)
            .flatMap(ts ->
                    ts.session().map(Stream::of).orElseGet(Stream::empty)
            )
            .collect(Collectors.toList());
}
```
This is our final solution, and the summary is that we reduced *15+* lines of code that contains two *for loops* and two *if expressions* with basically a one-liner, just with the two *flatMap* operations and one *filter*.

I'm sure that I got your attention now and that you finally realized the power of the declarative approach and functional style of programming. The solution is pretty readable and clearly expresses what our intention is. Maybe, you noticed that I used shorter variable names which is very common practice in a functional world, even though a typical Java developer will think that this is not expressive enough. 
There is no code nesting and ugly *pyramids of doom*, repeatable *for loops*, extensive usage of *if expressions*, and creation of *temporary variables* that will be filled upon iteration.

In the following few chapters, I will present a couple of concepts usually used in a functional world and, unfortunately, are not available in the Java Standard Library. We are going to use [*Vavr*](https://www.vavr.io/), which is a Java library that provides us immutable collections that already has operations like *map*, *flatMap*, *filter*, and others without the need to transform it to the *Stream* first. Besides the new collection API, *Vavr* gives us a few monadic containers types like *Option*, *Try*, and *Either* that we will explore to the details. These types are pretty interesting, and by applying them, your code becomes safer, declarative, and composable.

[Vavr Option](https://www.vavr.io/vavr-docs/#_option) is similar to [Java Optional](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html), but there are a couple of problems with Core Java implementation. If you want to find out more about it, check the following articles:

* [Why java.util.Optional is broken](https://blog.developer.atlassian.com/optional-broken/)
* [The agonizing death of an astronaut](http://blog.vavr.io/the-agonizing-death-of-an-astronaut/)
* [Do we have a better Option here?](https://softwaremill.com/do-we-have-better-option-here/)
* [In praise of Vavr's Option](https://blog.senacor.com/in-praise-of-vavrs-option/)
* [How Optional Breaks the Monad Laws and Why It Matters](https://www.sitepoint.com/how-optional-breaks-the-monad-laws-and-why-it-matters/)


## Option Type  

*Option* is a container type that represents an optional value. It helps to avoid *NullPointerException* in our code and to deal with missing values in a clear and elegant way. It is often used in data retrieval or validation logic when you don't need the details about potential errors. If an error happens, *Option* will be empty. [Vavr Option](https://www.vavr.io/vavr-docs/#_option) implementation is pretty similar to the [Option in Scala](https://www.scala-lang.org/api/current/scala/Option.html), where instances of *Option* are either an instance of *Some* or the *None*. Some common mistakes that developers are making while using *Option* will be reviewed in this chapter.

```java
final class UserService {

    public Option<User> getFromRepository(final Long id) {
        return repository.get(id);
    }

    public Option<User> getFromCache(final Long id) {
        return cache.get(id);
    }
}
```

Now we need to add another method to the *UserService*. Our task is to find a user by an id, but first, try to get a user from the cache and if it's not in the cache, try to find it in the repository. If the user is missing from the repository, throw some exception. A typical implementation that I have seen a lot of times is pretty similar to this:

```java
public User getUser(final Long id) {
    final Option<User> fromCache = service.getFromCache(id);

    if (fromCache.isEmpty()) {
        final Option<User> fromRepository = service.getFromRepository(id);
        if (fromRepository.isEmpty()) {
            throw new RuntimeException("User not found");
        }
        return userFromRepository.get();
    }

    return fromCache.get();
}
```

Again, no one can say it is an incorrect solution, but it just misses the point of *Option*. *If expressions* with unnecessary checks are still here. Earlier I mentioned that it is a *monadic type*, which means our *Options* can be composed. This is probably the shortest explanation ever, and if you are not planning to go fully functional, you don't have to know more about it, but I want to say that *Monad* is the essential functional structure, and it would be nice to find out more about it.
Now let's review the solution that demonstrates proper usage of *Options*.

```java
public User getUser(final Long id) {
    return service.getFromCache(id)
            .orElse(service.getFromRepository(id))
            .getOrElseThrow(() -> new RuntimeException("User not found"));
}
```

A much shorter solution with clear intentions without *if* checks. I think this code is pretty descriptive. Get a user from the cache or get it from the repository if there is no user throw exception. Methods like *orElse* and *getOrElseThrow* give us the possibility of composing multiple *Option* types, and you should use it all the time to avoid imperative style.

Suppose we need to modify our method to get the user in JSON format. In that case, the code will become pretty messy since there is a possibility of exceptions during the deserialization of an object. First, we need to introduce a method for deserialization that can throw an exception.

```java
private String toJson(final User user) throws JsonProcessingException {
    return mapper.writeValueAsString(user);
}
```

Now we should modify our method that after getting the user will map it to the JSON or return an empty JSON object upon exception.

```java
public String getUserJson(final Long id) {
    final Option<User> fromCache = service.getFromCache(id);

    if (fromCache.isEmpty()) {
        final Option<User> fromRepository = service.getFromRepository(id);

        if (fromRepository.isEmpty()) {
            throw new RuntimeException("User not found");
        }

        try {
            return toJson(fromRepository.get());
        } catch (final JsonProcessingException ex) {
            // handle, log exception
        }
    }

    try {
        return toJson(fromCache.get());
    } catch (final JsonProcessingException ex) {
        // handle, log exception
    }

    return "{}";
}
```

Huh, pretty messy if you ask me. Let's try Java 8 approach with Stream API, and maybe it will be a bit better.

```java
public String getUserJson(final Long id) {
    return service.getFromCache(userId)
            .orElse(service.getFromRepository(id))
            .map(user -> {
                try {
                    return mapper.writeValueAsString(user);
                } catch (final JsonProcessingException ex) {
                    // handle, log exception
                }
                return "{}";
            })
            .getOrElseThrow(() -> new RuntimeException("User not found"));
}
```

The more expressive and shorter solution, but there is a considerable lambda expression in *map* with *try/catch* block, which probably is not a great solution. Still, it is best to use Java since there is no way to avoid exception handling. For those coming from a Scala background *Try* will be a natural solution for this problem.

## Try or Throw?

*Try* is one more concept from [Vavr](https://www.javadoc.io/doc/io.vavr/vavr/0.9.2/io/vavr/control/Try.html). It's a monadic container type that represents a computation that may either result in an exception or return a successfully computed value.
Instances of *Try* are either an instance of *Success* or *Failure*. *Try* enables us to overcome problems with exception handling we had. Here is an improved solution utilizing the *Try*.

```java
public String getUserJson(final Long id) {
    return service.getFromCache(id)
            .orElse(service.getFromRepository(id))
            .map(user -> Try.of(() -> toJson(user)))
            .flatMap(userTry -> userTry.isSuccess()
                            ? Option.of(userTry.get())
                            : Option.of("{}"))
            .getOrElseThrow(() -> new RuntimeException("User not found"));
}
```

What we did to improve our code is basically just wrapping the invocation of an unsafe method into *Try*. Methods have the same meaning as the previous one, but it much more readable. If the method was successfully executed, the return value would be wrapped into *Try*. On the other hand, if an exception occurs, it will not interrupt our program, it will be just wrapped into *Try*, and we have absolute control over it. In the *flatMap* block, we first check if *Try* succeeds and then get its value inside. If computation failed, you could extract exception out of *Try* by invoking a method *getCause*. If you want a more straightforward solution, it is here:

```java
public String getUserJsonBetter(final Long id) {
    return service.getFromCache(id)
            .orElse(service.getFromRepository(id))
            .map(user -> Try.of(() -> toJson(user))
                    .recover(JsonProcessingException.class, "{}"))
            .map(Try::get)
            .getOrElseThrow(() -> new RuntimeException("User not found"));
}
```

What's new in this approach is using the *recover* method of *Try*, which accepts the exception type and a value that will be used upon recovery. You can chain multiple *recover* methods with different exception types since we implemented a recovery mechanism for our *Try* that handles all possible exceptions (in this case, it is just *JsonProcessingException*), our *Try* can't fail, so in the following *map* we are safe to get the value, and of course, these two *map* operations won't be executed if there is no user.

There is another recovery mechanism available for *Try*. To better demonstrate its usage, let's write some code again. Consider you have *UserService* like this:

```java
final class UserService {

    public User getFromRepository(final Long id) {
        final Option<User> user = repository.get(id);

        if (user.isEmpty()) {
            throw new NoUserInRepositoryException("No user in repository");
        }

        return user.get();
    }

    public User getFromCache(final Long id) {
        final Option<User> user = cache.get(id);

        if (user.isEmpty()) {
            throw new NoUserInCacheException("No user in cache");
        }

        return user.get();
    }
}
```

Like our first *UserService*, this one throws an exception when the user is not present. So, how to implement the safest possible method to retrieve a user in a JSON format when we know multiple exceptions can occur.

```java
public Try<String> getUser(final Long id) {
    return Try.of(() -> service.getFromCache(id))
            .recoverWith(
                    NoUserInCacheException.class,
                    Try.of(() -> service.getFromRepository(id))
            )
            .recoverWith(
                    NoUserInRepositoryException.class,
                    Try.success(new User(-1L, "default"))
            )
            .flatMap(user -> Try.of(() -> toJson(user)));
}
```

Here is another example of using *Try*, and it's perfectly fine to return *Try<*String*>* from a method. We have a strategy for handling exceptions if the user is missing. Still, we don't want to handle exceptions related to JSON serialization, so we just pushed that responsibility to the client-side. A developer who will use our method is aware that some exceptions are possible since we are returning a *Try<*String*>* and on him is to decide how to deal with potential errors. The first exception that can happen is when there is no user in a cache, so if *NoUserInCacheException* occurs, we will recover from that with another *Try*, which again can fail with *NoUserInRepositoryException*. Still,  we are always recovered successfully with the default user that we defined. Here I used *recoverWith* instead of *recover*, and the difference is that as a second parameter, *recoverWith* requires an instance of a *Try*. In contrast, recover accepts some value or just a regular method.

## The Beauty of Functional Style

Once again, I'm using the term *functional style* because our code looks *functional-ish*, but it's far from ideal by functional programming standards. So to point out that we are not allowed to call our code functional, saying it is written in a functional style is acceptable. Later I will refer you to the resources about pure functional programming in Scala since it's not entirely possible in Java, at least that I know. Scala has a few *state-of-the-art* libraries that unlock the full potential of functional programming. Even though we are far from that and started by just introducing basic functional concepts, we can notice that our code has improved.

Now let's see some more code. We have to implement a method that will accept a collection of endpoints, and we need to send a request to them and extract some information from the response. Information that we need from the response is the first and last name of the user, so before we start to write our business logic, we need to implement a class that will contain our user details.

```java
class UserInfo {

    private String firstName;
    private String lastName;

    public UserInfo(final String firstName, final String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // getter and setters
}
```

Now typical before Java 8 approach would go like this:

```java
public List<UserInfo> fetch(final List<String> urls) {
    final List<UserInfo> responses = new ArrayList<>();

    for (final String url : urls) {
        final HttpGet request = new HttpGet(url);
        
        try {
            final HttpResponse response = http.execute(request);
            final String json = EntityUtils.toString(
                response.getEntity(),
                "UTF-8"
            );
            final JsonNode data = mapper.readTree(json).path("data");
            final UserInfo userInfo = new UserInfo(
                    data.get("first_name").toString(),
                    data.get("last_name").toString()
            );
            responses.add(userInfo);
        } catch (final IOException ex) {
            // handle exception
        }
    }

    return responses;
}
```

With Core Java, we had to implement a simple temporary class just to represent a user with two fields, and then we have a colossal method where multiple exceptions can occur, but we wrapped it all into one *try/catch* block. Java 8 probably can help us to improve our code, but not as much as *Vavr* would. First, let's review Java 8 solution:

```java
public List<UserInfo> fetch(final List<String> urls) {
    return urls.stream()
            .map(HttpGet::new)
            .map(req -> {
                UserInfo userInfo = null;
                try {
                    final HttpResponse res = http.execute(req);
                    final String json = EntityUtils.toString(
                            res.getEntity(),
                            "UTF-8"
                    );
                    final JsonNode data = 
                            mapper.readTree(json).path("data");
                    userInfo = new UserInfo(
                            data.get("first_name").toString(),
                            data.get("last_name").toString()
                    );
                } catch (final IOException ex) {
                    // handle exception
                }
                return Optional.ofNullable(userInfo);
            })
            .flatMap(u -> u.map(Stream::of).orElseGet(Stream::empty))
            .collect(Collectors.toList());
}
```

Maybe a bit better, but once again, huge lambda expression with *try/catch*, which is inevitable without data type like we introduced in a previous chapter. If you are confused with a flatMap part, it can be replaced with these two lines:

```java
.filter(Optional::isPresent)
.map(Optional::get)
```

Basically, it just checks whether an *Optional* is present or not before trying to get inside the value. Java 8 Stream API is not ideal. It represents a try to imitate compositions of combinators like *map* and *flatMap* on collections of more functional languages, which support combinators by default. Not so good implementation of Stream API forces us to wrap the result of *Option* into that *flatMap* operation into *Stream*, which wouldn't be necessary if Java Collections were more functional and had combinators implemented on them directly.

```java
public List<Tuple2<String, String>> fetch(final List<String> urls) {
    return urls.map(HttpGet::new)
        .flatMap(req -> Try.of(() -> http.execute(req)))
        .flatMap(res -> Try.of(() -> EntityUtils.toString(res.getEntity(), "UTF-8")))
        .flatMap(entity -> Try.of(() -> mapper.readTree(entity).path("data")))
        .map(json -> Tuple.of(
                json.get("first_name").toString(),
                json.get("last_name").toString())
        );
}
```

And finally, here is a *Vavr* based solution, which is shorter, and we didn't have to implement some temporary class to store results. We just used [Tuple](https://www.vavr.io/vavr-docs/#_tuples), which is extremely useful in many cases. Still, unfortunately, Java doesn't support them by default.
Of course, this method would be much more compacted if we extracted all those methods for sending requests, parsing JSON, and mapping to Tuple. We wrote a lot of code utilizing the *Try* type, and to summarize, I want to note that *Try* is a great approach to gain full control of your code by taming exceptions, which are tricky and very dangerous since they can change the flow of a program secretly. Use *Try* whenever you need to make safe wrappers around API, which you do not control. And if you want an even more expressive approach to deal with exceptions, but equally powerful like a *Try*, *Vavr* has something for you.

<figure>
  <img src="{{ site.url }}{{ site.baseurl }}/images/fp/fp.png" class="center-image ">
    <figcaption>Image source: <a href="https://impurepics.com/">impurepics.com</a></figcaption>
</figure>

## Either Type

[*Either*](https://www.javadoc.io/doc/io.vavr/vavr/0.9.0/io/vavr/control/Either.html) is the last type that we are going to explore from Vavr.
*Either* represents a value of two possible types. An *Either* is either a *Left* or a *Right*. Both *Left* and *Right* can wrap any type. By convention, *Right* is right, containing the successful result, and the *Left* side should contain the error. This is not mandatory, but you should stick to the convention. I bet you noticed some similarity here with a *Try*, and the only difference is that error in *Try* must be an instance of a *Throwable*, while an error in *Either* can be whatever you want. You can use the custom message as a *String*, or you can wrap the exception. Example usage of *Either*:

```java
public Either<JsonProcessingException, String> toJson(final User user) {
    try {
        return Either.right(mapper.writeValueAsString(user));
    } catch (final JsonProcessingException ex) {
        return Either.left(ex);
    }
}
```

As I mentioned earlier, *Either* is more expressive than a *Try* since from a method signature, you have complete knowledge about the behavior of a method. Are you going to use *Either* or *Try* is totally up to you, and whatever you choose, it certainly better than constantly using *try/catch* blocks and checked exceptions. *Either* have very rich [API](https://www.javadoc.io/doc/io.vavr/vavr/0.9.0/io/vavr/control/Either.html) that you should explore. You should know that *Either* have also *map* which will apply mapping function on the *Right* side while completely ignoring *Left* side. Even though you need to *map* over *Left* side, you have the *mapLeft* method. Also, the *bimap* method is available, which accepts two functions, left and right mapper functions. *flatMap* and *filter* combinators are available, which by default operate on the *Right* side. There are plenty of exciting operators available in the *Either*, and I presented the most used ones.

## Scala and Functional Programming

As mentioned earlier, Scala is a much more friendly language for functional programming than Java. Scala natively supports all concepts presented here with Java and Vavr. Of course, there are much more futures that make Scala more powerful than Java. You should be aware of a few libraries if you are interested in purely functional programming in Scala. Most popular ones are: [ZIO](https://zio.dev/), [Cats](https://typelevel.org/cats/) and [Cats Effect](https://typelevel.org/cats-effect/), [Monix](https://monix.io/), and [Scalaz](https://scalaz.github.io/7/).

## Summary

We covered many new concepts that are pretty unusual in the Java world. Honestly, they are simple enough yet unbelievably powerful, whose effects just can't be ignored. The amount of confidence you felt for the first working with *Try* or *Either* should be present all the time while coding, which is not the case while handling the exceptions traditionally. Also, the composition of *Options* is pretty cool. Also, I noticed that many Java developers using it are unaware that you can apply *map*, *flatMap*, and *filter* on it, so I tried to show some common patterns on how to utilize those concepts properly.

I hope you have learned something new and realized the benefits offered by writing Java in a functional style. Of course, many concepts are not covered, like using immutable collections and immutable data types, but I think it’s enough to start improving your Java codebase.

You can find me at:
* [Linkedin](https://www.linkedin.com/in/aleksandar-skrbic/)
* [Github](https://github.com/aleksandarskrbic)

Or just send me a question to [skrbic.alexa@gmail.com]()