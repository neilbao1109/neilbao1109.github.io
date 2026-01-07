---
title: Chrome WebUI for Trusted and Untrusted 
date: 2025-05-04 16:10:00 +0800
categories: [Chrome]
tags: [webui]
render_with_liquid: false
mermaid: true
---

Chrome WebUI incorporates both trusted (chrome://) and untrusted (chrome-untrusted://) components. this would illustrate a clear separation of privileges and defined communication channels. The untrusted WebUI, handling external data and APIs, is isolated within an iframe inside the trusted WebUI, which communicates with the browser's native code via Mojo.

Here is a breakdown of the architecture.

---

**Architectural Components and Flow**

The architecture is designed to protect the privileged context of the browser from potentially unsafe data and code processed by the WebUI.

*Trusted WebUI (chrome://)*

* Role: Serves as the primary, fully trusted interface for the user. It has elevated privileges and can interact directly with the browser's backend.

* Communication: It uses the Mojo API to communicate with the browser's C++ native code. This is a high-performance, sandboxed Inter-Process Communication (IPC) mechanism that allows the WebUI to call browser services and receive data securely.

* Security: It operates under a strict Content Security Policy (CSP) that, among other things, will specify that it can embed an iframe from a chrome-untrusted:// source.


*Untrusted WebUI (chrome-untrusted://)*

* Role: This component is designed to handle data from untrusted sources. It is loaded into an <iframe> within the trusted WebUI. It has significantly fewer privileges than its trusted counterpart.

* External Communication:
  * RESTful APIs: It can make standard HTTPS requests to external web services, just like a regular webpage.

  * Chrome Extension APIs: It can be granted limited access to certain Chrome extension APIs, such as chrome.history. This access is explicitly defined and restricted.

* Internal Communication: It uses the standard web window.postMessage() API to communicate with the parent trusted WebUI. This is a crucial security boundary, as it allows for controlled message passing between the two origins without granting the untrusted frame direct access to the trusted parent's context.


*Communication Channels*

1. Trusted WebUI to Native Code (Mojo): The trusted WebUI sends and receives messages to and from the browser's C++ backend using Mojo interfaces. These interfaces are defined in .mojom files and provide a strongly-typed and secure way to interact with the browser's core functionalities.

2. Trusted WebUI to Untrusted WebUI (postMessage): The trusted WebUI listens for messages from the untrusted iframe and can also send messages to it. It is the responsibility of the trusted WebUI to validate and sanitize any data received from the untrusted context before acting on it.

3. Untrusted WebUI to External Services (HTTPS): The untrusted WebUI makes standard fetch or XMLHttpRequest calls to external RESTful APIs.

4. Untrusted WebUI to Chrome Extensions APIs: The untrusted WebUI can invoke extension APIs it has been granted permission to use.


