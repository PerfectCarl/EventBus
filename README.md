EventBus
========
EventBus is an Android optimized publish/subscribe event bus. 

An event bus eases the communication between Activities, Fragments, and background threads without introducing complex and error-prone dependencies and life cycle issues. 

EventBus propagates event to all participants (e.g. background service -> activity -> multiple fragments or helper classes).

EventBus decouples event senders and receivers and simplifies event/data exchange between your applications components. 

(And you don't need to implement a single interface!)

General usage and API
---------------------

**1- Define your event class as a POJO**
```java
public class MessageEvent {
    private String message;

    public MessageEvent(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}
```
**2- The suscribers registers the eventbus**

The **subscribers** implement event handling `onEvent` methods that will be called when an event is received. They also need to register themselves to the bus. 

```java
    @Override
    public void onStart() {
        super.onStart();
        // Registring the bus for MessageEvent
        EventBus.getDefault().register(this);
    }

    @Override
    public void onStop() {
        // Unregistering the bus
        EventBus.getDefault().unregister(this);
        super.onStop();
    }
    
    // This method will be called when a MessageEvent is posted
    public void onEvent(MessageEvent event){
        Toast.makeText(getActivity(), event.getMessage().toString(), Toast.LENGTH_SHORT).show();
    }
    
    // In case many events are suscribed, just add another method with the event type
    // This method will be called when a SomeOtherMessageEvent is posted
    public void onEvent(SomeOtherMessageEvent event){
        Toast.makeText(getActivity(), event.getMessage().toString(), Toast.LENGTH_SHORT).show();
    }
    
```
**3- Post your event from any part of your code**
```java
    EventBus.getDefault().post(new MessageEvent("hello!"));
```
Add EventBus to your project
----------------------------
EventBus is available on [Maven Central](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22de.greenrobot%22%20AND%20a%3A%22eventbus%22)
Gradle template ([check current version](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22de.greenrobot%22%20AND%20a%3A%22eventbus%22)):
```
dependencies {
    compile 'de.greenrobot:eventbus:2.2.1'
}
```
Maven template ([check current version](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22de.greenrobot%22%20AND%20a%3A%22eventbus%22)):
```
<dependency>
    <groupId>de.greenrobot</groupId>
    <artifactId>eventbus</artifactId>
    <version>2.2.1</version>
</dependency>
```
Ivy template ([check current version](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22de.greenrobot%22%20AND%20a%3A%22eventbus%22)):
```
<dependency
    name="eventbus"
    org="de.greenrobot"
    rev="2.2.1" />
```
[Download from maven](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22de.greenrobot%22%20AND%20a%3A%22eventbus%22)
Delivery threads and ThreadModes
--------------------------------
EventBus can handle threading for you: events can be posted in threads different from the posting thread. 

A common use case is dealing with UI changes. As you may know, UI changes must be done in the UI thread and networking (or other time consumming task) must be done in other treads. EventBus will help you to deal with those tasks and  synchronize with the UI thread (without having to delve into thread transistions, using AsyncTask, etc). 

In EventBus, you may define the thread that will call the event handling method `onEvent` by using a **ThreadMode**:
* **PostThread:** Subscriber will be called in the same thread, which is posting the event. This is the default. Event delivery implies the least overhead because it avoids thread switching completely. Thus this is the recommended mode for simple tasks that are known to complete is a very short time without requiring the main thread. Event handlers using this mode must return quickly to avoid blocking the posting thread, which may be the main thread.
This corresponds to this code:
```java
    // Called in the same thread (default)
    public void onEvent(MessageEvent event){
        Toast.makeText(getActivity(), event.getMessage().toString(), Toast.LENGTH_SHORT).show();
    }
```
* **MainThread:** Subscriber will be called in Android's main thread (sometimes referred to as UI thread). If the posting thread is the main thread, event handler methods will be called directly. Event handlers using this mode must return quickly to avoid blocking the main thread.
This corresponds to this code:
```java
    // Called in Android UI's main thread
    public void onEventMainThread(MessageEvent event){
        Toast.makeText(getActivity(), event.getMessage().toString(), Toast.LENGTH_SHORT).show();
    }
```
* **BackgroundThread:** Subscriber will be called in a background thread. If posting thread is not the main thread, event handler methods will be called directly in the posting thread. If the posting thread is the main thread, EventBus uses a single background thread that will deliver all its events sequentially. Event handlers using this mode should try to return quickly to avoid blocking the background thread.
```java
    // Called in the background thread
    public void onEventBackgroundThread(MessageEvent event){
        Toast.makeText(getActivity(), event.getMessage().toString(), Toast.LENGTH_SHORT).show();
    }
```
* **Async:** Event handler methods are called in a separate thread. This is always independent from the posting thread and the main thread. Posting events never wait for event handler methods using this mode. Event handler methods should use this mode if their execution might take some time, e.g. for network access. Avoid triggering a large number of long running asynchronous handler methods at the same time to limit the number of concurrent threads. EventBus uses a thread pool to efficiently reuse threads from completed asynchronous event handler notifications.
```java
    // Called in a separate thread (default)
    public void onEventAsync(MessageEvent event){
        Toast.makeText(getActivity(), event.getMessage().toString(), Toast.LENGTH_SHORT).show();
    }
```

*Note:* EventBus takes care of calling the `onEvent` method in the proper thread depending on its name (onEvent, onEventAsync, etc.).

Subscriber priorities and ordered event delivery
------------------------------------------------
You may change the order of event delivery by provinding a priority to the suscriber when registering your event bus.

```java
    @Override
    public void onStart() {
        super.onStart();
        // Registring the bus
        int priority = 1 ; 
        EventBus.getDefault().register(this, priority);
    }
```

Within the same delivery thread (ThreadMode), higher priority subscribers will receive events before others with a lower priority. The default priority is 0. 

*Note*: the priority does *NOT* affect the order of delivery among subscribers with different [ThreadModes](#delivery-threads-and-threadmodes)!

Cancelling event delivery
-------------------------
You may cancel the event delivery process by calling `cancelEventDelivery(Object event)` from a subscriber's event handling method. 
Any further event delivery will be cancelled: subsequent subscribers won't receive the event.
```java
    // Called in the same thread (default)
    public void onEvent(MessageEvent event){
    	// Process the event 
    	...
    	
    	EventBus.getDefault().cancelEventDelivery(event) ;
    }
```

Events are usually cancelled by higher priority subscribers. Cancelling is restricted to event handling methods running in posting thread [ThreadMode.PostThread](#delivery-threads-and-threadmodes).

Sticky Events
-------------
Some events carry information that is of interest after the event is posted. For example, this could be an event signalizing that some initialization is complete. Or if you have some sensor or location data and you want to hold on the most recent values. Instead of implementing your own caching, you can use sticky events. EventBus keeps the last sticky event of a certain type in memory. The sticky event can be delivered to subscribers or queried explicitly. Thus, you don't need any special logic to consider already available data.

You register your event bus with specific methods: 
```java
    @Override
    public void onStart() {
        super.onStart();
        // Registring the bus for MessageEvent
        EventBus.getDefault().registerSticky(this, MessageEvent.class);
    }

    @Override
    public void onStop() {
        // Unregistering the bus
        EventBus.getDefault().removeStickyEvent(MessageEvent.class);
        super.onStop();
    }
```
And then you post the event:
```java
    EventBus.getDefault().postSticky(new MessageEvent("hello!"));
```

You may get the last sticky event of a certain type with:  
```java
    EventBus.getDefault().getStickyEvent(Class<?> eventType)
```

Additional Features and Notes
-----------------------------

* **NOT based on annotations:** Querying annotations are slow on Android, especially before Android 4.0. Have a look at this [Android bug report](http://code.google.com/p/android/issues/detail?id=7811)
* **Based on conventions:** Event handling methods are called "onEvent" (you could supply different names, but this is not encouraged).
* **Performanced optimized:** It's probably the fastest event bus for Android.
* **Tiny:** The jar is less than 50 KBytes.
* **Convenience singleton:** You can get a process wide event bus instance by calling EventBus.getDefault(). You can still call new EventBus() to create any number of local busses.
* **Subscriber and event inheritance:** Event handler methods may be defined in super classes, and events are posted to handlers of the event's super classes including any implemented interfaces. For example, subscriber may register to events of the type Object to receive all events posted on the event bus.
* **Selective registration:** It's possible to register only for specific event types. This also allows subscribers to register only some of their event handling methods for main thread event delivery.

Comparison with Square's Otto
-----------------------------
Otto is another event bus library for Android; actually it's a fork of Guava's EventBus. greenrobot's EventBus and Otto share some basic semantics (register, post, unregister, ...), but there are differences which the following table summarizes:
<table>
    <tr>
        <th></th>
        <th>EventBus</th>
        <th>Otto</th>
    </tr>
    <tr>
        <th>Declare event handling methods</th>
        <td>Name conventions</td>
        <td>Annotations</td>
    </tr>	
    <tr>
        <th>Event inheritance</th>
        <td>Yes</td>
        <td>Yes</td>
    </tr>	
    <tr>
        <th>Subscriber inheritance</th>
        <td>Yes</td>
        <td>No</td>
    </tr>
    <tr>
        <th>Cache most recent events</th>
        <td>Yes, sticky events</td>
        <td>No</td>
    </tr>
    <tr>
        <th>Event producers (e.g. for coding cached events)</th>
        <td>No</td>
        <td>Yes</td>
    </tr>
    <tr>
        <th>Event delivery in posting thread</th>
        <td>Yes (Default)</td>
        <td>Yes</td>
    </tr>	
    <tr>
        <th>Event delivery in main thread</th>
        <td>Yes</td>
        <td>No</td>
    </tr>	
    <tr>
        <th>Event delivery in background thread</th>
        <td>Yes</td>
        <td>No</td>
    </tr>	
    <tr>
        <th>Aynchronous event delivery</th>
        <td>Yes</td>
        <td>No</td>
    </tr>
</table>

Besides features, performance is another differentiator. To compare performance, we created an Android application, which is also part of this repository (EventBusPerformance). You can also run the app on your phone to benchmark different scenarios.

Benchmark results indicate that EventBus is significantly faster in almost every scenario:
<table>
    <tr>
        <th></th>
        <th>EventBus</th>
        <th>Otto</th>
    </tr>
    <tr>
        <th>Posting 1000 events, Android 2.3 emulator</th>
        <td>~70% faster</td>
        <td></td>
    </tr>
	<tr>
        <th>Posting 1000 events, S3 Android 4.0</th>
        <td>~110% faster</td>
        <td></td>
    </tr>
    <tr>
        <th>Register 1000 subscribers, Android 2.3 emulator</th>
        <td>~10% faster</td>
        <td></td>
    </tr>
    <tr>
        <th>Register 1000 subscribers, S3 Android 4.0</th>
        <td>~70% faster</td>
        <td></td>
    </tr>
    <tr>
        <th>Register subscribers cold start, Android 2.3 emulator</th>
        <td>~350% faster</td>
        <td></td>
    </tr>	
    <tr>
        <th>Register subscribers cold start, S3 Android 4.0</th>
        <td colspan="2">About the same</td>
    </tr>	
</table>

ProGuard configuration
----------------------
ProGuard obfuscates method names. However, the onEvent methods must not renamed because they are accessed using reflection. Use the following snip in your ProGuard configuration file (proguard.cfg):
<pre><code>-keepclassmembers class ** {
    public void onEvent*(**);
}
</code></pre>

Example
-------
Thanks to *@kevintanhongann*, a sample application is [available](https://github.com/kevintanhongann/EventBusSample)


FAQ
---
**Q:** How's EventBus different to Android's BroadcastReceiver/Intent system?<br/>
**A:** Unlike Android's BroadcastReceiver/Intent system, EventBus uses standard Java classes as events and offers a more convenient API. EventBus is intended for a lot more uses cases where you wouldn't want to go through the hassle of setting up Intents, preparing Intent extras, implementing broadcast receivers, and extracting Intent extras again. Also, EventBus comes with a much lower overhead. 

[Release History](CHANGELOG.md)
---------------

License
-------
Copyright (C) 2012-2014 Markus Junginger, greenrobot (http://greenrobot.de)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
