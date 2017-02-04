---
layout: post
title: Implementing State Machines with std::variant
---

C++17 brings us [`std::variant`](http://en.cppreference.com/w/cpp/utility/variant) that will enable new kinds of type-safe programming patterns by providing the ability to construct [sum types](https://en.wikipedia.org/wiki/Algebraic_data_type). One interesting potential use of `std::variant` was presented by Ben Deane in his 2016 CppCon talk ["Using Types Effectively"](https://www.youtube.com/watch?v=ojZbFIQSdl8). If you haven't yet, you should go and watch the talk. It presents many clever ideas on how to use the C++ type system to your advantage.

The idea presented in the talk was to use `std::variant` to represent state machines. The main motivation was to make illegal states unrepresentable by embedding the variables only needed in one state inside a structure representing the state. All the state structures are then combined in a `std::variant` object to represent the current state. The example used in the talk showed the structure declarations for a state machine modeling some kind of server connection:

{% highlight c++ %}
struct Connection {
    std::string m_serverAddress;

    struct Disconnected {};
    struct Connecting {};
    struct Connected {
        ConnectionId m_id;
        std::chrono::system_clock::time_point m_connectedTime;
        std::optional<std::chrono::milliseconds> m_lastPingTime;
    };
    struct ConnectionInterrupted {
        std::chrono::system_clock::time_point m_disconnectedTime;
        Timer m_reconnectTimer;
    };

    std::variant<Disconnected,
                 Connecting,
                 Connected,
                 ConnectionInterrupted> m_connection;
};
{% endhighlight %}

The idea is that `m_lastPingTime` is only needed in `Connected` state, `m_disconnectedTime` only in `ConnectionInterrupted` state, and so on.

The talk didn't go further into details on how to implement the full state machine logic with state transitions, transition actions etc. using this pattern. In this post, I explore a possible implementation.

### Representing the Transitions

A state transition is made simply by assigning a new value to the state variant, `m_connection` in the case of our example. For example, the `Connection` class could have a function

{% highlight c++ %}
void disconnected()
{
    m_connection = Disconnected();
}
{% endhighlight %}

that would be called when a disconnection event is detected. This is the simplest transition implementation, as we always move to the same state as a result of the event, `Disconnected` in this case, regardless of the state the connection is when the event occurs.

A more interesting situation arises when the transition invoked by the event depends on the current state. This requires us to inspect which alternative the variant currently holds. A natural and safe way to do this is to use [`std::visit`](http://en.cppreference.com/w/cpp/utility/variant/visit). The visitor passed to `std::visit` is required to be callable with all the alternative types the variant can hold, otherwise the program is ill-formed. This property gives us a compile-time check that we handle all the source states in our transition visitors.

The visitor can also return a value. A useful value to return from a transition visitor is the target state of the transition. As an example, consider a visitor handling a connection interrupted event:

{% highlight c++ %}
using State = std::variant<
    Disconnected, Connecting, Connected, ConnectionInterrupted>;

struct InterruptedEvent {
    State operator()(const Disconnected&){ return Disconnected(); }
    State operator()(const Connecting&){ return Disconnected(); }
    State operator()(const Connected&)
    {
        return ConnectionInterrupted{std::chrono::system_clock::now(), 5000};
    }
    State operator()(const ConnectionInterrupted& s){ return s; }
};
{% endhighlight %}

If we get the interruption event when the connection hasn't been established yet (`Disconnected` or `Connecting` state), we move to (or stay in) the `Disconnected` state. From `Connected` state we move to `ConnectionInterrupted` state. If we are already in the `ConnectionInterrupted` state, we stay there.

The visitor can be used like this:

{% highlight c++ %}
void interrupted()
{
    m_connection = std::visit(InterruptedEvent(), m_connection);
}
{% endhighlight %}

### Adding Transition Actions

What if we'd like to execute some action in the `Connection` class when a transition is initiated in the visitor, for example, notify observers of the `Connection` about connection interruption? The visitor needs access to the `Connection` object. We can "capture" the object to the visitor:

{% highlight c++ %}
struct InterruptedEvent {
    InterruptedEvent(Connection& c) : m_c(c) {}
    State operator()(const Disconnected&){ return Disconnected(); }
    State operator()(const Connecting&){ return Disconnected(); }
    State operator()(const Connected&)
    {
        const auto now = std::chrono::system_clock::now();
        m_c.notifyInterrupted(now);
        return ConnectionInterrupted{now, 5000};
    }
    State operator()(const ConnectionInterrupted& s){ return s; }

private:
    Connection& m_c;
};
{% endhighlight %}

and pass the reference when calling `std::visit`:

{% highlight c++ %}
void interrupted()
{
    m_connection = std::visit(InterruptedEvent(*this), m_connection);
}
{% endhighlight %}

See the full code [here](https://gist.github.com/khuttun/ec546b2b23cfc37124f2395c9a2a8014).
