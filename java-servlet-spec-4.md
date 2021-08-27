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

```
ServletRequest 

    AsyncContext startAsync() throws IllegalStateException;
```

To start async processing, in **ServletRequest**, we call **AsyncContext startAsync(...)** to put the requests in asynchrounous mode, which initializes the `AsyncContext` for it. Calling this method also make sure that ***when the `Servlet#service` method is finished, the response isn't committed immediately*** (since the processing may be delegated to another thread. When the `service` method returns, it doesn't mean the processing is actually finished). 

### 2.4.2 Complete Async

```
AsyncContext
    
    void complete();
```

Only when the **AsyncContext#complete()** is called or the `AsyncContext` is **timed out**, the response is committed. 

### 2.4.3 Async Support

```
ServletRequest 

    boolean isAsyncSupported();
```

We can also check if request supports async processing through **ServletRequest#isAsyncSupported()**. Async support for a servlet request is disabled when one of the filter passed in or the servlet doesn't support async processing.

### 2.4.4 Check Is Async Started

```
ServletRequest 

    boolean isAsyncStarted();
```

**ServletRequest#isAsyncStarted()** tells whether the async processing for given request is started or not. If the request is dispatched (e.g., to another path) using **AsyncContext.dispatch** method, or the **AsyncContext#complete** is called (meaning the async processing is finished), this method returns false.

### 2.4.5 Request Dispatcher Type

```
ServletRequest 

    DispatcherType getDispatcherType();
```

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

# 3. Chap 3 The Request

## 3.1 HTTP Protocol Parameters

Request parameters are strings sent to **Servlet Container** as part of the request, when they are 'available', the container extracts these parameters from URI query strings (query parameters) and POST data, and store them in the request in forms of key-value map. There can be multiple values for the same parameter name. However, the **Path Parameters** for **GET** requests are not exposed through these APIs, they can only be retrieved by parsing **HttpServletRequest#getRequestURI** or **HttpServletRequest#getPathInfo** methods.

In **ServletRequest**, there are methods provided to access to these prameters:

- String getParameter(String);
- Enumeration<String> getParameterNames();
- String[] getParameterValues(String name);
- Map<String, String[]> getParameterMap();

## 3.2 File Upload

Files are uploaded when the request is **multipart/form-data**. When `mutipart/form-data` request is supported by the container, the data can be accessed through following methods on HttpServletRequest:

HttpServletRequest:

- Collection<Part> getParts() throws IOException, ServletException;
- Part getPart(String name) throws IOException, ServletException; 

Each **Part** provides access to the headers, content type and input stream for it. If the servlet container doesn't support multipart/form-data processing, the data will be accessible through **HttpServletRequest.getInputStream()**.

## 3.3 Attributes

Attributes of a request (ServletRequest) are extra information set for communication between components (e.g., between Servlets). The servlet container may set attributes to a request to make available some information about a client. One key corresponds to one single value, they can be set or retriveved from a request through following methods on ServletRequest:

ServletRequest:

- void setAttribute(String name, Object o);
- void removeAttribute(String name);
- Object getAttribute(String name);

## 3.4 Headers

Headers of HTTP request can be retrieved by follwing methods on **HttpServletRequest**. There can be multiple headers for the same name, `getHeader(String)` returns the first header, while `getHeaders(String)` return all header values. Convenient methods include `getIntHeader(String)` and `getDateHeader(String)`.

HttpServletRequest:

- String getHeader(String name);
- Enumeration<String> getHeaders(String name);
- Enumeration<String> getHeaderNames();

## 3.5 Request Path Elements

The request path consists of multiple sections:

- Context Path
    - The path prefix associated with the **ServletContext**, if the context is the default context rooted at the base of the web server's name space, this path will be empty string, otherwise, it will start with '/' but not end with '/'.
- Servlet Path
    - The path that is used to match servlet(s), this path will start with '/', unless the context path is '/' or '' empty string.
- PathInfo
    - The section that is not part of context or servlet path, think of it as an extra path. If there is no extra path, it's simply null, otherwise, it starts with '/'. 

```
requestURI = contextPath + servletPath + pathInfo
```

