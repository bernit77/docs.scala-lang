---
layout: singlepage-overview
title: The Scala Actors API

partof: actors

languages: [zh-cn, es]

permalink: /overviews/core/:title.html
---

**Philipp Haller and Stephen Tu**

## Introduction

This guide describes the API of the `scala.actors` package of Scala 2.8/2.9. The organization follows groups of types that logically belong together. The trait hierarchy is taken into account to structure the individual sections. The focus is on the run-time behavior of the various methods that these traits define, thereby complementing the existing Scaladoc-based API documentation.

NOTE: In Scala 2.10 the Actors library is deprecated and will be removed in future Scala releases.

## The actor traits Reactor, ReplyReactor, and Actor

### The Reactor trait

`Reactor` is the super trait of all actor traits. Extending this trait allows defining actors with basic capabilities to send and receive messages.

The behavior of a `Reactor` is defined by implementing its `act` method. The `act` method is executed once the `Reactor` is started by invoking `start`, which also returns the `Reactor`. The `start` method is *idempotent* which means that invoking it on an actor that has already been started has no effect.

The `Reactor` trait has a type parameter `Msg` which indicates the type of messages that the actor can receive.

Invoking the `Reactor`'s `!` method sends a message to the receiver. Sending a message using `!` is asynchronous which means that the sending actor does not wait until the message is received; its execution continues immediately. For example, `a ! msg` sends `msg` to `a`. All actors have a *mailbox* which buffers incoming messages until they are processed.

The `Reactor` trait also defines a `forward` method. This method is inherited from `OutputChannel`. It has the same effect as the `!` method. Subtraits of `Reactor`, in particular the `ReplyReactor` trait, override this method to enable implicit reply destinations (see below).

A `Reactor` receives messages using the `react` method. `react` expects an argument of type `PartialFunction[Msg, Unit]` which defines how messages of type `Msg` are handled once they arrive in the actor's mailbox. In the following example, the current actor waits to receive the string "Hello", and then prints a greeting:

    react {
      case "Hello" => println("Hi there")
    }

Invoking `react` never returns. Therefore, any code that should run after a message has been received must be contained inside the partial function that is passed to `react`. For example, two messages can be received in sequence by nesting two invocations of `react`:

    react {
      case Get(from) =>
        react {
          case Put(x) => from ! x
        }
    }

The `Reactor` trait also provides control structures which simplify programming with `react`.

#### Termination and execution states

The execution of a `Reactor` terminates when the body of its `act` method has run to completion. A `Reactor` can also terminate itself explicitly using the `exit` method. The return type of `exit` is `Nothing`, because `exit` always throws an exception. This exception is only used internally, and should never be caught.

A terminated `Reactor` can be restarted by invoking its `restart` method. Invoking `restart` on a `Reactor` that has not terminated, yet, throws an `IllegalStateException`. Restarting a terminated actor causes its `act` method to be rerun.

`Reactor` defines a method `getState` which returns the actor's current execution state as a member of the `Actor.State` enumeration. An actor that has not been started, yet, is in state `Actor.State.New`. An actor that can run without waiting for a message is in state `Actor.State.Runnable`. An actor that is suspended, waiting for a message is in state `Actor.State.Suspended`. A terminated actor is in state `Actor.State.Terminated`.

#### Exception handling

The `exceptionHandler` member allows defining an exception handler that is enabled throughout the entire lifetime of a `Reactor`:

    def exceptionHandler: PartialFunction[Exception, Unit]

`exceptionHandler` returns a partial function which is used to handle exceptions that are not otherwise handled: whenever an exception propagates out of the body of a `Reactor`'s `act` method, the partial function is applied to that exception, allowing the actor to run clean-up code before it terminates. Note that the visibility of `exceptionHandler` is `protected`.

