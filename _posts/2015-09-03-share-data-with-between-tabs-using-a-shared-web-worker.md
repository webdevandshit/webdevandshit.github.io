---
layout: post
title: Sharing data between tabs using a shared web worker
---
#### The problem
Single page apps are dominating the browser world. Thicker client means more responsibility
which also leads to more complexity. SPA's are long lived entities, therefore having multiple tabs open
of the same app can actually lead to stale state.

For the purpose of this post we will examine the use case of a client having to interact with the server providing a CSRF
token. A CSRF token is one-time short-lived token (nonce). When the client interacts with the server it has to always
provide the latest issued token and also capture the next token. 

In an angular context this logic could be easily implemented with an [interceptor](https://docs.angularjs.org/api/ng/service/$http) factory; Every request can be decorated
with the token and every response can be intercepted in order to extract the newly issued token and set into a service.
I will not explain on this post why someone would use a CSRF token and what exactly it is; if you are interested, read more [here](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet).

![alt text](/img/CSRF.png "CSRF diagram")

#### The scenario
Imagine a user having two tabs open of the same single page application. The tab that interacted last with the server
is the one which obtained the latest CSRF token. This subsequently means that if the user attempts to use another tab,
other than the one that has the latest CSRF token, they will get an error.

The question is how do we communicate this piece of information between tabs without using a persistence layer
(db, cookies, localstorage). Shared web worker FTW! We will use a mediator to notify all our listeners
except the publisher (if the publisher is not excluded from the "to notify" peers everything the application will go in an infinite loop) whenever this token has changed.

![alt text](/img/flow.png "Fucking flow diagram")

Note that a shared web worker can be accessed only by the tabs that are under the same domain. Also, everything passed
to a web worker is passed by value not by reference. There are ways of passing data by reference using ArrayBuffers but that's not
the purpose of this blog post.

#### The solution

The solution below assumes that you use browserify, gulp and ngInject to pack your code. The code is
organised in a component-like structure:

1. Entry point of the component (index.js)
2. Run block, Angularjs initialization block (run.js)
3. Actual factory (workerFactory.js)
4. The actual worker code (sharedWorker.js)
5. Example registering an Event
6. Example publishing a message

#### index.js
{% highlight js %}
'use strict';

var angular = require('angular');

module.exports = angular
    .module('share',[])
    .run(require('./run'))
    .factory('Worker',require('./workerFactory'));
    
{% endhighlight %}

#### run.js
{% highlight js %}

'use strict';

/**
 * @description
 * If there is no worker already instantiated, start the worker
 * @ngInject
 * @param shareWorker
 */
function run(Worker) {
    if (!Worker.hasWorker()) {
        shareWorker
            .start();
    }
}

module.exports = run;
    
{% endhighlight %}


#### workerFactory.js
{% highlight js %}

'use strict';

/**
 * @ngInject
 * @returns {{hasWorker: *}}
 */
function workerFactory($q) {
    /**
     * @description
     * Hold the worker instance
     */
    var worker;
    /**
     * @description
     * Hold the connectionId for the single page app. 
     * Each tab essentially is an instance of the whole single page app
     */
    var connectionId;
    /**
     * @description
     * This array holds all the events that need to be triggered when the worker
     * publishes a new message
     * @type {Array}
     */
    var events = [];
    /**
     * @description
     * The connection event will set the instance if the event is a type of instance
     */
    function connectionEvent(e, instance) {
        if (e.data.type === 'CONNECTION') {
            worker = instance;
            connectionId = e.data.connectionId;
        }
    }
    /**
     * @description
     * Checks if there is a worker and a connection to it
     */
    function hasWorker() {
        return (worker &&  connectionId && connectionId > 0);
    }
    /**
     * @description
     * Helper method that filters the events by type.
     */
    function filterByType(type) {
        return function(event) {
            return event.type === type;
        };
    }
    /**
     * @description
     * Helper method that trigers for a given even their action passing also to the action the event context
     */
    function triggerAction(e) {
        return function(event) {
            event.action(e);
        };
    }
    /**
     * @description
     * Creates a shared worker
     */
    function createWorker(promise) {
        return function() {
            var workerInstance = new SharedWorker('/workers/sharedWorker.js','shared');
            workerInstance.port.addEventListener('message', function(e) {
                connectionEvent(e, workerInstance);
                events
                    .filter(filterByType(e.data.type))
                    .forEach(triggerAction(e));
                promise.resolve();
            }, false);
            workerInstance.port.start();
        };
    }
    /**
     * @description
     * Starts a worker and returns a promise
     */
    function start() {
        var dfd = $q.defer();
        isSupported()
            .then(createWorker(dfd),notSupported(dfd));
        return dfd.promise;
    }
    /**
     * @description 
     * Helper method that returns a rejected promise
     */
    function notSupported(promise) {
        return function() {
            promise.reject();
        };
    }
    /**
     * @description
     * Checks if the browser supports web workers and returns a promise
     */
    function isSupported() {
        var dfd = $q.defer();
        if (window.SharedWorker) {
            dfd.resolve();
        } else {
            dfd.reject();
        }
        return dfd.promise;
    }
    /**
     * @description
     * Register an event callback
     */
    function addEvent(event) {
        events.push(event);
    }
    /**
     * @description
     * Publish a message to the worker passing the actuall message and its type
     */
    function send(message, type) {
        if (hasWorker()) {
            worker.port.postMessage({
                port: connectionId,
                msg: message,
                type: type
            });
        }
    }
    return {
        hasWorker: hasWorker,
        start: start,
        send: send,
        addEvent: addEvent
    };
}

module.exports = workerFactory;
    
{% endhighlight %}

#### sharedWorker.js
{% highlight js%}

'use strict';

/**
 * @description
 * A counter that increases everytime a new connection to the worker is established
 */
var count = 0;
/**
 * @description
 * All the connection objects with their events
 */
var peers = [];
/**
 * @description
 * The event listener for the connection event.
 */
self.addEventListener('connect', function(e) {
    /**
     * @description
     * Everytime a connection is established the following is happening:
     * 1) Get the port object
     * 2) Increase the counter
     * 3) Register the new peer and his connection id
     * 4) Tell him that you registered him, by sending him a message of type connection alongside with his connectionId
     */
    var port = e.ports[0];
    count += 1;
    peers.push({
        connectionId: count,
        port: port
    });
    port.postMessage({
        connectionId: count,
        type: 'CONNECTION'
    });
    /**
     * @description
     * Everytime a new message comes in from a peer, all the other peers are notified with the message apart from the publisher
     */
    port.addEventListener('message', function(e) {
        peers.filter(function(peer) {
            return peer.connectionId !== e.data.port;
        }).forEach(function(peer) {
            peer.port.postMessage(e.data);
        });
    });
    port.start();
});
{% endhighlight%}

#### Example registering event during initialization of our Auth component
{% highlight js %}

'use strict';

function run(Auth, Worker){
    /**
     * @description
     * The event should have a type and an action. The action is a callback which will get passed the whole event object
     * @type {Object}
     */
    var CSRFEvent = {
        type: 'CSRF_TOKEN',
        action: function(e) {
            Auth.CSRFHeaders = e.data.msg;
        }
    };
    Worker.addEvent(CSRFEvent);
}

module.exports = run;

{% endhighlight %}

#### Example publishing a message
{% highlight js %}

'use strict';

/**
 * @description
 * Setter for CSRF header object
 */
function setCSRFHeaders(headers) {
    factory.CSRFHeaders = {};
    if (headers) {
        factory.CSRFHeaders = {
            name: headers['x-csrf-header'],
            value: headers['x-csrf-token']
        };
        if (shareWorker.hasWorker()) {
            shareWorker.send(factory.CSRFHeaders,'CSRF_TOKEN');
        }
    }
};

{% endhighlight %}
#### Tip
This could also be used to keep other kind of state in sync, among tabs. Logging in or logging out, for instance. Logging out in one tab should log you out in all other tabs. Sharing a socket connection rather than all the tabs establishing their own! Sharing a cache layer... The applications are endless.

#### Browser support
Browser support is [not broad](http://caniuse.com/#feat=sharedworkers)

#### Debugging
To debug a shared web worker, visit chrome://inspect/#workers with your chrome browser.

#### Disclaimer
This code is far from perfect and could be massively improved and extended. Everything you read take with a pinch of salt. Don't go and implement it on your employer's codebase without doing research, proving the concept, benchmarking it and writing a good test harness.