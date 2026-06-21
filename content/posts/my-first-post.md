+++
date = '2026-06-21T21:26:19+05:30'
draft = false
title = 'Introduction to Node.js'
+++

JavaScript has long been the language of the browser, powering interactive user experiences across the web. However, with the introduction of Node.js, JavaScript expanded beyond the browser and became a powerful tool for building server-side applications, APIs, command-line tools, and more.

## What is Node.js?

Node.js is an open-source, cross-platform runtime environment that allows developers to execute JavaScript code outside the browser. Built on Google's V8 JavaScript engine, Node.js compiles JavaScript into machine code, making it fast and efficient.

Unlike traditional web servers that create a new thread for every request, Node.js uses an event-driven, non-blocking I/O model. This enables it to handle thousands of concurrent connections with minimal resource consumption.

## Why Use Node.js?

Node.js has become one of the most popular backend technologies for several reasons:

- **JavaScript Everywhere** – Use the same language on both the frontend and backend.
- **High Performance** – Powered by the V8 engine for fast execution.
- **Scalability** – Ideal for real-time applications and APIs.
- **Large Ecosystem** – Access thousands of packages through npm.
- **Strong Community Support** – Backed by a vast and active developer community.

## Installing Node.js

The easiest way to install Node.js is through the official installer available for Windows, macOS, and Linux.

After installation, verify it is installed correctly:

```bash
node --version
npm --version
```

## Your First Node.js Program

Create a file named `app.js`:

```javascript
console.log("Hello, Node.js!");
```

Run it using:

```bash
node app.js
```

Output:

```text
Hello, Node.js!
```

## Creating a Simple HTTP Server

One of the most common use cases for Node.js is building web servers.

```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Hello from Node.js!");
});

server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

Start the server:

```bash
node app.js
```

Visit:

```text
http://localhost:3000
```

You should see:

```text
Hello from Node.js!
```

## The npm Ecosystem

Node.js includes npm (Node Package Manager), which provides access to millions of open-source packages.

Initialize a project:

```bash
npm init -y
```

Install a package:

```bash
npm install express
```

Express is one of the most widely used frameworks for building web applications and APIs with Node.js.

## Common Use Cases

Node.js is commonly used for:

- REST APIs
- Real-time chat applications
- Streaming services
- Microservices
- Command-line tools
- Server-side rendering
- IoT applications

## Conclusion

Node.js transformed JavaScript from a browser-only language into a full-stack development platform. Its performance, scalability, and rich ecosystem make it an excellent choice for building modern web applications and backend services.

Whether you're a frontend developer looking to expand into backend development or a full-stack engineer building scalable systems, Node.js is a valuable technology to learn and master.