Handling exceptions using `exceptionHandler` works well together with the control structures for programming with `react`. Whenever an exception has been handled using the partial function returned by `exceptionHandler`, execution continues with the current continuation closure. Example:

    loop {
      react {
        case Msg(data) =>
          if (cond) // process data
          else throw new Exception("cannot process data")
      }
    }

Assuming that the `Reactor` overrides `exceptionHandler`, after an exception thrown inside the body of `react` is handled, execution continues with the next loop iteration.

### The ReplyReactor trait

The `ReplyReactor` trait extends `Reactor[Any]` and adds or overrides the following methods:

- The `!` method is overridden to obtain a reference to the current
  actor (the sender); together with the actual message, the sender
  reference is transferred to the mailbox of the receiving actor. The
  receiver has access to the sender of a message through its `sender`
  method (see below).

- The `forward` method is overridden to obtain a reference to the
  `sender` of the message that is currently being processed. Together
  with the actual message, this reference is transferred as the sender
  of the current message. As a consequence, `forward` allows
  forwarding messages on behalf of actors different from the current
  actor.

- The added `sender` method returns the sender of the message that is
  currently being processed. Given the fact that a message might have
  been forwarded, `sender` may not return the actor that actually sent
  the message.

- The added `reply` method sends a message back to the sender of the
  last message. `reply` is also used to reply to a synchronous message
  send or a message send with future (see below).

- The added `!?` methods provide *synchronous message sends*. Invoking
  `!?` causes the sending actor to wait until a response is received
  which is then returned. There are two overloaded variants. The
  two-parameter variant takes in addition a timeout argument (in
  milliseconds), and its return type is `Option[Any]` instead of
  `Any`. If the sender does not receive a response within the
  specified timeout period, `!?` returns `None`, otherwise it returns
  the response wrapped in `Some`.

- The added `!!` methods are similar to synchronous message sends in
  that they allow transferring a response from the receiver. However,
  instead of blocking the sending actor until a response is received,
  they return `Future` instances. A `Future` can be used to retrieve
  the response of the receiver once it is available; it can also be
  used to find out whether the response is already available without
  blocking the sender. There are two overloaded variants. The
  two-parameter variant takes in addition an argument of type
  `PartialFunction[Any, A]`. This partial function is used for
  post-processing the receiver's response. Essentially, `!!` returns a
  future which applies the partial function to the response once it is
  received. The result of the future is the result of this
  post-processing.

- The added `reactWithin` method allows receiving messages within a
  given period of time. Compared to `react` it takes an additional
  parameter `msec` which indicates the time period in milliseconds
  until the special `TIMEOUT` pattern matches (`TIMEOUT` is a case
  object in package `scala.actors`). Example:

    reactWithin(2000) {
      case Answer(text) => // process text
      case TIMEOUT => println("no answer within 2 seconds")
    }

  The `reactWithin` method also allows non-blocking access to the
  mailbox. When specifying a time period of 0 milliseconds, the
  mailbox is first scanned to find a matching message. If there is no
  matching message after the first scan, the `TIMEOUT` pattern
  matches. For example, this enables receiving certain messages with a
  higher priority than others:

    reactWithin(0) {
      case HighPriorityMsg => // ...
      case TIMEOUT =>
        react {
          case LowPriorityMsg => // ...
        }
    }

  In the above example, the actor first processes the next
  `HighPriorityMsg`, even if there is a `LowPriorityMsg` that arrived
  earlier in its mailbox. The actor only processes a `LowPriorityMsg`
  *first* if there is no `HighPriorityMsg` in its mailbox.

In addition, `ReplyReactor` adds the `Actor.State.TimedSuspended` execution state. A suspended actor, waiting to receive a message using `reactWithin` is in state `Actor.State.TimedSuspended`.

### The Actor trait

The `Actor` trait extends `ReplyReactor` and adds or overrides the following members:

