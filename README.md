# HTTP-Server-Latency-Benchmarker
Multi-process dynamic server that hosts database and files and complimentary benchmarker.

*Last updated on Mar 2024*

## Installation 
To build the executables:

First navigate to the sub directory containing the source code, for instance [./lookup/](./lookup/),
then execute the following command in your terminal: 
```bash
$ make
```
To remove the intermediate build artifacts (object file and executable):
```bash
$ make clean
```
Specific usages of executables are introduced below.

## 1) `mdb-lookup-server`
Directory: [./lookup/](./lookup/)

### Usage
`mdb-lookup-server` is a program that searches for a particular name or message in a given database file. It will prompt you for a string to search for.  The executable communicates with clients over a TCP/IP
connection
* If you simply press ENTER, it will show you all the records in the database. 
* If you type something, it will show you the records containing what you typed, either in the name field or in the msg field.

`mdb-lookup-server` takes the following arguments:
```bash
$ mdb-lookup-server <server-port> <database>
```
`mdb-lookup-server` listens on `<server-port>` for TCP connections, and searches through the `<database>` file according to each client’s search queries.
`mdb-lookup-server` supports multiple simultaneous client connections by handling each client in a separate process.

## 2) dynamic web server
Directory: [./server/](./server/)

### Usage
`http-server` works in
conjunction with `mdb-lookup-server`, using a three-tier architecture: it will defer to `mdb-lookup-server` to perform each search, and
format the search results as an HTML table that it sends to the HTTP client.

`http-server` also serves static web pages from `<web-root>`, in addition to providing a lookup table.

#### Testing on a local machine

You may test out the dynamic `http-server` like this:
1. Start mdb-lookup-server with the class database file:
```bash
$ ./mdb-lookup-server 9999 ~j-hui/cs3157-pub/bin/mdb-cs3157
```

2. Open up **another terminal window** to the same machine and run http-server, which will connect to the mdb-lookup-server you started earlier:
```bash
$ ./http-server 8888 ~/html localhost 10086
```

3. Point your browser to the HTTP server:
`http://the.host.name:8888/mdb-lookup`


4. If you type "hello" into the text box (without quotes) and submit, you will
see a few messages containing hello nicely formatted in a HTML table.
If you look at the browser’s location bar, you’ll see that your search
string, "hello", is now appended to the URL:
`http://the.host.name:8888/mdb-lookup?key=hello`

### Why is it multi-process?

If `http-server` is built to handle only one connection at a time. A malicious client could easily take advantage of this limitation by opening a connection and never closing it, preventing your server from handling additional requests.

In this particular implementation, we address this limitation by extending `http-server` to handle multiple requests simultaneously. The easiest way to do so (from a programmer’s point of view) is to create a child process for each new connection using the `fork()` system call, and handle each HTTP process in that child process while the parent continues to `accept()` subsequent connections.

### Why is it dynamic?
Since the HTML table that holds lookup search results is generated on-the-fly, the server not only serves static files but is also a dynamic one.


## 3) Alternative Implementation
Directory: [./overhead/](./overhead/)

Recall that `mdb-lookup-server` only reads in the message database file once per client connection: entries added to the database since a connection is established won’t be included in any searches requested by the connected client.
If a client wants the most up-to-date search results, it can always reconnect with `mdb-lookup-server`, causing the new `mdb-lookup` handler to read in the
database file again.

The previous implementation of `http-server` does not do anything to avoid outdated search results: it
only maintains a single, persistent connection with `mdb-lookup-server` for the
entirety of its execution. While this leads to incomplete search results, it
also avoids the overhead of establishing a new mdb-lookup connection for each
search query.

In this alternative implementation, we create a copy of the previous `http-server` implementation, and modify it to establish
a new connection to `mdb-lookup-server` for EACH `/mdb-lookup?key=<search word>` search request.

This modification will introduce additional latency for those search requests.However, the magnitude of the `mdb-lookup-server` connection overhead is dwarfed by other sources of latency and is probably too small for anyone to notice.

To quantify that overhead, please see next part.


## 4) Benchmarking
Directory: [./bench/](./bench/)
We measure the latency using a latency benchmarking tool, `http-lat-bench`

### Usage

`http-lat-bench` takes the following parameters:
```bash
$ http-lat-bench <hostname> <port> <URIs...>
```
For example, you can ask it to connect with localhost on port 8888 and randomly alternate between HTTP requests for `/mdb-lookup?key=yo and /mdb-lookup` by running it like this:
```bash
$ ./http-lat-bench localhost 8888 /mdb-lookup?key=yo /mdb-lookup
```