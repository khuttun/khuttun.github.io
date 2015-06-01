---
layout: post
title: Polymorphic Message Queue in C++11
---

Recently I was working on a C++ hobby project where I needed a simple, lightweight way to pass messages between threads. The message passing system would need to fulfill two requirements:

* The threads would need to pass different kinds of data between them, so the messaging system should be able to support messages with any type of payload.
* The payload data passed between threads could be relatively large, so it shouldn't be copied around. Only it's ownership should be passed from sender to the queue, and from the queue to the receiver.

I decided to try writing my own message passing system utilizing C++11 move semantics.

The idea for the high level design of the message passing system is to have a basic message type that has no payload data. Other message types with different payload data types will then be derived from the basic type. The messages shouldn't be copyable but they should be movable.

First shot at the `Msg` class implementing the basic message type:

{% highlight c++ %}
class Msg
{
public:
    Msg(int msgId)
      : msgId_(msgId)
    {
    }

    // Enable moving
    Msg(Msg&&) = default;
    Msg& operator=(Msg&&) = default;

    // Disable copying
    Msg(const Msg&) = delete;
    Msg& operator=(const Msg&) = delete;

    virtual ~Msg() {}

    int getMsgId() const
    {
        return msgId_;
    }

private:
    int msgId_;
};

{% endhighlight %}

`Msg` only has a message ID that identifies what kind of a message it is. There's no payload data. The `DataMsg` class template implements the message type with payload data:

{% highlight c++ %}
template <typename PayloadType>
class DataMsg : public Msg
{
public:
    template <typename ... Args>
    DataMsg(int msgId, Args&& ... args)
      : Msg(msgId),
        pl_(new PayloadType(std::forward<Args>(args) ...))
    {
    }

    // Enable moving
    DataMsg(DataMsg&&) = default;
    DataMsg& operator=(DataMsg&&) = default;

    // Disable copying
    DataMsg(const DataMsg&) = delete;
    DataMsg& operator=(const DataMsg&) = delete;

    virtual ~DataMsg() {}

    const PayloadType& getPayload() const
    {
        return *pl_;
    }

private:
    std::unique_ptr<PayloadType> pl_;
};
{% endhighlight %}

`DataMsg` owns the payload data, so it only gives const reference to it in `getPayload()`. Notice also that `DataMsg` uses [perfect forwarding](http://thbecker.net/articles/rvalue_references/section_07.html) to pass its constructor arguments to `PayloadType` constructor.

Okay, then what about the message queue itself? The `Queue` class's `put()` member function might look something like this (ignoring locks and condition variables for now):

{% highlight c++ %}
...
void put(Msg&& msg)
{
    queue_.push_back(std::unique_ptr<Msg>(new Msg(std::move(msg))));
}
...
private:
    std::list<std::unique_ptr<Msg>> queue_;
...
{% endhighlight %}

`put()` takes the message in as an rvalue reference to signify that the ownership of the message is transferred to the queue. A new `Msg` instance is move constructed, wrapped to `unique_ptr` and pushed to the list.

But hold on, *a new `Msg` instance is contructed* - we lose the polymorphism here, everything is "flattened" to a `Msg`! We would need a way to move construct the messages in such a way that the move constructor of the correct `Msg` derivant would dynamically be called.

There is a well known solution for the same problem with copy constructors called [virtual constructor idiom](https://isocpp.org/wiki/faq/virtual-functions#virtual-ctors). It's implemented by adding a virtual `clone()` member function to all the classes in the class hierarchy. Let's extend this idiom to move constructors and add virtual `move()` functions, "virtual move constructors", to our message classes:

{% highlight c++ %}
class Msg
{
public:
    ...
    virtual std::unique_ptr<Msg> move()
    {
        return std::unique_ptr<Msg>(new Msg(std::move(*this)));
    }
    ...

template <typename PayloadType>
class DataMsg : public Msg
{
public:
    ...
    virtual std::unique_ptr<Msg> move() override
    {
        return std::unique_ptr<Msg>(new DataMsg<PayloadType>(std::move(*this)));
    }
    ...
{% endhighlight %}

Calling `move()` on a message object moves the data from that object into a new one. The original object is left in valid but unspecified state. Now we can rewrite `Queue`'s `put()` and `get()` member functions (again ignoring thread synchronization for now):

{% highlight c++ %}
...
void put(Msg&& msg)
{
    queue_.push_back(msg.move());
}
...
std::unique_ptr<Msg> get()
{
    auto msg = queue_.front()->move();
    queue_.pop_front();
    return msg;
}
...
{% endhighlight %}

`put()` moves the data out from the `Msg` instance it receives as a parameter into a new instance, and pushes that new instance to the queue. `get()` moves the data out from the `Msg` instance at the head of the queue into a new instance and returns the new instance.

That forms the basis of the messaging system implementation. Get the full code from [GitHub](https://github.com/khuttun/PolyM).