- The added `receive` method behaves like `react` except that it may
  return a result. This is reflected in its type, which is polymorphic
  in its result: `def receive[R](f: PartialFunction[Any, R]): R`.
  However, using `receive` makes the actor more heavyweight, since
  `receive` blocks the underlying thread while the actor is suspended
  waiting for a message. The blocked thread is unavailable to execute
  other actors until the invocation of `receive` returns.

- The added `link` and `unlink` methods allow an actor to link and unlink
  itself to and from another actor, respectively. Linking can be used
  for monitoring and reacting to the termination of another actor. In
  particular, linking affects the behavior of invoking `exit` as
  explained in the API documentation of the `Actor` trait.

- The `trapExit` member allows reacting to the termination of linked
  actors independently of the exit reason (that is, it does not matter
  whether the exit reason is `'normal` or not). If an actor's `trapExit`
  member is set to `true`, this actor will never terminate because of
  linked actors. Instead, whenever one of its linked actors terminates
  it will receive a message of type `Exit`. The `Exit` case class has two
  members: `from` refers to the actor that terminated; `reason` refers to
  the exit reason.

#### Termination and execution states

When terminating the execution of an actor, the exit reason can be set
explicitly by invoking the following variant of `exit`:

    def exit(reason: AnyRef): Nothing

An actor that terminates with an exit reason different from the symbol
`'normal` propagates its exit reason to all actors linked to it. If an
actor terminates because of an uncaught exception, its exit reason is
an instance of the `UncaughtException` case class.

The `Actor` trait adds two new execution states. An actor waiting to
receive a message using `receive` is in state
`Actor.State.Blocked`. An actor waiting to receive a message using
`receiveWithin` is in state `Actor.State.TimedBlocked`.

## Control structures

The `Reactor` trait defines control structures that simplify programming
with the non-returning `react` operation. Normally, an invocation of
`react` does not return. If the actor should execute code subsequently,
then one can either pass the actor's continuation code explicitly to
`react`, or one can use one of the following control structures which
hide these continuations.

The most basic control structure is `andThen`. It allows registering a
closure that is executed once the actor has finished executing
everything else.

    actor {
      {
        react {
          case "hello" => // processing "hello"
        }: Unit
      } andThen {
        println("hi there")
      }
    }

For example, the above actor prints a greeting after it has processed
the `"hello"` message. Even though the invocation of `react` does not
return, we can use `andThen` to register the code which prints the
greeting as the actor's continuation.

Note that there is a *type ascription* that follows the `react`
invocation (`: Unit`). Basically, it lets you treat the result of
`react` as having type `Unit`, which is legal, since the result of an
expression can always be dropped. This is necessary to do here, since
`andThen` cannot be a member of type `Nothing` which is the result
type of `react`. Treating the result type of `react` as `Unit` allows
the application of an implicit conversion which makes the `andThen`
member available.

The API provides a few more control structures:

- `loop { ... }`. Loops indefinitely, executing the code in braces in
  each iteration. Invoking `react` inside the loop body causes the
  actor to react to a message as usual. Subsequently, execution
  continues with the next iteration of the same loop.

- `loopWhile (c) { ... }`. Executes the code in braces while the
  condition `c` returns `true`. Invoking `react` in the loop body has
  the same effect as in the case of `loop`.

- `continue`. Continues with the execution of the current continuation
  closure. Invoking `continue` inside the body of a `loop` or
  `loopWhile` will cause the actor to finish the current iteration and
  continue with the next iteration. If the current continuation has
  been registered using `andThen`, execution continues with the
  closure passed as the second argument to `andThen`.

The control structures can be used anywhere in the body of a `Reactor`'s
`act` method and in the bodies of methods (transitively) called by
`act`. For actors created using the `actor { ... }` shorthand the control
structures can be imported from the `Actor` object.

#### Futures

The `ReplyReactor` and `Actor` traits support result-bearing message
send operations (the `!!` methods) that immediately return a
*future*. A future, that is, an instance of the `Future` trait, is a
handle that can be used to retrieve the response to such a message
send-with-future.

