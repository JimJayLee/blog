### Go HTTP PROXY: Not working on Firefox
I happened to find that the socks proxy was not working on Firefox browser, while it works well on Safari and Chrome. 

Debug log showed socks proxy application received socks v4 command from firefox, which is awkward. I got to switch socks proxy to http proxy.


### Go HTTP PROXY: Too Many Open Files
I recently had a situation where a proxy application I was working on in Go would run into the “too many open files” error. After some debugging, I discovered it wasn’t closing HTTP connections being made to an HTTP server.

Errors on console: 

```bash
2022/03/24 17:14:18 Error accepting: accept tcp 127.0.0.1:1080: accept: too many open files
2022/03/24 17:14:18 Got http connection:  t13.baidu.com:443
2022/03/24 17:14:18 Error accepting: accept tcp 127.0.0.1:1080: accept: too many open files
2022/03/24 17:14:18 Got http connection:  pic.rmb.bdstatic.com:443
2022/03/24 17:14:18 Error accepting: accept tcp 127.0.0.1:1080: accept: too many open files
```

Update: It turns out, I was using an default http client comes with http package which is set no timeout by default. Therefore, I have to make a custom http custom and confirm the request body  is closed where it was used. Finally  the request wasn’t setting the Connection: close header, therefore neither the client or server was requesting the connection to be closed. If the server had set the Connection header, things would’ve been fine.

The code looked like this:

  transport := &http.Transport {
    Proxy: http.ProxyFromEnvironment,
    Dial: (&net.Dialer{
      Timeout: 0,
      KeepAlive: 0
    }).Dial,
    TLSHandshakeTimeout: 10 * time.second
  }

  httpClient := &http.Client{Transport: transport}

  function submitData(url string, fileReader io.Reader) error {
    req, reqErr := http.NewRequest("POST", url, fileReader)
    if reqErr != nil {
      return reqErr
    }
    req.Header.Set("Content-Type", "application/octet-stream")

    resp, err := httpClient.Do(req)
    if err != nil {
      return err
    }
    defer resp.Body.Close()

    // read the body

    return nil
  }
The submitData function gets called over and over whenever new data needs to be POSTed. Using netstat I noticed the connections to the server were staying ESTABLISHED instead of being closed. I set the “timeout” on the dialer to 0, because the POST should block until the server is done and responds (which is an undetermined amount of time).

I know the server is responding, because the code that processes the response was in fact getting data.

It turns out, HTTP/1.1 uses persistent connections by default:

A significant difference between HTTP/1.1 and earlier versions of HTTP is that persistent connections are the default behavior of any HTTP connection.

http://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html

The solution was to inform the server that the client wants to close the connection after the transaction is complete. This can be done by setting the Connection header,

req.Header.Set("Connection", "close")
or by setting the Close property to true on the http.Request:

req.Close = true
After doing that, the “too many open files” issue went away as the program was no longer keeping HTTP connections open and thus not using up file descriptors.