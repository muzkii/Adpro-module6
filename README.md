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

### Commit 2 Reflection Notes: Returning HTML

| PNG File | Image |
| ----- | ----- |
| commit2.png | ![Commit2screencapture](https://github.com/user-attachments/assets/125bba01-7c1e-4196-8ee0-a91d8fc24bdf) |

**Current Code:**
```Rust
use std::{
    fs,
    io::{prelude::*, BufReader},
    net::{TcpListener, TcpStream},
};


fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();


    for stream in listener.incoming() {
        let stream = stream.unwrap();


        handle_connection(stream);
    }
}


fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();


    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();

    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

In this milestone, I extended the basic TCP server to handle and respond to HTTP requests with a simple HTML page. By enhancing the `handle_connection` function, I learned how to construct an HTTP response in Rust manually and send it back to the client's browser.

I also now know how file reading works in Rust using the `fs::read_to_string` function. The function uses the parameter `hello.html`, in the full code `fs::read_to_string("hello.html").unwrap()` reads the contents of the HTML file named `hello.html`. The use of `unwrap()` assumes the file is present, and if not, the program will panic.


### Commit 3 Reflection Notes: Validating Request and Selectively Responding

| PNG File | Image |
| ----- | ----- |
| commit3.png | ![commit3](https://github.com/user-attachments/assets/c81bf0aa-0182-4a64-be9a-7aead0b9b771) |

**Current Code:**
```Rust
use std::{
    fs,
    io::{prelude::*, BufReader},
    net::{TcpListener, TcpStream},
};


fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();


    for stream in listener.incoming() {
        let stream = stream.unwrap();


        handle_connection(stream);
    }
}


fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();
    
    let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

This milestone introduced conditional responses based on the request path. I implemented such that the implementation allowed the server to differentiate valid routes and return a 404 page for invalid ones. By using pattern matching on the request line, I understood the basics of Rust's pattern matching capabilities. By applying some refactoring also based on [Validating Request and Selectively Responding, A Touch of Refactoring](https://doc.rust-lang.org/stable/book/ch21-01-single-threaded.html?highlight=validating%20the%20request#validating-the-request-and-selectively-responding). 

Additionally, I explored error handling when reading files that may not exist (`404.html`). By modifying the previous `hello.html` to display an error based on an ERROR 404 response. On `handle_connection` function, the server now checks the request line (first line of the incoming request) to determine the requested path. By using `.lines().next()` to extract the first line, I identified the requested resource (like / for the root path). The condition `if request_line == "GET / HTTP/1.1"` specifically checks for a request to the root (/). If the path is not the root, a 404 error response is returned.

### Commit 4 Reflection Notes: Simulating Slow Requests

**Changed Code:**
```Rust
use std::thread;
use std::time::Duration;

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();
    
    let (status_line, filename) = match request_line.as_str() {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(10));
            ("HTTP/1.1 200 OK", "hello.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

In this milestone, I simulated a slow response by adding a delay using `thread::sleep`. By accessing `/sleep` path and the default route simultaneously in multiple browser windows, I observed how the single-threaded server handled requests sequentially, causing delays for all requests.

Based on the code snippet above, the server now checks the request line and matches it againts two possible paths: `/` and `/sleep`. If the `/sleep` path is accessed, the server sleeps for 10 seconds before responding. The simulated delay would mimic the scenanarios where there are multiple users that is currently accessing the web server. 

This exercise demonstrated the limitations of a single-threaded approach, especially when handling multiple clients simultaneously. It showed the importance of concurrency in server programming and highlighted potential issues with blocking operations in a real-world scenario.

### Commit 5 Reflection Notes: Multithreaded Server using Threadpool

**Changed Code:** `main.rs`
```Rust
use std::{
    fs,
    io::{prelude::*, BufReader},
    net::{TcpListener, TcpStream},
    thread,
    time::Duration,
};