The sender of a message send-with-future can wait for the future's
response by *applying* the future. For example, sending a message using
`val fut = a !! msg` allows the sender to wait for the result of the
future as follows: `val res = fut()`.

In addition, a `Future` can be queried to find out whether its result
is available without blocking using the `isSet` method.

A message send-with-future is not the only way to obtain a
future. Futures can also be created from computations directly.
In the following example, the computation body is started to
run concurrently, returning a future for its result:

    val fut = Future { body }
    // ...
    fut() // wait for future

What makes futures special in the context of actors is the possibility
to retrieve their result using the standard actor-based receive
operations, such as `receive` etc. Moreover, it is possible to use the
event-based operations `react` and `reactWithin`. This enables an actor to
wait for the result of a future without blocking its underlying
thread.

The actor-based receive operations are made available through the
future's `inputChannel`. For a future of type `Future[T]`, its type is
`InputChannel[T]`. Example:

    val fut = a !! msg
    // ...
    fut.inputChannel.react {
      case Response => // ...
    }

## Channels

Channels can be used to simplify the handling of messages that have
different types but that are sent to the same actor. The hierarchy of
channels is divided into `OutputChannel`s and `InputChannel`s.

`OutputChannel`s can be sent messages. An `OutputChannel` `out`
supports the following operations.

- `out ! msg`. Asynchronously sends `msg` to `out`. A reference to the
  sending actor is transferred as in the case where `msg` is sent
  directly to an actor.

- `out forward msg`. Asynchronously forwards `msg` to `out`. The
  sending actor is determined as in the case where `msg` is forwarded
  directly to an actor.

- `out.receiver`. Returns the unique actor that is receiving messages
  sent to the `out` channel.

- `out.send(msg, from)`. Asynchronously sends `msg` to `out` supplying
  `from` as the sender of the message.

Note that the `OutputChannel` trait has a type parameter that specifies
the type of messages that can be sent to the channel (using `!`,
`forward`, and `send`). The type parameter is contravariant:

    trait OutputChannel[-Msg]

Actors can receive messages from `InputChannel`s. Like `OutputChannel`,
the `InputChannel` trait has a type parameter that specifies the type of
messages that can be received from the channel. The type parameter is
covariant:

    trait InputChannel[+Msg]

An `InputChannel[Msg]` `in` supports the following operations.

- `in.receive { case Pat1 => ... ; case Patn => ... }` (and similarly,
  `in.receiveWithin`). Receives a message from `in`. Invoking
  `receive` on an input channel has the same semantics as the standard
  `receive` operation for actors. The only difference is that the
  partial function passed as an argument has type
  `PartialFunction[Msg, R]` where `R` is the return type of `receive`.

- `in.react { case Pat1 => ... ; case Patn => ... }` (and similarly,
  `in.reactWithin`). Receives a message from `in` using the
  event-based `react` operation. Like `react` for actors, the return
  type is `Nothing`, indicating that invocations of this method never
  return. Like the `receive` operation above, the partial function
  passed as an argument has a more specific type:

    PartialFunction[Msg, Unit]

### Creating and sharing channels

Channels are created using the concrete `Channel` class. It extends both
`InputChannel` and `OutputChannel`. A channel can be shared either by
making the channel visible in the scopes of multiple actors, or by
sending it in a message.

The following example demonstrates scope-based sharing.

    actor {
      var out: OutputChannel[String] = null
      val child = actor {
        react {
          case "go" => out ! "hello"
        }
      }
      val channel = new Channel[String]
      out = channel
      child ! "go"
      channel.receive {
        case msg => println(msg.length)
      }
    }

