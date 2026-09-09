---
title: Not another blog post on APIs!
date: 2026-09-09 11:05:00 +0530
categories: [Networking Automation]
tags: [rest-api, python, cml, tls]
---

The word API has become such a common term in our everyday lives that we have almost stopped thinking about what happens behind the scenes.

Very simply put, an API helps entity A talk to entity B. That entity could be a switch, a server, a piece of software, or anything that is capable of running the necessary protocols.

It is a contract, an **agreed** upon contract, between client and server.

While there are different API architectures that use different protocols, the one that's widely used and the focus of today's blog is the REST API. It uses "verbs" to get work done between client and server, and runs over HTTP/S.

We will be performing the following two tasks:

1. Send credentials via client (Python script) to login to CML
2. Retrieve token and get CML lab IDs

## Task One: Send credentials via client (Python script) to login to CML

The script is ready. I have added comments for better understanding.

![Python script that posts CML credentials from environment variables to the authenticate API](/assets/img/posts/not-another-blog-post-on-apis/login-script.png)

When we run it, Python (client) indicates it cannot trust the CML (server)! See the error:

![SSLError showing Python cannot verify CML's self-signed certificate](/assets/img/posts/not-another-blog-post-on-apis/ssl-error.png)

With the HTTP'S' (S = TLS (Transport Layer Security)) in my CML URL, the client and server first perform a TLS handshake before anything!

This TLS handshake does 2 things:

a. Encrypt the line for us to send information
b. Make sure the server (our CML) is authentic! — this is where the TLS certificate comes into the picture

For the lab, we'll turn it off, but ideally we should trust the lab's CA explicitly by exporting CML's self-signed cert to a `.pem` file and adding `verify=/path/to/ca.pem` to our script. In prod, have a trusted CA sign the cert.

With this, we have completed our first task and successfully authenticated to CML via API.

## Task Two: Retrieve token and get CML lab IDs

Every HTTP conversation has 2 parts: Request (client sends) and Response (server responds).

Request has 4 parts: URL, Method, Headers, Body.

Response has 3 parts: Status code, Headers, Body.

The token goes in the header of every request. REST is stateless. The server forgets you after each request, so every call must re-carry the token to prove who you are.

Adding to the previous code (read the comments):

![Python script that reuses the login token in the Authorization header to list CML labs](/assets/img/posts/not-another-blog-post-on-apis/labs-script.png)

The output shows the 5 lab UUIDs:

![Terminal output listing five CML lab UUIDs after a successful login](/assets/img/posts/not-another-blog-post-on-apis/lab-uuids.png)
