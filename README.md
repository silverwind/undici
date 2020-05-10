# undici

[![Build
Status](https://travis-ci.com/mcollina/undici.svg?branch=master)](https://travis-ci.com/mcollina/undici)

An HTTP/1.1 client, written from scratch for Node.js.

> Undici means eleven in Italian. 1.1 -> 11 -> Eleven -> Undici.
It is also a Stranger Things reference.

<!--
Picture of Eleven
-->

## Install

```
npm i undici
```

## API

<a name='client'></a>
### new undici.Client(url, opts)

A basic HTTP/1.1 client, mapped on top a single TCP/TLS connection.
Keepalive is enabled by default, and it cannot be turned off.

The `url` will be used to extract the protocol and the domain/IP
address. The path is discarded.

Options:

- `timeout`, the timeout after which a request will time out, in
  milliseconds.
  Default: `30e3` milliseconds (30s).

- `maxAbortedPayload`, Maximum number of bytes read after which an
  aborted response will close the connection. Closing the connection
  will error other inflight requests in pipeline.
  Default: `1e6` bytes (1MiB).

- `pipelining`, the amount of concurrent requests to be sent over the
  single TCP/TLS connection according to
  [RFC7230](https://tools.ietf.org/html/rfc7230#section-6.3.2). Default: `1`.

<a name='request'></a>
#### `client.request(opts, cb(err, data))`

Performs an HTTP request.

Options:

* `path`
* `method`
* `body`, it can be a `String`, a `Buffer` or a `stream.Readable`.
* `headers`, an object with header-value pairs.
* `idempotent`, whether the requests can be safely retried or not.
  If `false` the request won't be sent until all preceeding
  requests in the pipeline has completed.
  Default: `true` if `method` is `HEAD` or `GET`.

Headers are represented by an object like this:

```js
{
  'content-length': '123',
  'content-type': 'text/plain',
  connection: 'keep-alive',
  host: 'mysite.com',
  accept: '*/*'
}
```
Keys are lowercased. Values are not modified.
If you don't specify a `host` header, it will be derived from the `url` of the client instance.

The `data` parameter in the callback is defined as follow:

* `statusCode`
* `headers`
* `body`, a `stream.Readable` with the body to read. A user **must**
  either fully consume or destroy the body unless there is an error, or no further requests
  will be processed.

Example:

```js
const { Client } = require('undici')
const client = new Client(`http://localhost:3000`)

client.request({
  path: '/',
  method: 'GET'
}, function (err, data) {
  if (err) {
    // handle this in some way!
    return
  }
  const {
    statusCode,
    headers,
    body
  } = data


  console.log('response received', statusCode)
  console.log('headers', headers)

  body.setEncoding('utf8')
  body.on('data', console.log)

  client.close()
})
```

Abortion is supported by destroying the request or
response body.

```js
// Abort while sending request.
const body = new stream.Passthrough()
const promise = client.request({
  path: '/',
  method: 'POST',
  body
})
body.destroy()
const { statusCode, headers } = await promise
```

```js
// Abort while reading response.
const { statusCode, headers, body } = await client.request({
  path: '/',
  method: 'GET'
})
body.destroy()
```

Promises and async await are supported as well!
```js
const { statusCode, headers, body } = await client.request({
  path: '/',
  method: 'GET'
})
```
#### `client.pipelining`

Property to get and set the pipelining factor.

#### `client.full`

True if the number of requests waiting to be sent is greater
than the pipelining factor. Keeping a client full ensures that once the
inflight set of requests finishes there is a full batch ready to go.

#### `client.closed`

True after `client.close()` has been called.

#### `client.destroyed`

True after `client.destroyed()` has been called or `client.close()` has been
called and the client shutdown has completed.

<a name='close'></a>
#### `client.close([callback])`

Closes the client and gracefully waits fo enqueued requests to
complete before invoking the callback.

Returns a promise if no callback is provided.

<a name='destroy'></a>
#### `client.destroy([err][, callback])`

Destroy the client abruptly with the given `err`. All the current and enqueued
requests will be aborted and error. Waits until socket is closed before 
invoking the callback.

Returns a promise if no callback is provided.

#### Events

* `'drain'`, emitted when the queue is empty unless the client
  is closed or destroyed.

### undici.Pool

A pool of [`Client`][] connected to the same upstream target.
A pool creates a fixed number of [`Client`][]

Options:

* `connections`, the number of clients to create. Default `100`.
* `pipelining`, the pipelining factor. Default `1`.
* `timeout`, the timeout for each request. Default `30000` milliseconds.

#### `pool.request(req, cb)`

Calls [`client.request(req, cb)`][request] on one of the clients.

#### `pool.close()`

Calls [`client.close()`](#close) on all the clients.

#### `pool.destroy()`

Calls [`client.destroy()`](#destroy) on all the clients.

## License

MIT

[`Client`]: #client
[request]: #request
