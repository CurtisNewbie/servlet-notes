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

In Java Servlet APIs, two classes implements **Servlet**, they are **GenericServlet** and **HttpServlet**.

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
|  Generic Servlet  |           |  HttpServlet   |
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

The life cycle of a `Servlet` is expressed in the `Servlet` interface through **init**, **service** and **destroy** methods. The Servlet Container is responsible for instantiating (through normal class loading mechanism) and initializing each `Servlet` instance before it's active and consumes incoming requests. 

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
    
    void complete()
```

Only when the **AsyncContext#complete()** is called or the `AsyncContext` is **timed out**, the response is committed. 

### 2.4.3 Async Support

```
ServletRequest 

    boolean isAsyncSupported()
```

We can also check if request supports async processing through **ServletRequest#isAsyncSupported()**. Async support for a servlet request is disabled when one of the filter passed in or the servlet doesn't support async processing.

### 2.4.4 Check Is Async Started

```
ServletRequest 

    boolean isAsyncStarted()
```

**ServletRequest#isAsyncStarted()** tells whether the async processing for given request is started or not. If the request is dispatched (e.g., to another path) using **AsyncContext.dispatch** method, or the **AsyncContext#complete** is called (meaning the async processing is finished), this method returns false.

### 2.4.5 Request Dispatcher Type

```
ServletRequest 

    DispatcherType getDispatcherType()
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

At **dispatched** state, the request start async mode by calling **startAsync()**, which then enters the **AsyncStarted** state, before calling the **complete()** method, this async operation is not committed yet, it remains at **AsyncWait** or **Completing** state, after calling **complete()**, it then enters **Completed** state.

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

E.g, in tomcat, the upgrade for websocket is implemented in a Filter, the Filter starts with checking if the ServletRequest is a HTTP request and whether it contains the upgrade header for the websocket. If it does, it starts the negotiation process, and if it was successful, it creates the `WsHttpUpgradeHandler` that implements `HttpUpgradeHandler` by calling the `HttpServletRequest#upgrade(...)` method. Then it stores the handler in the request (via `UpgradeToken`) for the general purpose components to later trigger the upgrade process (by calling `HttpUpgradeHandler#init(...)`). 

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
        // store the handler in UpgradeToken and trigger the upgrade later
        // ...
    FI
FI

// ... later, check if we are upgrading for the request
// if so, do the upgrade

IF has UpgradeToken
THEN
    wsUpgradeHandler = upgradeToken.getHandler(...)
    wsUpgradeHandler.init(webConnection)
    wsUpgradeHandler.destroy()