Running this example prints the string `"5"` to the console. Note that
the `child` actor has only access to `out` which is an
`OutputChannel[String]`. The `channel` reference, which can also be used
to receive messages, is hidden. However, care must be taken to ensure
the output channel is initialized to a concrete channel before the
`child` sends messages to it. This is done using the `"go"` message. When
receiving from `channel` using `channel.receive` we can make use of the
fact that `msg` is of type `String`; therefore, it provides a `length`
member.

An alternative way to share channels is by sending them in
messages. The following example demonstrates this.

    case class ReplyTo(out: OutputChannel[String])

    val child = actor {
      react {
        case ReplyTo(out) => out ! "hello"
      }
    }

    actor {
      val channel = new Channel[String]
      child ! ReplyTo(channel)
      channel.receive {
        case msg => println(msg.length)
      }
    }

The `ReplyTo` case class is a message type that we use to distribute a
reference to an `OutputChannel[String]`. When the `child` actor receives a
`ReplyTo` message it sends a string to its output channel. The second
actor receives a message on that channel as before.

## Schedulers

A `Reactor` (or an instance of a subtype) is executed using a
*scheduler*. The `Reactor` trait introduces the `scheduler` member which
returns the scheduler used to execute its instances:

    def scheduler: IScheduler

The run-time system executes actors by submitting tasks to the
scheduler using one of the `execute` methods defined in the `IScheduler`
trait. Most of the trait's other methods are only relevant when
implementing a new scheduler from scratch, which is rarely necessary.

The default schedulers used to execute instances of `Reactor` and `Actor`
detect the situation when all actors have finished their
execution. When this happens, the scheduler shuts itself down
(terminating any threads used by the scheduler). However, some
schedulers, such as the `SingleThreadedScheduler` (in package `scheduler`)
have to be shut down explicitly by invoking their `shutdown` method.

The easiest way to create a custom scheduler is by extending
`SchedulerAdapter`, implementing the following abstract member:

    def execute(fun: => Unit): Unit

Typically, a concrete implementation would use a thread pool to
execute its by-name argument `fun`.

## Remote Actors

This section describes the remote actors API. Its main interface is
the `RemoteActor` object in package `scala.actors.remote`. This object
provides methods to create and connect to remote actor instances. In
the code snippets shown below we assume that all members of
`RemoteActor` have been imported; the full list of imports that we use
is as follows:

    import scala.actors._
    import scala.actors.Actor._
    import scala.actors.remote._
    import scala.actors.remote.RemoteActor._

### Starting remote actors

A remote actor is uniquely identified by a [`Symbol`](https://www.scala-lang.org/api/current/scala/Symbol.html). This symbol is
unique to the JVM instance on which the remote actor is executed. A
remote actor identified with name `'myActor` can be created as follows.

    class MyActor extends Actor {
      def act() {
        alive(9000)
        register('myActor, self)
        // ...
      }
    }

Note that a name can only be registered with a single (alive) actor at
a time. For example, to register an actor *A* as `'myActor`, and then
register another actor *B* as `'myActor`, one would first have to wait
until *A* terminated. This requirement applies across all ports, so
simply registering *B* on a different port as *A* is not sufficient.

### Connecting to remote actors

Connecting to a remote actor is just as simple. To obtain a remote
reference to a remote actor running on machine `myMachine`, on port
8000, with name `'anActor`, use `select` in the following manner:

    val myRemoteActor = select(Node("myMachine", 8000), 'anActor)

The actor returned from `select` has type `AbstractActor` which provides
essentially the same interface as a regular actor, and thus supports
the usual message send operations:

    myRemoteActor ! "Hello!"
    receive {
      case response => println("Response: " + response)
    }
    myRemoteActor !? "What is the meaning of life?" match {
      case 42   => println("Success")
      case oops => println("Failed: " + oops)
    }
    val future = myRemoteActor !! "What is the last digit of PI?"

Note that `select` is lazy; it does not actually initiate any network
connections. It simply creates a new `AbstractActor` instance which is
ready to initiate a new network connection when needed (for instance,
when `!` is invoked).
