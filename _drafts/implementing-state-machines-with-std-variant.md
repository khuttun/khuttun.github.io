---
layout: post
title: Implementing State Machines with std::variant
---

C++17 brings us [`std::variant`](http://en.cppreference.com/w/cpp/utility/variant) that will enable new kinds of type-safe programming patterns by providing the ability to construct [sum types](https://en.wikipedia.org/wiki/Algebraic_data_type). One interesting potential use of `std::variant` was presented by [Ben Deane](https://twitter.com/ben_deane) in his 2016 CppCon talk ["Using Types Effectively"](https://www.youtube.com/watch?v=ojZbFIQSdl8). If you haven't yet, you should go and watch the talk. It presents many clever ideas on how to use the C++ type system to your advantage.

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

The idea is that `m_lastPingTime` is only needed in `Connected` state, `m_disconnectedTime` only in `ConnectionInterrupted` state etc.