FI
```

# 3. Chap 3 The Request

## 3.1 HTTP Protocol Parameters

Request parameters are strings sent to **Servlet Container** as part of the request, when they are 'available', the container extracts these parameters from URI query strings (query parameters) and POST data, and store them in the request in forms of key-value map. There can be multiple values for the same parameter name. However, the **Path Parameters** for **GET** requests are not exposed through these APIs, they can only be retrieved by parsing **HttpServletRequest#getRequestURI** or **HttpServletRequest#getPathInfo** methods.

In **ServletRequest**, there are methods provided to access to these parameters:

- String getParameter(String)
- Enumeration<String> getParameterNames()
- String[] getParameterValues(String name)
- Map<String, String[]> getParameterMap()

## 3.2 File Upload

Files are uploaded when the request is **multipart/form-data**. When `multipart/form-data` request is supported by the container, the data can be accessed through following methods on HttpServletRequest:

HttpServletRequest:

- Collection<Part> getParts() throws IOException, ServletException
- Part getPart(String name) throws IOException, ServletException 

Each **Part** provides access to the headers, content type and input stream for it. If the servlet container doesn't support multipart/form-data processing, the data will be accessible through **HttpServletRequest.getInputStream()**.

## 3.3 Attributes

Attributes of a request (ServletRequest) are extra information set for communication between components (e.g., between Servlets). The servlet container may set attributes to a request to make available some information about a client. One key corresponds to one single value, they can be set or retrieved from a request through following methods on ServletRequest:

ServletRequest:

- void setAttribute(String name, Object o)
- void removeAttribute(String name)
- Object getAttribute(String name)

## 3.4 Headers

Headers of HTTP request can be retrieved by following methods on **HttpServletRequest**. There can be multiple headers for the same name, `getHeader(String)` returns the first header, while `getHeaders(String)` return all header values. Convenient methods include `getIntHeader(String)` and `getDateHeader(String)`.

HttpServletRequest:

- String getHeader(String name)
- Enumeration<String> getHeaders(String name)
- Enumeration<String> getHeaderNames()

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

- String getContextPath()
- String getServletPath()
- String getPathInfo()

## 3.6 Path Translation Method

Two methods are provided to translate given path to a file system path.
- ServletContext.getRealPath(String)
- HttpServletRequest.getPathTranslate()

## 3.7 Non Blocking IO

**ServletInputStream** and **ServletOutputStream** register **ReadListener** and the **WriteListener** for non blocking I/O, they react to the callback invocation by reading from the `ServletInputStream` or writing to the `ServletOutputStream`. 

For `ServletInputStream`, the registered listener is **ReadListener**, its callback methods are invoked by the container: 

1. when there are data available (for which the implementation will consume the incoming data, e.g., a byte buffer)  
2. when all data has been consumed by the consumer (i.e., we are done here) 
3. when there are some sort of I/O related problems occurred. 

This works in a reactive fashion, so reactive programming like RxJava is normally used, e.g., when `onDataAvailable()` is called by the container, the implementation of **ReadListener** consumes/read data from the inputStream in forms of a `ByteBuffer`, then it publishes this `ByteBuffer` to its subscriber.

ReadListener:

- void onDataAvailable() throws IOException
- void onAllDataRead() throws IOException
- void onError(Throwable t)

WriteListener:

- void onWritePossible() throws IOException
- void onError(final Throwable t)

## 3.8 HTTP/2 Server Push

**Server push** is for performance improvement. When a client requests a specific resource (say *A*), server may know in advance that the client will be requesting resource *B*, *C* and *D* following the initial request, in this case, the server may use **Server Push** to push the bytes of resource *B*, *C*, and *D* right after the request for *A* in parallel. 

The server push is used by creating a **PushBuilder** (through calling **HttpServletRequest.newPushBuilder()**), then customize the `PushBuilder` and finally call **PushBuilder.push()** to push the resource to client. 

## 3.9 Cookies

All Cookies presented on a request can be retrieved through **HttpServletRequest.getCookies()**, `Cookie` can be set to **HttpOnly** that asks the browser not to expose cookie values to scripts.

## 3.10 SSL Attributes

When a request is transmitted over a secure protocol like HTTPS, the **ServletRequest.isSecure()** will return true. The web container will also exposes following attributes that can be retrieved from `ServletRequest`.

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

Then in Spring, the framework writes a `Filter` to retrieve the certificate from attribute

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

## 3.11 Internationalization

Clients may tell the server which language they prefer, this information is communicated using header **Accept-Language**. This information can be retrieved by following methods in **ServletRequest**. 

ServletRequest:

- Locale getLocale()
- Enumeration<Locale> getLocales()

## 3.12 Request Data Encoding

Data encoding is described through the header **Content-Type**. 

## 3.13 Lifecycle of Request Object

***"Each request object is valid only within the scope of a servlet’s service method, or within the scope of a filter’s doFilter method, unless the asynchronous processing is enabled for the component and the startAsync method is invoked on the request object. In the case where asynchronous processing occurs, the request object remains valid until complete is invoked on the AsyncContext."***

# 4. Chap 4 Servlet Context

**ServletContext** defines a servlet's view of the web application. 

## 4.1 Scope of a ServletContext Interface

There is one **ServletContext** instance associated with each web application deployed into a container (per JVM).  

## 4.2 Initialization Parameters

**ServletContext** interface provides following method sto access to the context initialization parameters that are specified by the developers in **deployment descriptor**

- String getInitParameter(String name)
- Enumeration<String> getInitParameterNames()

## 4.3 Configuration Methods

Since Servlet 3.0, **ServletContext** interface provides methods to programmatically configure the context, e..g, configuring servlets, filters, listeners and so on. ***These configuration methods can only be called during initialization of application.*** 

Configuration methods can only be called:

- for **ServletContextListener.contextInitialized()**, which can be registered by **ServletContext.addListener(...)** methods 
- for **ServletContainerInitializer.onStartup(Set<Class<?>> c, ServletContext ctx)** 

Some of the configuration methods are explained below:

ServletContext:

- for servlets
    - addServlet(...) - for adding servlet
    - addJspFile(...) - for adding jsp 
    - ServletRegistration getServletRegistration(String servletName) - get the registration (e.g., the mapping info) of a servlet by name
- for filters
    - addFilter(...k) - for adding filter
    - FilterRegistration getFilterRegistration(String filterName) - get the registration (e.g., the mapping info) of a filter by name
- for listeners 
    - addListener(...) - for adding listener
- for sessions
    - getSessionTimeout() - get the configured session timeout
    - setSessionTimeout(int) - set session timeout
- for character encoding
    - getRequestCharacterEncoding()
    - setRequestCharacterEncoding(String)
    - getResponseCharacterEncoding()
    - setResponseCharacterEncoding(String)

## 4.4 Context Attributes

Attributes are shared within the servlet context / web application (with other servlets). Attributes of the context can be set or retrieved by following methods:

ServletContext:

- void setAttribute(String, Object)
- Object getAttribute(String)
- Enumeration<String> getAttributeNames()
- void removeAttribute()

## 4.5 Resources

**ServletContext** interface provides only access to the **static** content and documents that are part of the web application, e.g., html, images, these files are located at the **'META-INF/resources'** folder, and the path passed in are relative to this folder inside web application. These methods are not used for dynamic content.

ServletContext:

- URL getResource(String path) throws MalformedURLException
- InputStream getResourceAsStream(String path)

## 4.6 Multiple Hosts and Servlet Contexts

Web servers may support multiple logical hosts that share a single IP address on the server, this is called **virtual hosting**. In such case, ***each logical host has its onw servlet context (s).** The method **ServletContext.getVirtualServerName()** will return same name for all servlet contexts for different logical hosts. 

## 4.7 Temporary Working Directories

A temporary storage directory is provided for each servlet context by the container. This directory is available through the **javax.servlet.context.tempdir** context attribute. This directory may or may not be maintained during a servlet container restarts, but it's not visible to other web applications on the container.

# 5. Chap 5 The Response

**Response** object encapsulates the information that is returned from server to client, for HTTP protocol, this information is transmitted by HTTP headers or body.

## 5.1 Buffering

A servlet container may or may not buffer output to client. We can configure the buffering by methods on **ServletResponse**. 

ServletResponse

- int getBufferSize() 
    - get buffer size (if not buffering is used, it will be 0)
- void setBufferSize(int) 
    - set buffer size
- boolean isCommitted() 
    - check if the response is committed
- void reset() 
    - clear any data left in buffer including status code and headers
- void resetBuffer() 
    - clear any data left in buffer
- void flushBuffer() 
    - flush the buffer

## 5.2 Headers

Headers of a HTTP response can be set using the methods in **HttpServletResponse** interface. Headers must be set before the response is committed, otherwise, ther are simply ignored.

HttpServletResponse:

- void setHeader(String name, String value)
- void addHeader(String name, String value)

## 5.3 Non Blocking IO

Non-blocking IO only works with async request and upgrade processing. Similar to Non blocking IO for request, we registers a **WriteListener** to **ServletResponse** object that provides callback methods triggered by the container.

WriteListener:

- void onWritePossible() throws IOException 
    - invoked by container when it can write data
- void onError(final Throwable t) 
    - handle errors occurred

ServletOutputStream:

- boolean isReady() 
    - tells if the ServletOutputStream is ready to take data
- void setWriteListener(WriteListener writeListener) 
    - registers a `WriteListener` to write data when it's appropriate

## 5.4 Convenience Methods

**HttpServletResponse** interface provides following convenience methods, these convenience methods have side effect of committing the response, however, if these methods are called when the response is already committed, IllegalStateException is thrown.

- void sendRedirect(String location) throws IOException 
    - send a redirect response to client
- void sendError(int sc, String msg) throws IOException 
    - send an error response to client

## 5.5 Internationalization

Locale and character encoding of a response can be set using **ServletResponse.setLocale(...)** method and **ServletResponse.setCharacterEncoding(...)**. If none character encoding is specified, the default is **ISO-8859-1**.

## 5.6 Closure of Response Object

When a response is closed, the container immediately flushes all remaining data in buffer to the client. Following are events indicating the request is satisfied and the response is closed.

1. The `Servlet.service` method is returned (for non-async requests)
2. The amount of data specified in `ServletResponse.setContentLength` or `ServletResponse.setContentLengthLong` methods has been written to the response (i.e., the response body is written)
3. The `HttpServletResponse.sendError` method is called 
4. The `HttpServletResponse.sendRedirect` method is called 
5. The `AsyncContext.complete` method is called (for async requests)

## 5.7 Lifecycle of Response Object

***"Each response object is valid only within the scope of a servlet’s service method, or within the scope of a filter’s doFilter method, unless the associated request object has asynchronous processing enabled for the component. If asynchronous processing on the associated request is started, then the response object remains valid until complete method on AsyncContext is called."***

# 6. Filtering

A **Filter** is a component that can transform or modify the content and headers of HTTP requests and response. Application developer creates a filter by implementing the interface **javax.servlet.Filter** and provide a **public default constructor** (for bean instantiation), the filter is declared in **deployment descriptor** and mapped to one or more servlets or a URL pattern. 

## 6.1 Filter Lifecycle

The container first locates and instantiates the list of filters, and calls their **Filter.init(FilterConfig)** method. Only one instance for each Filter is created per JVM.  When the container receives a request, it takes the first filter in the list and calls the **Filter.doFilter(ServletRequest, ServletResponse, FilterChain)** method. A filter in filter chain may choose to block the request by not calling **FilterChain.doFilter(...)**. 

```
+---------------------------------------------------+ 
|                                                   | FilterChain
|   FilterChain -+ <+  -------+                     |
|  ^             |  |         |                     |
|  |             |  |         |                     |
|  |             |  |         |                     |
|  |             |  |         |                     |
|  |             |  |         |                     |
| doFilter()     | doFilter() |                     |
|  ^             |  ^         | ^                   |
|  |             v  |         v |                   |
| ++--------+-------+--+--------+-+---------------+ |
| |         |          |          |               | |
| | Filter1 | Filter2  | Filter3  | . . . . .     | |
| |         |          |          |               | |
| +---------+----------+----------+---------------+ |
|                                                   |
+---------------------------------------------------+
```

An implementation might be just like above. A **FilterChain** implementation internally contains a list of **Filter** for a request, it internally maintains a pointer (say `'int currFilter'`) pointing to current filter, it starts with calling first filter's `doFilter(...)` method, once the first filter finishes, it calls the filter chain's `doFilter(...)` method, then the filter chain calls next filter just like `'filters[++currFilter].doFilter(...)'`. 

Each filter may transform the request and response object by wrapping them in order, so that their behaviours are modified. E.g., for handling multipart/form-data requests, there may be a filter that wraps the original `HttpServletRequest` and `HttpServletResponse` with custom implementation that is able to parse multipart requests' Part and parameters.

After invocation of next filter in the chain, current filter may evaluates the response object. It may look like below.

```
      +---------+
      |         |        1)
