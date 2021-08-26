# Java Servlet Spec 4.0 Notes

Link: https://javaee.github.io/servlet-spec/downloads/servlet-4.0/servlet-4_0_FINAL.pdf

# 1. Chap 1 Overview

Relation between `web server`, `servlet container` and `servlet`:

```
                             +----------------------+
+------------+               | Servlet Container    |               +-----------------+
|            |  web request  |                      |    matches    |                 |
| Web Server +-------------->|                      +-------------->| Matched Servlet |
|            |               |  +-----------+       |               |                 |
|            |<--------------+  |           |       |<--------------+                 |
+------------+  web response |  | servlet 1 |       |   processes   +-----------------+
                             |  |           |       |
                             |  +-----------+       |
                             |                      |
                             |  +-----------+       |
                             |  |           |       |
                             |  | servlet 2 |       |
                             |  |           |       |
                             |  +-----------+       |
                             |    ....              |
                             |                      |
                             +----------------------+
```

# 2. Chap 2 Servlet Interface

## 2.1 Servlet, Generic Servlet and HttpServlet 

In Java Servlet APIs, two classes implements **Servletx**, they are **GenericServlet** and **HttpServlet**.

```
                   +-----------+
                   |           |
                   |  Servlet  |
                   |           |
                   +--^-----^--+
                      |     |
                      |     |
                      |     |
                 +----+     +------+
                 |                 |
+----------------+--+           +--+-------------+
|                   |           |                |
|  Generic Servlet  |           |  HttplServlet  |
|                   |           |                |
+-------------------+           +----------------+
```

`Servlet` interface defines **service** method for handling requests, this method will handle incoming requests from clients concurrently. **HttpServlet** abstract class, that implements `Servlet` interface's `service` method, wherein, it redirects the requests to HTTP-based request methods, such as **doGet**, **doPost**, **doPut**, **doDelete**. 

More specifically:

- doGet for GET
- doPost for POST
- doPut for PUT
- doDelete for DELETE
- doHead for HEAD
- doOptions for OPTIONS
- doTrace for TRACE

## 2.2 Number of Instances

Servlet container will create only one instance per serlvet declaration (i.e., one instance per JVM) unless which implements **SingleThreadModel** interface, in which case there may be more than one instance. Note that `SingleThreadModel` interface is deprecated.

## 2.3 Servlet Life Cycle

The life cycle of a `Servlet` is expressed in the `Servlet` interface through **init**, **service** and **destroy** methods. The Servlet Container is responsible for instantiating (through normal class loading mechanism) and initializsing each `Servlet` instance before it's active and consumes incoming requests. 

The **init** method is called to initialize the `Servlet` instance, during the method call, the instance may throw **UnavailableException** or **ServletException**, in which case the `Servlet` instance is no longer active, and it's **destroy** method is not called because it's not successfully initialized. 

Once the `Servlet` instance is initialized, the Servlet Container will start to use it to handle requests, requests are represented by objects of type **ServletRequest**, and they are handled by **Servlet#service** method, the result is represented by **ServletResponse**. When the requests are HTTP requests, the corresponding types will be **HttpServletRequest** and **HttpServletResponse**.

During request handling (in `service` method), the `Servlet` instance may throw **ServletException** or **UnavailableException**. The Servlet Container may return **'404 NOT FOUND'** or **'503 SERVICE UNAVAILABLE'** when **UnavailableException** is caught, it may be permanent or temporary, and it's containers' choice to decide whether to differentiate them or not, for both cases, the container will call **destroy** method for the `Servlet` instance,  

## 2.4 Async Processing

Async processing in `Servlet` is through the use of **AsyncContext**. 

### 2.4.1 Start Async 

To start async processing, in **ServletRequest**, we call **AsyncContext startAsync(...)** to put the requests in asynchrounous mode, which initializes the `AsyncContext` for it. Calling this method also make sure that ***when the `Servlet#service` method is finished, the response isn't committed immediately*** (since the processing may be delegated to another thread. When the `service` method returns, it doesn't mean the processing is actually finished). 

### 2.4.2 Complete Async

Only when the **AsyncContext#complete()** is called or the `AsyncContext` is **timed out**, the response is committed. 

### 2.4.3 Async Support

We can also check if request supports async processing through **ServletRequest#isAsyncSupported()**. Async support for a servlet request is disabled when one of the filter passed in or the servlet doesn't support async processing.

### 2.4.4 Check Is Async Started

**ServletRequest#isAsyncStarted()** tells whether the async processing for given request is started or not. If the request is dispatched (e.g., to another path) using **AsyncContext.dispatch** method, or the **AsyncContext#complete** is called (meaning the async processing is finished), this method returns false.

### 2.4.5 Request Dispatcher Type

DispatcherType of a request can be checked using **ServletRequest#getDispatcherType()** method. ***"The dispatcher type of a request is used by the container to select the filters that need to be applied to the request. Only filters with the matching dispatcher type and url patterns will be applied."***  

DispatcherType has five values:
- REQUEST: initial dispatcher type of a request.
- FORWARD: dispatched using **RequestDispatcher.forward(...)** method, forward current request to another resource (e.g., forward request to another servlet).
- INCLUDE: dispatched using **RequestDispatcher.include(...)** method, which is server-side include, meaning the current response will include the another resource (e.g., the response of another servlet).
- ASYNC: dispatched using **AsyncContext.dispatch(...)** method.
- ERROR: dispatched to an error page handled by container.

