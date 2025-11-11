
# Understanding DNS and the OSI Model Through My NGINX Deployment

When I deployed my NGINX server on an AWS EC2 instance and connected it to my domain `nginxaishabtidon.org`, I experienced several real-world networking behaviors that perfectly demonstrate how **DNS** works and how **data flows across the OSI layers**.

---

## 1. The Scenario Overview

* **Server:** AWS EC2 Ubuntu instance running NGINX
* **Public IP:** `98.81.181.142`
* **Domain:** `nginxaishabtidon.org` (managed via Cloudflare DNS)
* **Goal:** When someone types `nginxaishabtidon.org` into their browser, it should load my NGINX welcome page.

At first, the NGINX page worked using the **IP address**, but not the **domain name**.
This issue revealed how DNS resolution and OSI layers interact to deliver data.

---

## 2. Step 1 – DNS Layer: Translating Names to IP Addresses (Application Layer)

When I typed:

```
https://nginxaishabtidon.org
```

The browser didn’t know where that domain was hosted.
So it began the **DNS resolution process** — which happens at the **Application Layer (Layer 7)** of the OSI model.

**Here’s what happens:**

1. The browser checks its **local DNS cache** (recent lookups).
2. If not found, it queries the **operating system’s resolver**.
3. The resolver asks a **DNS recursive server** (often Google’s 8.8.8.8 or Cloudflare’s 1.1.1.1).
4. The DNS resolver finds the IP address of the domain from Cloudflare’s **authoritative DNS**.

**Real example:**

```bash
nslookup nginxaishabtidon.org 8.8.8.8
```

returned

```
Non-authoritative answer:
Name: nginxaishabtidon.org
Address: 98.81.181.142
```

So now, the browser knows the **destination IP (98.81.181.142)** — which is your EC2 instance.

**Error I faced:**
At first, it showed `DNS_PROBE_FINISHED_NXDOMAIN` — meaning DNS could not resolve the domain.

**Cause:**
I had Cloudflare’s **proxy (orange cloud)** turned ON. The proxy hides the original IP unless HTTPS is set up.

**Fix:**
I set the record to **DNS only (gray cloud)**, which allows direct mapping to my EC2 IP.

---

## 3. Step 2 – OSI Layers in Action (When a User Connects)

Once DNS returns the IP, the browser starts a **TCP connection** to that IP on port 80 (HTTP).
Here’s how each OSI layer contributes to making this work:

| OSI Layer                  | Example in My Project                                                          | Function                                            |
| -------------------------- | ------------------------------------------------------------------------------ | --------------------------------------------------- |
| **Layer 7 – Application**  | Browser (HTTP request to `nginxaishabtidon.org`)                               | User interacts with website                         |
| **Layer 6 – Presentation** | Browser encodes data (HTML, text, images)                                      | Data formatting and encryption (if HTTPS used)      |
| **Layer 5 – Session**      | Keeps the HTTP connection active                                               | Manages start/end of connection                     |
| **Layer 4 – Transport**    | **TCP** connection established on port **80**                                  | Ensures reliable delivery of data (3-way handshake) |
| **Layer 3 – Network**      | Uses **IP addresses**: source = your device, destination = EC2 (98.81.181.142) | Handles end-to-end routing of packets               |
| **Layer 2 – Data Link**    | Uses **MAC addresses** between each hop (router → router → EC2 NIC)            | Hop-to-hop delivery inside the same network         |
| **Layer 1 – Physical**     | Electrical signals or Wi-Fi transmitting the bits                              | Sends the actual 1s and 0s across the medium        |

---

## 4. Layer 3 vs Layer 2 in My Example

* **Layer 3 (IP Address):**
  My laptop’s IP (for example, `192.168.1.10`) communicated with EC2’s public IP (`98.81.181.142`).
  This was **end-to-end delivery** across the internet.

* **Layer 2 (MAC Address):**
  Within each hop (e.g., laptop → router, router → ISP, ISP → Amazon’s network), the **MAC address** changed.
  This is **hop-to-hop delivery** handled by switches and routers.
  Each router strips and replaces the Layer 2 header but keeps the Layer 3 IP header intact.

**Visualization:**

```
My Laptop (MAC A1) → Router (MAC B1)
Router (MAC B2) → ISP Router (MAC C1)
ISP → AWS Router → EC2 NIC (MAC Z1)
```

So, the same packet travels through many MAC addresses but always keeps the same destination IP (`98.81.181.142`).

---

## 5. Step 3 – Transport Layer (Layer 4): TCP Connection

When I opened `http://nginxaishabtidon.org`, my browser created a **TCP connection** to port 80 on the EC2 server.
This is handled by **Layer 4**.

**Example:**

```
Source port (random): 53422
Destination port: 80
```

TCP guarantees reliable delivery.
If it was a streaming or gaming application, **UDP** might be used instead for speed (no acknowledgment required).

---

## 6. Step 4 – The Web Server Responds

At the **Application layer**, NGINX received the HTTP GET request and responded with the default HTML page.

Browser displays:

```
Welcome to nginx!
If you see this page, the nginx web server is successfully installed and working.
```

This means the data traveled **up and down** all OSI layers:

1. Browser request (Layer 7 → 1)
2. NGINX received and replied (Layer 1 → 7)
3. The final web page was displayed to the user

---

## 7. Step 5 – Errors and How They Mapped to Layers

| Error                               | OSI Layer Involved                            | Cause                                 | Fix                          |
| ----------------------------------- | --------------------------------------------- | ------------------------------------- | ---------------------------- |
| `DNS_PROBE_FINISHED_NXDOMAIN`       | Layer 7 (DNS/Name Resolution)                 | Cloudflare proxy and missing A record | Changed record to “DNS only” |
| Site works on phone but not laptop  | Layer 7 (DNS cache)                           | Cached old DNS record locally         | Used `ipconfig /flushdns`    |
| “Welcome to NGINX” works on IP only | Layer 3 (Routing) + Layer 7 (Name resolution) | Domain didn’t resolve to IP           | Fixed DNS A record           |

---

## 8. Step 6 – De-Encapsulation (Reverse Flow)

When the response comes back from EC2 to your laptop:

1. **Layer 1:** Physical signals travel back through routers and ISPs.
2. **Layer 2:** Each router replaces its MAC header for the next hop.
3. **Layer 3:** IP header remains (source: EC2, destination: laptop).
4. **Layer 4:** TCP reassembles the packets.
5. **Layer 7:** Browser finally renders the HTML page.

---

## 9. Key Takeaways

* **DNS** is how human-readable names are converted into machine-usable IP addresses.
* **Layer 3 (IP)** handles global routing.
* **Layer 2 (MAC)** handles local delivery between hops.
* **Layer 4 (TCP/UDP)** ensures the correct application gets the right data.
* **Layer 7 (Application)** is where users actually see and interact with the website.
* Problems like “site not reachable” or “DNS NXDOMAIN” often occur at **Layer 7 or Layer 3**, not the physical layer.

---

### Summary Diagram: How Data Flowed in My Project

```
Browser → DNS Query (Cloudflare) → IP Resolved
↓
TCP Connection (Port 80)
↓
HTTP GET Request
↓
EC2 NGINX → HTTP Response
↓
Browser Renders “Welcome to nginx!”
```
