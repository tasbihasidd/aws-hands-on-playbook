# Lab 01: Core Concepts — Web Servers, Proxies & Load Balancing

> 🛠️ **Hands-On Lab:** Once you finish reading these concepts, follow the step-by-step AWS EC2 & Nginx implementation guide in [**`practical.md`**](./practical.md).

Before deploying servers on AWS, it is essential to understand how traffic flows across the internet and how modern web architectures stay fast, secure, and reliable. This guide breaks down the foundational concepts behind **Lab 01**.

---

## 1. What is a Web Server?

A **Web Server** is software (and the underlying hardware) that stores website files—such as HTML, CSS, JavaScript, and images—and delivers them to users over the internet when requested by a web browser.

### How It Works:
1. You type a web address (like `http://example.com`) into your browser.
2. Your browser sends an **HTTP/HTTPS request** over the internet to the server's IP address on **Port 80 (HTTP)** or **Port 443 (HTTPS)**.
3. The web server finds the requested file (e.g., `index.html`) and sends it back as an **HTTP response**, which your browser renders as a webpage.

### Popular Web Servers:
* **Nginx** (Lightweight, super fast for static content, and widely used as a reverse proxy)
* **Apache HTTP Server**

---

## 2. Forward Proxy (Client-Side Proxy)

A **Forward Proxy** sits **in front of client devices** (users/computers) and acts as a middleman when those clients send requests out to the internet.

```text
[ Client / User ] ---> [ Forward Proxy ] ---> ( Internet ) ---> [ Destination Website ]
```

### How It Works (University Dorm Example):
Instead of your computer talking directly to a website, your request first goes to the Forward Proxy (like a University Campus Gateway). The Forward Proxy inspects the request and forwards it to the website on your behalf. To the destination website, it looks like the request came from the **proxy server's public IP**, not from your private device.

### Key Benefits & Use Cases:
* **Privacy & Anonymity:** Hides the user's internal IP address from external websites (e.g., VPNs).
* **Content Filtering & Security:** Companies and universities use forward proxies to block access to restricted or distracting websites (returning `403 Forbidden`).
* **Caching:** If multiple users download the same software update or visit the same page, the proxy caches the data to save bandwidth and speed up access.

![What is a Forward Proxy - University Example](./images/forward-proxy.jpeg)

---

## 3. Reverse Proxy (Server-Side Proxy)

A **Reverse Proxy** sits **in front of backend servers** and intercepts incoming requests from clients before they reach the actual application servers.

```text
[ Client / User ] ---> ( Internet ) ---> [ Reverse Proxy (Nginx) ] ---> [ Backend Server (Port 3000) ]
```

### How It Works (The Gate Guard Analogy):
Think of a Reverse Proxy as the **Gate Guard** outside a large hospital building. Visitors (clients) are not allowed to wander freely into restricted areas like the ICU or server rooms. Instead, the guard stops them at the entrance, takes their request, fetches what they need from inside, and hands it back—keeping the internal layout and private server ports (`:3000`) completely hidden.

### Key Benefits & Use Cases:
* **Backend Security (Port Hiding):** Backend application servers remain completely hidden inside a private network; users never see their internal IP addresses or ports.
* **Smart Path-Based Routing:** A single entry point can route `/` to a static website and `/api` to a backend API server.
* **SSL/TLS Termination:** Handles HTTPS encryption/decryption in one place so backend apps don't have to.
* **Caching & Compression:** Speeds up responses by caching static assets and compressing data.

### Forward Proxy vs. Reverse Proxy (Quick Comparison)

| Feature | Forward Proxy | Reverse Proxy |
| :--- | :--- | :--- |
| **Where it sits** | In front of **Clients** (Users) | In front of **Servers** (Backends) |
| **Who it protects** | Protects and hides the **Client's** identity | Protects and hides the **Server's** identity |
| **Who knows about it** | Configured on the client side | Transparent to the client (client thinks it's talking to the main server) |
| **Common Examples** | VPNs, Corporate/University Gateways | **Nginx**, AWS Application Load Balancer (ALB), Cloudflare |

---

## 4. Load Balancing & API Gateway

As website traffic grows, a single backend server can become overwhelmed and crash. **Load Balancing** is the process of distributing incoming network traffic across **multiple backend servers** so no single server bears too much load.

```text
                                      +---> [ Backend Server 1 ]
[ Clients ] ---> [ Load Balancer ] ---+---> [ Backend Server 2 ]
                                      +---> [ Backend Server 3 ]
```

### How It Works (The Hospital Queue Manager & Smart Reception):
* **Load Balancer (The Queue Manager):** Acts like a nurse managing the General OPD queue. Using algorithms like **Round Robin** (`Doctor A` $\rightarrow$ `Doctor B` $\rightarrow$ `Doctor C`), it distributes patients evenly across healthy doctors. It also performs continuous **Health Checks**—if `Doctor B` is closed/down, traffic is automatically redirected to healthy servers without downtime.
* **API Gateway (The Smart Reception):** Works alongside reverse proxies to handle smart routing and security checks—such as verifying **ID/JWT tokens**, **rate limiting** free-tier users, and directing requests to specific departments (`/cardio`, `/pharmacy`, `/billing`).

### Key Benefits of Load Balancing:
* **High Availability & Fault Tolerance:** Prevents single points of failure by routing around crashed servers.
* **Scalability:** Easily add or remove backend servers based on traffic demand.
* **Better Performance:** Prevents bottlenecks and keeps response times fast for all users.

![The Hospital Analogy - Reverse Proxy, Load Balancer & API Gateway](./images/hospital-analogy.jpeg)

---

## 5. How Theory Connects to Lab 01

In [**`practical.md`**](./practical.md), we put these concepts into action on an **AWS EC2 Ubuntu server** using **Nginx**:

1. **Web Server Role:** Nginx listens on **Port 80** and serves a static HTML file directly from `/var/www/html` when users visit `/`.
2. **Reverse Proxy Role:** When users visit `/api`, Nginx acts as a Reverse Proxy—internally forwarding the request to a backend Python service running on `127.0.0.1:3000`, keeping Port `3000` closed and protected in the AWS Security Group.

---

### Next Step: Start the Hands-On Lab
Ready to build this on AWS? Open the step-by-step lab guide with commands and troubleshooting notes:
👉 [**Go to Practical Lab (`practical.md`)**](./practical.md)