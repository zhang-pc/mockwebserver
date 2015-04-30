### Enqueue a sequence of responses ###
This example enqueues an HTTP redirect and then a final HTTP response. The client follows the redirect and sees the final response:
```
    public void testRedirect() throws Exception {
        server.play();
        server.enqueue(new MockResponse()
                .setResponseCode(HttpURLConnection.HTTP_MOVED_TEMP)
                .addHeader("Location: " + server.getUrl("/new-path"))
                .setBody("This page has moved!"));
        server.enqueue(new MockResponse().setBody("This is the new location!"));

        URLConnection connection = server.getUrl("/").openConnection();
        InputStream in = connection.getInputStream();
        BufferedReader reader = new BufferedReader(new InputStreamReader(in));
        assertEquals("This is the new location!", reader.readLine());

        RecordedRequest first = server.takeRequest();
        assertEquals("GET / HTTP/1.1", first.getRequestLine());
        RecordedRequest redirect = server.takeRequest();
        assertEquals("GET /new-path HTTP/1.1", redirect.getRequestLine());
    }
```

### Invalid Responses ###
This is difficult to test with regular web servers. We're testing that our HTTP client responds appropriately when the HTTP response is malformed.

In this case we want to make sure it throws an `IOException` and not a `NumberFormatException` when the chunked encoding is invalid. This test uses `clearHeaders` to remove the `Content-Length` header that's added by default:
```
    public void testNonHexadecimalChunkSize() throws Exception {
        server.enqueue(new MockResponse()
                .setBody("G\r\nxxxxxxxxxxxxxxxx\r\n0\r\n\r\n")
                .clearHeaders()
                .addHeader("Transfer-encoding: chunked"));
        server.play();

        URLConnection connection = server.getUrl("/").openConnection();
        InputStream in = connection.getInputStream();
        try {
            in.read();
            fail();
        } catch (IOException expected) {
        }
    }
```

### Request Sequencing ###
Efficient HTTP clients usually reuse socket connections. MockWebServer includes a _sequence number_ on each recorded request which exposes whether a socket connection was reused. If the sequence number is 0, the request came on a new connection. If it is 3, then 3 other requests were made on the same socket connection.

In this test we check that a new socket is opened when a connection times out. We simulate a timeout by making the actual content length disagree with the `Content-Length` header.
```
    public void testResponseTimeout() throws Exception {
        server.enqueue(new MockResponse()
                .setBody("ABC")
                .clearHeaders()
                .addHeader("Content-Length: 4"));
        server.enqueue(new MockResponse()
                .setBody("DEF"));
        server.play();

        URLConnection urlConnection = server.getUrl("/").openConnection();
        urlConnection.setReadTimeout(1000);
        InputStream in = urlConnection.getInputStream();
        assertEquals('A', in.read());
        assertEquals('B', in.read());
        assertEquals('C', in.read());
        try {
            in.read(); // if Content-Length was accurate, this would return -1 immediately
            fail();
        } catch (SocketTimeoutException expected) {
        }

        URLConnection urlConnection2 = server.getUrl("/").openConnection();
        InputStream in2 = urlConnection2.getInputStream();
        assertEquals('D', in2.read());
        assertEquals('E', in2.read());
        assertEquals('F', in2.read());
        assertEquals(-1, in2.read());

        // each request comes on its own socket connection
        assertEquals(0, server.takeRequest().getSequenceNumber());
        assertEquals(0, server.takeRequest().getSequenceNumber());
    }
```

### Any HTTP Client ###
This example uses [Apache HTTP client](http://hc.apache.org/httpcomponents-client-ga/) with MockWebServer.
```
    public void testApacheHttpClient() throws Exception {
        server.enqueue(new MockResponse().setBody("hello world"));
        server.play();

        URL url = server.getUrl("/");
        DefaultHttpClient client = new DefaultHttpClient();
        HttpGet get = new HttpGet(url.toURI());
        get.addHeader("Accept-Language", "en-US");
        HttpResponse response = client.execute(get);
        InputStream in = response.getEntity().getContent();
        BufferedReader reader = new BufferedReader(new InputStreamReader(in));
        assertEquals(HttpURLConnection.HTTP_OK, response.getStatusLine().getStatusCode());
        assertEquals("hello world", reader.readLine());

        RecordedRequest request = server.takeRequest();
        assertEquals("GET / HTTP/1.1", request.getRequestLine());
        assertTrue(request.getHeaders().contains("Accept-Language: en-US"));
    }
```