----->| Filter1 |                    +---------+
  ^   |         +----> doFilter(...) |         |        2)
  |   +---------+                ^   | Filter2 |                    +---------+
  |                              |   |         +----> doFilter(...) |         |        3)
  +------------------------------+   +---------+                ^   | Filter3 |
          6)                     |                              |   |         +----> doFilter(...)
                                 +------------------------------+   +---------+        |
                                           5)                   |                      |
                                                                +----------------------+
                                                                         4)
```

Before container removes a filter instance, it will call **Filter.destroy()** method to clean up resources.

# 7. Chap 7 Sessions

Standard name of session tracking cookie (name) must be **'JSESSIONID'**, this may be customized. A session is considered **'new'** only when it has not been established. ***When session tracking information (e.g., the cookie) has returned from client to server, the session is established***, i.e., it's not considered as 'new' any more. 

The session is considered to be 'new' if:

- the client doesn't yet know about the session
- the client chooses not to join/establish the session (not returning it back to server, i.e., the session is ignored)

Session id can be retrieved or changed using methods below:

- HttpSession.getId() - get current session id
- httpServletRequest.changeSessionId() - change to newly generated session id

## 7.1 Binding Attributes into a Session 

Attributes can be bound to a session (**HttpSession** implementation). Listeners that implements **HttpSessionBindingListener** are notified for the events of attributes binding **valueBound** (attribute set) and **valueUnbound** (attribute removed). 

## 7.2 Session Timeouts

Session timeouts can be configured using methods below (in seconds). If the timeout period is set to 0, the sessions never timeout.

- ServletContext.getSessionTimeout()
- ServletContext.setSessionTimeout(int)
- HttpSession.getMaxInactiveInterval()
- HttpSession.setMaxInactiveInterval(int)

## 7.3 Last Accessed Times

The **HttpSession.getLastAccessedTime()** can be used by servlet to determine the last time the session that was accessed.

## 7.4 Threading Issues

Multiple servlets in different threads might be accessing the same session concurrently, so ***access to session's attributes must be properly synchronized***. 

# 8. Chap 9 Dispatching Requests

**RequestDispatcher** provides ways to achieve: 

- forward processing of a request from a servlet to another servlet
- include the output of another servlet in response  
- dispatch request back to servlet container for async processing request

## 8.1 Obtaining RequestDispatcher

RequestDispatcher can be obtained through **ServletContext**.

ServletContext:

- RequestDispatcher getRequestDispatcher(String path) 
    - get request dispatcher for given path, the path is relative to servlet context's path
- RequestDispatcher getNamedDispatcher(String name) 
    - get request dispatcher based on the name of a servlet that is known to the servlet context

RequestDispatcher that is relative to current request, we can also get this kind of RequestDispatcher from **ServletRequest** using **ServletRequest.getRequestDispatcher(String path)**.

## 8.2 Query Parameters in Request Dispatcher Paths

RequestDispatcher allows using query parameters, but this only applies to type **include** and **forward**.

e.g.,

```
RequestDispatcher rd = servletContext.getRequestDispatcher("/some/request?fruit=apple");
rd.include(request, response);
```

## 8.3 Using Request Dispatcher

To use a RequestDispatcher, a servlet either calls **RequestDispatcher.include(...)** or **RequestDispatcher.forward(...)** method. The dispatch of a request to the target servlet occurs in the same thread as the original request, i.e., the request is handled by same thread (excluding async processing). 

### 8.3.1 Include Method

The **RequestDispatcher.include(...)** method can be called at any time, the target servlet has access to all aspects of the request, but it can only write data to OutputStream of the response, ***it can't modify the headers of response object***, such calls may be ignored. 

Except for servlets obtainer by `getNamedDispatcher(...)` method, when a request is dispatched using the `include` method, the container will set following attributes that made available for the targeted servlet

- javax.servlet.include.mapping - javax.servlet.include.request_uri - same as HttpServletRequest.getRequestURI()   
- javax.servlet.include.context_path
    - same as HttpServletRequest.getContextPath()
- javax.servlet.include.servlet_path
    - same as HttpServletRequest.getServletPath()
- javax.servlet.include.path_info
    - same as HttpServletRequest.getPathInfo()
- javax.servlet.include.query_string
    - same as HttpServletRequest.getQueryString()

### 8.3.2 Forward Method

The **RequestDispatcher.forward(...)** method can only be called when there is no output committed to client. If there are data in response output buffer, these data are also cleared before the forwarded servlet's `service` method. The response will be committed and closed by the servlet container before the `forward` method returns, i.e., this is completely handled by the forwarded servlet. For the forwarded servlet, except for dispatcher that is obtained by `getNamedDispatcher(...)`, container will set following attributes:

- javax.servlet.forward.mapping
- javax.servlet.forward.request_uri
     - same as HttpServletRequest.getRequestURI()   
- javax.servlet.forward.context_path
    - same as HttpServletRequest.getContextPath()
- javax.servlet.forward.servlet_path
    - same as HttpServletRequest.getServletPath()
- javax.servlet.forward.path_info
    - same as HttpServletRequest.getPathInfo()
- javax.servlet.forward.query_string
    - same as HttpServletRequest.getQueryString()

## 8.4 Obtaining an AsyncContext

**AsyncContext** is obtained through **ServletRequest.startAsync()**, we can either complete the processing through **AsyncContext.complete()** or dispatch the requests back to the container (to another servlet).

AsyncContext:

- dispatch(path)
- dispatch(ServletContext, path)
- dispatch()
    - this method uses the same path as the original URI

For requests dispatched from AsyncContext, container will set following attributes:

javax.servlet.async.mapping
javax.servlet.async.request_uri
     - same as HttpServletRequest.getRequestURI()   
javax.servlet.async.context_path
    - same as HttpServletRequest.getContextPath()
javax.servlet.async.servlet_path
    - same as HttpServletRequest.getServletPath()
javax.servlet.async.path_info
    - same as HttpServletRequest.getPathInfo()
javax.servlet.async.query_string
    - same as HttpServletRequest.getQueryString()

# 9. Chap 10 Web Applications

## 9.1 WebApp and ServletContext

Web application and ServletContext has an **one to one** relationship. 

## 9.2 Directory Structure 

```
- index.html
- *.jsp
- /images/
- /WEB-INF/lib
    - web.xml (deployment descriptor)
    - /classes 
    - /lib/*.jar
```

## 9.3 WebApp Class Loader

***"An implementation MUST also guarantee that for every web application deployed in a container, a call to Thread.currentThread.getContextClassLoader() MUST return a ClassLoader instance that implements the contract specified in this section. Furthermore, the ClassLoader instance MUST be a separate instance for each deployed web application."***

# 10. Chap 11 Application Lifecycle Events

## 10.1 Servlet Context Events

Event Type | Description | Listener Interface
-----------|-------------|-------------------
Lifecycle | The servlet context has just been created and is available to service its first request, or the servlet context is about to be shut down. | javax.servlet.ServletContextListener
Changes to attributes | Attributes on the servlet context have been added, removed, or replaced. | javax.servlet.ServletContextAttributeListener

## 10.2 HTTP Session Events

Event Type | Description | Listener Interface
-----------|-------------|-------------------
Lifecycle | An HttpSession has been created, invalidated, or timed out. | javax.servlet.http.HttpSessionListener
Changes to attributes | Attributes have been added, removed, or replaced on an HttpSession. | javax.servlet.http.HttpSessionAttributeListener 
Changes to id | The id of HttpSession has been changed. | javax.servlet.http.HttpSessionIdListener
Session migration | HttpSession has been activated or passivated. | javax.servlet.http.HttpSessionActivationListener
Object binding | Object has been bound to or unbound from HttpSession | javax.servlet.http.HttpSessionBindingListener

## 10.3 Servlet Request Events

Event Type | Description | Listener Interface
-----------|-------------|-------------------
Lifecycle | A servlet request has started being processed by Web components. | javax.servlet.ServletRequestListener
Changes to attributes | Attributes have been added, removed, or replaced on a ServletRequest. | javax.servlet.ServletRequestAttributeListener
Async events | A timeout, connection termination or completion of async processing | javax.servlet.AsyncListener

***Proper synchronization for HttpSession and ServletContext attributes are needed by the application developer, including listeners.***

# 11. Chap 12 Mapping Requests to Servlets

For **context path**, ***"the Web application selected must have the longest context path that matches the start of the request URL".***

For **request path**, following mapping rules are **used in order**, and the **first successful match is used** with no further attempts.

Rules (**in-order** and **case-sensitive**):

1. find exact match of path
2. recursively match the longest path-prefix
3. match by extension if the path contains one (e.g., ends with '.jsp')
4. attempt to serve requests using default servlet

# 12. Chap 13 Security

**HttpServletRequest** interface provides a few methods for security:

- boolean authenticate(HttpServletResponse response)
- void login(String username, String password) throws ServletException
- void logout() throws ServletException 
- String getRemoteUser()
- boolean isUserInRole(String role)
- Principal getUserPrincipal()

## 12.1 Authentication

- HTTP Basic Authentication
    - username and password based authentication, server requests (or say, challenges) user to authenticate itself, as part of this request/challenge, the server passes a **realm** (which is essentially a specified string that may never change, e.g., "Access to the site"), then the client encodes the username and password using **base64**, it's not secure. 
- HTTP Digest Authentication
    - similar to BASIC, but it sends password's hash (possibly with additional data that is hashed together with the password) to server.
- HTTPS Client Authentication
    - exchanging digital certificates between client and server, instead of using username and password.
- Form Based Authentication


