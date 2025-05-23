---
title: GRPC
layout: default
---

[← Back to Main Page]({{ "/" | relative_url }})

* TOC
{:toc}


# GRPC
# Remote Procedure Calls (RPC)
Remote Procedure Calls (RPC) let you write code that appears to call functions locally but actually runs those functions on remote machines. The RPC layer handles all the network messaging and data conversion so that you, the programmer, can pretend it’s all just a simple local function call.
* HTTP is a higher-level application protocol primarily for request–response, often tied to web-based communication patterns.
* RPC frameworks give you function-call semantics that can be more flexible than just sending HTTP requests.

With RPC, you call someFunction(args...) as if it’s local, but under the hood, the RPC system sends a message across the network to another machine, which runs someFunction and sends a response back.
Underneath, sockets are still used. However, all the low-level networking code is auto-generated by the RPC compiler or framework, so you don’t manually write socket logic.

Typically, RPCs block the caller until the response arrives over the network.
This is different from a purely local function call in terms of latency and possible partial failures, but the concept is similar syntactically.

You generally can’t send raw pointers over an RPC. Those pointers refer to memory addresses valid only on the local machine.
If you absolutely need the data at the server, you must serialize the entire object and reconstruct it on the remote side.

Different machines might store data in different byte orders or formats.
* “Marshalling” means converting your data into a standard (e.g., XDR) format before sending over the network.
* “Unmarshalling” is converting from that standard format back into the local machine’s format.

The server registers itself (its methods, version, address, etc.) in a naming or directory service.
Clients look up which server provides the method they need, then connect to it using IP/port details from the directory service.

Lost messages can be handled by timeouts and retry logic in the underlying transport. If a client crashes after sending a request, the response that arrives at the client side is called an “orphan.” Various strategies (like “extermination,” “reincarnation,” etc.) can handle these stale replies.
<script src="{{ '/assets/js/dark-mode.js' | relative_url }}"></script>
