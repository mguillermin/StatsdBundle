# Usage


## Configure the servers

You have to configure the m6_statsd extension with the list of all the servers that will be available.

```yaml
m6_statsd:
    servers:
        default:
            address: 'udp://localhost'
            port:     1234
        serv1:
            address: 'udp://lolcaThost'
            port:     1235
        serv2:
            address: 'udp://lolcaThost'
            port:     1236
    clients:
        default:
            servers:   ["default"]        # the 'default' client will use only the default server
        swag:
            servers:   ["serv1", "serv2"] # the 'swag' client will use serv1 OR serv2 to send the datas
        mighty:
            servers: ["all"] # use all servers configured
        shell_patern:
            servers: ["s?rv*"] # use servers aliases matching given pattern, here "serv1" and "serv2"
```


You have to use the same graphite server behind statsd instance in order to see your stats.


## Basic usage

In a Symfony controller :

```php
// m6_statsd access to the default client by convention
$this->get('m6_statsd')->increment('a.graphite.node');
$this->get('m6_statsd')->timing('another.graphite.node', (float) $timing);
// access to the swag client
$this->get('m6_statsd.swag')->increment('huho');
```

By default, the send method of each clients is called on the kernel.terminate event. But you can call it manually in a controller :

```php
$this->get('m6_statsd')->send();
```

## Bind on events

### Increments

We don't really like mixing our business code with monitoring stuff. We prefer launch events with significant informations, listen to them, and send our monitoring stuffs in the listeners. The good news is that StastdBundle are doing it for you.

At each client level, you can specify events listened in order to build statsd increment or timing based on them.
For example, with the following configuration :

```yaml
m6_statsd:
    clients:
        default:
            events:
                forum.read:
                    increment : mysite.forum.read
```

On the Symfony event dispatcher, when the ```forum.read``` event is fired, our statds client catch this event and add to this queue the increment on the ```mysite.forum.read``` node.

So you can now just fire the event from a controller :
```php
$this->get('event_dispatcher')->dispatch('forum.read', new Symfony\Component\EventDispatcher\Event());
```

It's also possible to create tokens in the Symfony configuration, allowing you to pass custom value in the node.

The resolution of the token will be based on a method or a propertie of the event given.


```yaml
m6_statsd:
    clients:
        default:
            events:
                forum.read:
                    increment : mysite.forum.<name>.read
```

The event dispatched must have a getName method implemented or a $name public propertie.

### Count, Set and Gauge
You can also send count, set and gauge using this configuration:
```yaml
m6_statsd:
    clients:
        default:
            events:
                forum.read:
                    count: mysite.forum.read
                memory.used:
                    set: mysite.memory
                gauge.event:
                    gauge: mysite.custom_gauge
```

The sent event must implements a ```getValue``` method


### Timers

```yaml
m6_statsd:
    clients:
        default:
            events:
                action.longaction:
                    timing : timer.mysite.action
```

In this case, we will add a timer on timer.mysite.action node (of course you can still use this notation : `timer.<site>.action`). The timer value will be the output of the `getTiming` method of the event.

```yaml
m6_statsd:
    clients:
        default:
            events:
                action.longaction:
                    custom_timing : { node : timer.mysite.action, method : getRaoul }
```

The `custom_timing` allow you to set a custom method to collect the timer (here `getRaoul`).

**You can add multiple timing and increments under an event**

```yaml
m6_statsd:
    clients:
        default:
            events:
                action.longaction:
                    custom_timing : { node : timer.mysite.action, method : getRaoul }
                    timing : timer.mysite.action
                    timing : timer.mysite.action2
                    increment : mysite.forum.<name>.read
                    increment : mysite.forum.<name>.read2
                    # ...
```

### Generic Event
You can use the [StatsdEvent](https://github.com/M6Web/StatsdBundle/blob/master/src/Statsd/StatsdEvent.php) class to trigger your basic events.

```php
$this->get('event_dispatcher')
    ->dispatch('forum.read', new M6Web\Bundle\StatsdBundle\Statsd\StatsdEvent($valueOrTiming));
```


### Immediate send

In basic usage, the data is really sent to the StatsD servers during the `kernel.terminate` event. But if you want to use this bundle in commands, you may want to send data immediately.

```yaml
m6_statsd:
    clients:
        default:
            events:
                console.exception:
                    increment: mysite.command.<command.name>.exception
                    immediate_send: true
```

## Console custom events

This bundle can trigger custom console events that allow to get command start time and execution duration.

Use the following configuration to enable these events:

```yml
m6_statsd:
    console_events: true
```

Now, each time a command start, ends or throw ecxeption, one of the following events is triggered:
* `m6web.console.command`
* `m6web.console.terminate`
* `m6web.console.exception`

For instance, if you want to monitor the execution duration or your commands, use the following configuration:

```yml
m6_statsd:
    console_events: true
    clients:
        default:
            events:
                m6web.console.terminate:
                    custom_timing :
                        node : timer.mysite.command.<underscoredCommandName>.duration
                        method : getExecutionTime
```

As you can see in the previous exemple, the event object also provide a `getUnderscoredCommandName`. This method return the command name with colons replaced by underscores. This can be usefull because statsd use colon as separator.

These events also provide a `getTiming` method (which is an alias of `getExecutionTime`) that allow to use simple `timer` entry.

## Collect basics metrics on your Symfony application

Basics metrics can be http code, memory consumption, execution time. Thoses metrics can be collected when the `kernel.terminate` event.

Some basic collectors are already implemented in the bundle, but not activated by default.

To activate them, you have to set the ```base_collectors``` option to ```true```:
```yaml
m6_statsd:
    servers:
        # ...
    base_collectors: true
```

Those collectors just send events. You have to catch them as explained previously:
```yaml
m6_statsd:
    servers:
        # ...
    base_collectors: true
    clients:
        default:
            servers: ['default']
            events:
                statsd.memory_usage:
                    gauge: "website.memory"
                statsd.time:
                    timing: "website.time"
                statsd.exception:
                    increment: "website.exception.<value>"

                kernel.terminate: # this event is a symfony basic, you just have to listen to it to have the number of page view
                    increment: "website.page_view"
```

For now, those events are trigger:
* statsd.memory_usage
* statsd.time
* statsd.exception

## DATA collector

TODO


## Using the component only

use directly [the component on packagist](https://packagist.org/packages/m6web/statsd).

(if you working with Zend Framework or whatever !)

```php
use \M6Web\Component\Statsd;
$statsd = new Statsd\Client(
    array(
        'serv1' => array('address' => 'udp://xx.xx.xx.xx', 'port' => 'xx'),
        'serv2' => array('address' => 'udp://xx.xx.xx.xx', 'port' => 'xx'))
);
$statsd->increment('service.coucougraphite');
// we can also pass a sampling rate, default value is 1
$statsd->decrement('service.coucougraphite')->increment('service.test')->timing('service.timeismoney', 0.2);
// ..
$statsd->send();
```

[TOC](../README.md)