use hello::ThreadPool;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();
        
        pool.execute(|| {
            handle_connection(stream);
        });
    }
}
...
```
Based on [Turning Our Single-Threaded Server into a Multithreaded Server](https://doc.rust-lang.org/stable/book/ch21-02-multithreaded.html), we refactored the server to leverage a multi-threaded thread pool using a dedicated `lib.rs` file. By integrating the ThreadPool, we efficiently managed multiple client requests simultaneously, improving the server's responsiveness and scalability.

First of all, we need to implement the `lib.rs` file to use ThreadPool in `main.rs`. So, we have created this `lib.rs` file as follows:

```Rust
use std::{
    sync::{mpsc, Arc, Mutex},
    thread::{self, JoinHandle},
};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);
        self.sender.send(job).unwrap();
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();

            println!("Worker {id} got a job; executing.");

            job();
        });

        Worker { id, thread }
    }
}
```

1. The ThreadPool struct has two main components:
- `workers`: A vector of Worker structs representing individual threads.
- `sender`: A `mpsc::Sender<Job>` for sending tasks to the workers.

The type alias `Job` defines a boxed closure `(Box<dyn FnOnce() + Send + 'static>)` that can be sent across threads by `type Job = Box<dyn FnOnce() + Send + 'static>;`

2. Creating a New Thread Pool:
The new method initializes the thread pool with a specified number of worker threads (size).
It uses a multi-producer, single-consumer (mpsc) channel to communicate between the main thread and worker threads.

```Rust
pub fn new(size: usize) -> ThreadPool {
    assert!(size > 0);

    let (sender, receiver) = mpsc::channel();
    let receiver = Arc::new(Mutex::new(receiver));

    let mut workers = Vec::with_capacity(size);

    for id in 0..size {
        workers.push(Worker::new(id, Arc::clone(&receiver)));
    }

    ThreadPool { workers, sender }
}
```
3. Executing Tasks:
The execute method is responsible for sending a task to the available worker threads. It boxes the task `(FnOnce() + Send + 'static)` to fit the defined Job type. As follows:

```Rust
pub fn execute<F>(&self, f: F)
where
    F: FnOnce() + Send + 'static,
{
    let job = Box::new(f);
    self.sender.send(job).unwrap();
}
```

4. The Worker Struct and Thread Handling:
Each Worker has an id and a thread (a JoinHandle).
The new method spawns a thread that continuously waits for jobs through the shared receiver.
The receiver is locked using receiver.lock().unwrap(), ensuring exclusive access to the job queue.

```Rust
struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();
            println!("Worker {id} got a job; executing.");
            job();
        });

        Worker { id, thread }
    }
}
```

Creating a thread pool improved the server's ability to handle multiple requests concurrently. This experience also demonstrated Rust's emphasis on safety, as it required understanding ownership, borrowing, and synchronization to avoid data races.

### Commit Reflection Bonus: Function Improvement

In this milestone, I added a new function called build to replace the `new` method for constructing a ThreadPool. The main goal was to handle potential errors gracefully during the creation process, making the pool more robust.

**Added Function and Code in `lib.rs`**:
```Rust
...
#[derive(Debug)]
pub struct PoolCreationError;
...

    pub fn build(size: usize) -> Result<ThreadPool, PoolCreationError> {
        if size == 0 {
            return Err(PoolCreationError);
        }

        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        Ok(ThreadPool { workers, sender })
    }
...
```

Initially, the `new` method used an `assert!(size > 0);` statement to ensure the size was greater than zero. While effective, it could abruptly terminate the program in case of an invalid size, which is not ideal for production-level code. The `build` method improves upon this by returning a Result<ThreadPool, PoolCreationError> instead of panicking. and providing better error handling with a custom error type (PoolCreationError).

**Comparison:**

- Before:
`new` method
```Rust
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }
```
Here, the `new` method uses `assert!` for size validation, which it would panic if the size is invalid. It returns a fully initialized `ThreadPool`

- After:
`build` method
```Rust
    pub fn build(size: usize) -> Result<ThreadPool, PoolCreationError> {
        if size == 0 {
            return Err(PoolCreationError);
        }

        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        Ok(ThreadPool { workers, sender })
    }
```
Here, the `build` returns a `Result` which would make error handling explicit. It allows calling the function to handle errors without crashing. If we compare it to the `new` method, it is more suitable because when we're talking about web production, graceful error recovery is essential rather than panics.

**Changed Code in `main.rs`**:
```Rust
...
let pool = ThreadPool::build(4).expect("Failed to create thread pool");
...
```
By using `.expect()`, we still get a panic if pool creation fails, but it's now a controlled panic with a meaningful error message. If needed, we could replace `expect` with proper error handling.