These three sections can be accessed through:

HttpServletRequest:

- String getContextPath();
- String getServletPath();
- String getPathInfo();

## 3.6 Path Translation Method

Two methods are provided to translate given path to a file system path.

- ServletContext.getRealPath(String);
- HttpServletRequest.getPathTranslate();

## 3.7 Non Blocking I/O

**ServletInputStream** and **ServletOutputStream** register **ReadListener** and the **WriteListener** for non blocking I/O, they react to the callback invocation by reading from the `ServletInputStream` or writing to the `ServletOutputStream`. 

For `ServletInputStream`, the registered listener is **ReadListener**, its callback methods are invoked by the container: 

1. when there are data available (for which the implementation will consume the incoming data, e.g., a byte buffer)  
2. when all data has been consumed by the consumer (i.e., we are done here); 
3. when there are some sort of I/O related problems occurred. 

This works in a reactive fasion, so reactive programming like RxJava is normally used, e.g., when `onDataAvailable()` is called by the container, the implementation of **ReadListener** consumes/read data from the inputStream in forms of a `ByteBuffer`, then it publishes this `ByteBuffer` to its subscriber.

ReadListener:

- void onDataAvailable() throws IOException;
- void onAllDataRead() throws IOException;
- void onError(Throwable t);

WriteListener:

- void onWritePossible() throws IOException;
- void onError(final Throwable t);

## 3.8 HTTP/2 Server Push

**Server push** is for performance improvement. When a client requests a specific resource (say *A*), server may know in advance that the client will be requested resource *B*, *C* and *D* following the initial request, in this case, the server may use **Server Push** to push the bytes of resource *B*, *C*, and *D* right after the request for *A*. 

The server push is used by creating a **PushBuilder** (through calling **HttpServletRequest.newPushBuilder()**), then customize the `PushBuilder` and finally call **PushBuilder.push()** to push the resource to client. 

## 3.9 Cookies

All Cookies presented on a request can be retrieved through **HttpServletRequest.getCookies()**, `Cookie` can be set to **HttpOnly** that asks the browser not to expose cookie values to scripts.

## 3.10 SSL Attributes

When a request is tranmitted over a secure protocol like HTTPS, the **ServletRequest.isSecure()** will return true. The web container will also exposes following attributes that can be retrieved from `ServletRequest`.

Attribute|Attribute Name (key)|Java Type|
---|---|---|
cipher suite|javax.servlet.request.cipher_suite|String|
bit size of algorithm|javax.servlet.request.key_size|Integer|
SSL session id|javax.servlet.request.ssl_session_id|String|
SSL certificate|java.security.cert.X509Certificate|X509Certificate|

E.g.,

In tomcat, it put a SSL attribute if found:

```
public static final String CERTIFICATES_ATTR = "javax.servlet.request.X509Certificate";

// ...

attr = coyoteRequest.getAttribute(Globals.CERTIFICATES_ATTR);
if (attr != null) {
    attributes.put(Globals.CERTIFICATES_ATTR, attr);
}

// ...
```

Then in Spring, the framework writes a `Filter` to retrive the certificate from attribute

```
//...

private X509Certificate extractClientCertificate(HttpServletRequest request) {
    X509Certificate[] certs = 
        (X509Certificate[]) request.getAttribute("javax.servlet.request.X509Certificate");
    if (certs != null && certs.length > 0) {
        return certs[0];
    }
    return null;
}

// ...
```

## 3.11 Internationization

Clients may tell the server which language they prefer, this information is communciated using header **Accept-Language**. This information can be retrieved by following methods in **ServletRequest**. 

ServletRequest:

- Locale getLocale();
- Enumeration<Locale> getLocales();

## 3.12 Request Data Encoding

Data encoding is described through the header **Content-Type**. 

## 3.13 Lifecycle of Request Object

***"Each request object is valid only within the scope of a servlet’s service method, or within the scope of a filter’s doFilter method, unless the asynchronous processing is enabled for the component and the startAsync method is invoked on the request object. In the case where asynchronous processing occurs, the request object remains valid until complete is invoked on the AsyncContext."***

# 4. Chap 4 Servlet Context