### 2.4.6 AsyncContext

AsyncContext represents the execution context of an async operation. It's created by **ServletRequest.startAsync**. A few methods are explained below:

**AsyncContext**

- `ServletRequest getRequest()` : get the request used to initialize the AsyncContext
- `ServletResponse getResponse()` : get the response used to initialize the AsyncContext 
- `void setTimeout(milliseconds)` :  set the timeout for the async operation, by default it's 30000
- `long getTimeout()` : get timeout for the async operation
- `void addListener(AsyncListener)` :  register listener for events: **onComplete**, **onTimeout**, **onError**, **onStartAsync**.
- `void dispatch(String)` : dispatch the request and response to the given path.
- `void start(Runnable)` : asks the container to dispatch a thread to run the given Runnable
- `void complete()` : commit and close the result of async operation

### 2.4.7  AsyncListener

Listener for **AsyncEvent** occurred in **AsyncContext**.

- `void onComplete(AsyncEvent event)` 
- `void onTimeout(AsyncEvent event)`
- `void onError(AsyncEvent event)` 
- `void onStartAsync(AsyncEvent event)`

### 2.4.8 State Transition Diagram for Async Operation

Once request enters the servlet container, it's at **dispatching** state, the servlet container dispatches it (via **onFilter()** or **service()** methods) to the filters and servlets that match, once the request is dispatched, it enters the **dispatched** state. The request can later be dispatched again, in which case, it enters the **dispatching** state again.

```
                 +-------------+                     +------------+
    dispatch()   |             |                     |            |
---------------->| Dispatching +-------------------->| Dispatched |
                 |             | onFilter()/service()|            |
                 +-------------+                     +------------+
```

At **dispatched** state, the request start async mode by calling **startAsync()**, which then enters the **AsyncStarted** state, before calling the **complete()** method, this async operation is not commited yet, it remains at **AsyncWait** or **Completing** state, after calling **complete()**, it then enters **Completed** state.

```
                +------------+                +--------------+  
                |            |  startAsync()  |              |  
                | Dispatched +--------------->| AsyncStarted |  
                |            |                |              |  
                +----+-------+                +--+------+----+  
+-----------+        |                           |      |            
|           |        |                           |      |            
| AsyncWait |        |          return           |      |                
|           |<-------+---------------------------+      |  complete()
+---+-------+        |                                  |        
    |                |  return                          |        
    |                |                                  |        
    |                v                                  v        
    |             +-----------+                +------------+    
    | complete()  |           | return         |            |    
    +------------>| Completed |<---------------+ Completing |    
                  |           |                |            |    
                  +----+------+                +------------+    
                       |             
                       |onComplete() 
                       |             
                       v             
                      End
```

## 2.5 Thread Safety 

***"...implementations of the request and response objects are not guaranteed to be thread safe. This means that they should either only be used within the scope of the request handling thread or the application must ensure that access to the request and response objects are thread safe."***

## 2.6 Upgrade Processing

In HTTP/1.1, the header **Upgrade** is used to negotiate a communication protocol switching, it allows the client to specify the protocol that it wants to use, and the server may switch to it if this protocol is supported. When a **Servlet** receives an upgrade request, the servlet can then call **HttpServletRequest#upgrade(Class<? extends HttpUpgradeHandler>)** to triggers the upgrade process. 

Notice that the argument is a class of type **HttpUpgradeHandler**, this handler communicates with the **Servlet Container** via byte streams, Servlet Container doesn't have any knowledge about the upgraded protocol. 

When the `#upgrade` method is called, and the servlet has done processing the request, the servlet container knows that this connection should be handled by the appropriate **HttpUpgradeHandler**, so it calls the **HttpUpgradeHandler#init(WebConnection)** method for the handler to access to data streams, providing the **WebConnection** object, which internally contains **ServletInputStream** and **ServletOutputStream**. Once the upgrade is done, the **HttpUpgradeHandler#destroy** is called.

E.g, in tomcat, the upgrade for websocket is implemented in a Filter, the Filter starts with checking if the ServletRequest is a HTTP request and whether it contains the upgrade header for the websocket. If it does, it starts the negotiation process, and if it was succesful, it creates the `WsHttpUpgradeHandler` that implements `HttpUpgradeHandler` by calling the `HttpServletRequest#upgrade(...)` method. Then it stores the handler in the request (via `UpgradeToken`) for the general purpose components to later trigger the upgrade process (by calling `HttpUpgradeHandler#init(...)`). 

General idea:

```
IF is HTTP request
THEN
    IF has upgrade header
    THEN
        // do some negotiation 
        // ... 

        wsUpgradeHandler = request.upgrade(WsHttpUpgradeHandler.class)

        // do some pre-init things for the handler
        // store the handler in UpradeToken and trigger the upgrade later
        // ...
    FI
FI

// ... later, check if we are upgrading for the request
// if so, do the upgrade

IF has UpradeToken
THEN
    wsUpgradeHandler = upgradeToken.getHandler(...)
    wsUpgradeHandler.init(webConnection)
    wsUpgradeHandler.destroy()
FI
```
