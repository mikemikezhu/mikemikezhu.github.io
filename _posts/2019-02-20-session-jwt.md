---
layout: post
title:  "Comparison - Session vs JSON Web Token Authentication"
categories: [Dev]
tag: [session, jwt, spring]
author: Mikemike Zhu
---

## Abstract

Recently, I am trying both session and JWT (JSON web token) authentication during the development of the backend projects. In this article, I will briefly discuss the difference between these two methods to authenticate users.

<br/>

## Why do we need them?

The reason why we need session and JWT authentication is that, **HTTP is stateless**. 

For example, suppose that our user has logged in an online system before. Considering that HTTP is stateless, our user may need to login the system every time he makes some requests.
Hence, we need something to capture such login state, and therefore session and JWT authentication are introduced to solve this problem.

Session and JWT are the techniques to let the server knows **whether the user has been authenticated**, by sending either session id or token with HTTP requests.

<br/>

## What is the difference of them?

#### 1. Session

In the session based authentication, the server will create a session with session id after the user has logged in. Then the session id will be stored in the cookie. Whenever the user makes some requests, the session id will be sent through the cookie, which allows the server to verify the user identity by comparing the session stored in its memory.

**-- Advantages:**

- The session id will be stored in **HttpOnly cookie**, which means JavaScript can not read our session id from the cookie, because the HttpOnly flag tells the browser that such cookie can only be accessed by the server. This can prevent the malicious JavaScript code to perform **XSS attack**, and therefore secure our service.

**-- Disadvantages:**

- Considering that session is stored in the **memory of the server**, this will increase the memory usage of the server when we have a huge number of users using the system at once.

- Session will only be stored in the memory of one particular server. This works fine for building small applications. However, if we have multiple servers, others server may not know the session state stored in that particular server, which leads to the **bad scalability**.

In order to solve the above problems, we have to come up with a separate central session storage system that all of the servers have access to. For example, we can have a dedicated server to run a tool like **Redis** for session storage, which can solve the above problems.

- Another vulnerability of session based authentication is that, cookie is vulnerable to **CSRF (Cross-site Request Forgery) attack**. For example, suppose that the user has already logged in the trusted website, with the session id stored in the cookie. Then the user may continue to visit the malicious website. And the malivious website may send some forged requests to server with the specific cookie. As the server can verify the session id stored in the cookie, it will consider the requests as valid, which indeed comes from the malicious website.

In order to protect our service from CSRF attack, we may verify the "Referer" of HTTP header to check the source of the request. However, some browsers such as IE6 allows the modifications on the "Referer" of HTTP header. Alternatively, we can create an "Anti-CSRF token". If the token from the requests is verified with the token stored in the session, then we may consider the requests as valid.

#### 2. JWT

In JWT (JSON web token) authentication, the server will create a token with a secret and send it to the client. Usually, the client will store the token locally (e.g. local storage, or even cookie). Then the client will include such token in every HTTP request. The server will verify the identify of user with the token.

**-- Advantages:**

- Considering that, the session is **not** stored in the memory of the server, JWT authentication solves the memory usage problems of session based authentication.

- In addition, JWT authentication provides the **scalability** if we have multiple servers, because each server only needs to verify the token provided by the user, instead of relying on the session stored in the memory of one particular server.

**-- Disadvantages:**

- Storing JWT token in local storage or session storage in browser is extremely vulnerable to **XSS attack**, because hackers may make use of JavaScript code to get the JWT token from local storage or session storage.

<br/>