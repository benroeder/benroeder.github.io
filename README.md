##This page intentionally left blank

# Integrating Nameko and StatsD

In this article we're going to learn how to create a [StatsD](https://github.com/jsocol/pystatsd) dependency provider for the [Nameko](https://github.com/nameko).


## The challenge

In order to understand why StatsD presents a challenge you need to be familiar with the way Nameko handles dependency injection. In a nutshell, you write a dependency provider class, and you set an instance of it as a class attribute on your service. When Nameko starts a worker for the service, that class attribute is replaced with whatever comes out of the dependency provider's `get_dependency` method.

So, when you're writing your service class, you don't have access to the actual dependency, as it will be created for you when it's needed. Another thing to keep in mind is that at the time of writing the service class you don't have access to the config values.
 
So how can we decorate an entrypoint with a `StatsClient().timer()` if we don't have the client to start with? We can only create the client once we have access to the config values, and this happens when the dependency injection mechanism is running.


## Our solution

At [Sohonet](https://sohonet.com), we experimented with a couple of different approaches, and we decided to opensource the one we deemed most interesting. The result is a custom dependency provider called [nameko-statsd](https://github.com/sohonetlabs/nameko-statsd).

### A usage example

Let's start by seeing a quick example on how you use the library. You simply declare it on the service and then you can use it within any of the service methods (entrypoints, simple methods, etc.).

```python
from nameko_statsd import StatsD, ServiceBase

class Service(ServiceBase):

    statsd = StatsD('prod1')

    @entrypoint
    @statsd.timer('process_data')
    def process_data(self):
        ...

    @rpc
    def get_data(self):
        self.statsd.incr('get_data')
        ...

    def simple_method(self, value):
        self.statsd.gauge(value)
        ...
```


The `statsd.StatsClient` instance exposes a set of methods that you can access without having to go through the client itself.  The dependency acts as a pass-through for them.  They are: `incr`, `decr`, `gauge`, `set`, and `timing`.

In the above code example, you can see how we access ``incr`` and ``gauge``.

You can also decorate any method in the service with the ``timer`` decorator, as shown in the example.  This allows you to time any method without having to change its logic.


## Configuration

The library expects the following values to be in the config file you use for your service (you need one configuration block per different statsd server).  For example, if we had two statsd servers, prod1 and prod2, we would have something like this:

```yaml
STATSD:
  prod1:
    host: "host1"
    port: 8125
    prefix: "prefix-1"
    maxudpsize: 512
    enabled: true
  prod2:
    host: "host2"
    port: 8125
    prefix: "prefix-2"
    maxudpsize: 512
    enabled: false
```


The first four values of each block are passed directly to `statsd.StatsClient` on creation.  The last one, `enabled`, will activate/deactivate all stats, according to how it is set (`true`/`false`).  In this example, production 1 is enabled while production 2 is not.


## Minimum setup

In order to give the users of this library the ability to decorate methods with the `timer` decorator, we need to do a little wiring behind the scenes.  The only thing required for the end user is to write the service class so that it inherits from `nameko_statsd.ServiceBase`.

The *type* of `nameko_statsd.ServiceBase` is a custom metaclass that provides the necessary wirings to any `nameko_statsd.StatsD` dependency.

If, for any reason, you cannot inherit from `nameko_statsd.ServiceBase`, no problems, all you have to do is to make sure you pass a `name` argument to any `nameko_statsd.StatsD` dependency provider, the value of which has to match the attribute name of the attribute on the service class.

The following configuration:

```python
class MyService(ServiceBase):

    statsd = StatsD('prod1')

    ...
```

is equivalent to (notice it inherits from `object`):

```python
class MyService(object):

    statsd = StatsD('prod1', name='statsd')

    ...
```


## The `StatsD.timer` decorator

You can pass any arguments to the decorator, they will be given to the `statsd.StatsClient().timer` decorator.

So, for example:

```python
class MyService(ServiceBase):

    statsd = StatsD('prod1')

    @entrypoint
    @statsd.timer('my_stat', rate=5)
    def method(...):
        # method body

    @statsd.timer('another-stat')
    def another_method(...):
        # method body
```

is equivalent to the following:

```python
class MyService(ServiceBase):

    statsd = StatsD('prod1')

    @entrypoint
    def method(...):
        with self.statsd.client.timer('my_stat', rate=5):
            # method body

    def another_method(...):
        with self.statsd.client.timer('another-stat'):
            # method body
```

> When using `self.statsd.client.timer` as a context manager, you're bypassing the dependency, which means that the timer will be acted regardless of how the `enabled` setting is configured.


## About the lazy client

When you attach a `nameko_statsd.StatsD` dependency to your service, no client is created.  Only when you use the dependency explicitly or when you run a method that has been decorated with the `timer` decorator, a client is created.

This lazy feature means you can attach as many `nameko_statsd.StatsD` dependencies to your service as you fancy, and no client will be created unless it is actually used.


## Conclusion

Writing this code was fun. If you have any questions or suggestions on how to improve it, do get in touch!
