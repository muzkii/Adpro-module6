# Advanced Programming Rust 
by Andriyo Averill Fahrezi, NPM of 2306172325

## Module 6

### Commit 1 Reflection Notes: Handling Connection and Checking Response

In this milestone, I learned how to establish a basic TCP connection using Rust's `TcpListener`. By running the server on `127.0.0.1:7878`, I observed the connection requests being received from the browser, such as the message "Connection established!" in the terminal, indicating that the server successfully received a request. On this **Code Breakdown**:

```Rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();
        println!("Connection established!");
    }
}
```

Now, adding `handle_connection` function as attached below. It is basically a function to receive a TCP stream from `(mut stream: TcpStream)`. Then, it efficiently reads and processes HTTP request headers line by line by `buf_reader.lines().map(...).take_while(...)` until an empty line is encountered `take_while(|line| !line.is_empty())`. After every line have been read, it stores them in a vector using `.collect()`, and then prints the collected headers for debugging.

**`handle_connection` function**:
```Rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();


    println!("Request: {:#?}", http_request);
}
```

After updating the code with the `handle_connection` function that processed each incoming stream. I noticed that whenever a browser connect to the server, it automatically sends a request, typically looking like:

```
Request: [
    "GET / HTTP/1.1",
    "Host: 127.0.0.1:7878",
    "User-Agent: Mozilla/5.0...",
    "Accept: text/html",
    "Connection: keep-alive",
]
```

I also explored the `BufReader` and the `.lines()` iterator to read the HTTP request from the browser. This allowed me to understand the structure of a basic HTTP request, including the request method (`GET`), path, and headers. Understanding this low-level detail helps to see what happens behind the scenes when accessing web pages through a browser.
