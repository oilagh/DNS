---
title: "Project 6: DNS Server"
date: 2021-08-25T15:24:08-04:00
draft: false
menu:
  docs:
    parent: "projects"
weight: 60
toc: true
description: "Build a homebrew DNS server"
lead: "Build a homebrew DNS server"
---

<div class="alert alert-info" role="alert">
    <b>This project is due at 11:59pm on April 21, 2026. </b>
</div>

## Description

You will design and implement an recursive DNS server. Your server will be (a) the authoritative server for a specified domain, and be (b) responsible for serving as a recursive resolver for a number of local clients. In doing part (b), your server will be required to interact with other DNS servers, maintain an (accurate) DNS cache, and avoid succumbing to any security vulnerabilities. You will use the real DNS protocol, though you should _not_ communicate directly with any real DNS servers.

For the assignment, we will provide you a test script that will launch your DNS server and then send it queries. We will also run a network of DNS servers that you will use to respond to clients.

## Your Programs

For this project, you will submit a program `4700dns` that implements an authoritative DNS server and a recursive DNS resolver. You must use UDP as the transport protocol; you are not required to implement TCP-based DNS. You may not use any DNS libraries other than the ones mentioned below in your program without approval. Specifically, you can only use DNS libraries that pack and unpack DNS packets; **you may not use libraries that actually interpret those packets for you, manage a DNS cache, or check for certain errors.**

### Requirements

Your program must send and receive real DNS packets. Specifically, your program must:

- Your program must be named `4700dns`, and will never exit
- Bind to a local UDP port specified at the command line
- Serve as the authoritative server for a domain we provide, serving records including `A`, `CNAME`, `NS`, `MX`, and `TXT` records
- Accept properly-formatted DNS requests from local clients, and send responses back
- Reject incoming DNS requests from clients you do not serve
- Implement recursive DNS by communicating with other DNS servers that you discover
- Maintain, properly manage, and use a cache of DNS responses
- Avoid introducing security vulnerabilities through incorrect management of responses or the cache
- You must also submit a `Makefile` that builds your DNS server; if your server does not require compilation, you must submit a `Makefile` with a blank target (e.g., `all:`)

Additionally, if you are enrolled in CS 5700, your program must:

- Handle non-responsive DNS servers by sending appropriate responses to client requests

As with other projects in the class, correctness matters most; performance is a secondary concern. Remember that network-facing code should be written defensively. Your code should check the integrity of every packet received.

### Language

You can write your code in whatever language you choose, as long as your code compiles and runs on Gradescope. Do not use libraries that are disallowed for this project. Similarly, your code must compile and run on the command line. You may use IDEs (e.g., Eclipse) during development, but do not turn in your project without a Makefile. Make sure you code has **no dependencies** on your IDE. We provide starter code in Python and Java; you are welcome to use this, but if you decide use another language, you will need to port the starter code to your language yourself.

## Testing Environment

The goal of this project is to write a real, functional DNS server. However, because DNS is often a vector for attacks, you should **not** communicate directly with any real DNS servers in this project. Instead, we will construct a network of DNS servers that you are allowed to interact with. We provide a `run` script that will test your code under different configurations.

## Starter code

Very basic starter code for the assignment is available [on the Khoury GitHub server](https://github.khoury.northeastern.edu/cs3700/dns-server-starter-code). Provided is a simple implementation of a DNS server that returns a **static** response to all queries. You may use this code as a basis for your project if you wish. To get started, you should create a copy of the starter code on your local machine (or on the Khoury Linux machines).

## Simulator

As noted above, rather than testing your router on a real network, we will test it in a simulator. The simulator takes care of reading in the configuration file, launching your server, sending requests, and checking the correctness of your output. The simulator comes with a suite of configuration files in the directory `configs/` that define situations your server must be able to handle. These are the same configuration files that we will use when evaluating everyone's code (in addition to a few additional configurations we do not reveal). Note that you do not need to parse the configuration files; the simulator is responsible for this functionality.

The simulator is implemented in a script named `run` that is available as part of the [starter code](https://github.khoury.northeastern.edu/cs3700/dns-server-starter-code).

**IMPORTANT NOTE**: You will likely need to install the `dnslib` and/or `yaml` libraries in order to run the simulator. To do so, you can run the following commands

```
pip3 install dnslib
pip3 install PyYAML
```

As noted below, if you write your program in Python, you can use the `dnslib` library to help implement your DNS server.

**ANOTHER IMPORTANT NOTE**: At no point during this project should you modify `run`. Furthermore, when we grade your code, we will use the original versions of `run` and the configurations in `configs/`, not your (possibly modified) versions.

The simulator accepts the following command line specification:

```
$ ./run <config-file>
```

The simulator will read in the config file, start up your DNS server with the appropriate arguments, and then send DNS requests to your server. The simulator will record the responses your DNS server sends back, and will compare that output to what is expected. The simulator will tell you whether your DNS server passed the test case, printing out a summary of the error if your code did not do what was expected. Note that, in most cases, the simulator will not compare the value of the `TTL` field, and will also not compare what is the `AUTHORITY` section (the exception to this is answers to `NS` queries).

The simulator send one or more queries to your DNS server, then subsequently checks all of the responses. For all of the cases where your program did not provide the correct response, the simulator will print out the query number, the response that was expected, and the response that was actually received. You can then look back in the logs to figure out what the query was and why your code did not produce the expected result.

## Program Specification

The command line syntax for your DNS server is given below.

```
$ ./4700dns [--port <port>] <root_server_IP> <file_with_your_domain_entries>
```

You DNS server should accept three command line arguments, the first of which is optional.

- `port` (Optional) The port number your program should bind to; if not provided you should bind to `0` to get an auto-assigned port.
- `root_server_IP` (Required) The IP address of the root DNS server.
- `file_with_your_domain_entries` (Required) A file with DNS entries for your authoritative domain that your server will serve (see below).

**IMPORTANT NOTE**: DNS normally uses UDP port 53, but that requires root access to bind to. Instead, in this project, we'll be using a different port for your program. When your program starts up, it should request a UDP port on localhost (i.e., it should bind to the IP address `127.0.0.1`). Once it has constructed this socket and is listening, it should print out:

`Bound to port <port>`

so that the simulator knows which port it is listening on. Additionally, to avoid our traffic being confused with real DNS traffic, for all communication with the root server and any other authoritative nameservers you learn about, **you should use UDP port 60053 instead of the normal DNS UDP port 53.**

You can assume that DNS requests only contain a single question. The DNS protocol allows multiple questions, but most servers do not support queries that have multiple questions. If you receive a request with multiple questions, you should respond with a status of `SERVFAIL`.

Your DNS server must appropriately respect and indicate DNS flags, including the `AA` and `RD` flags. If recursion is needed to resolve a query but the `RD` flag is not set, you should respond with a status of `SERVFAIL`. If you are serving a response from your local zone file, you should set the `AA` bit indicating the answer is authoritative.

Your `4700dns` program should never exit. If it receives an unexpected message, it should do its best to send back an informative error, and potentially print information about the error to the command line. But it should not exit unless it is killed by the simulator program (or receives a `SIGINT` signal that the user has press `Control-C`).

## Authoritative Server

Your server will be responsible for serving a single domain authoritatively; we will give you a zone file with all of the information you need to do this. We will be using BIND-style zone files, which represent all of the records for the domain for which your server is authoritative. A simple zone file might look like:

```
$ORIGIN example.com.
$TTL 360
@	IN	SOA	ns.example.com.	hostmaster.example.com. (20211016 21600 3600 604800 86400);
	IN	NS	ns.example.com.
	IN	MX	10	mail.example.com.
ns	IN	A	10.0.1.1
mail	IN	A	10.0.1.2
www	IN	A	10.0.1.2
```

The `$ORIGIN` means that any non-terminated labels (i.e., names that do not end in a ".") should have `example.com` appended to them. For example, the `ns` label on the sixth line is actually `ns.example.com.`. The `$ORIGIN` also tells you what domain you are authoritative for (in this example, you are authoritative for `example.com`). The `$TTL` states the time-to-live value your server should use for all records. The remaining lines of the file represent the actual records that your domain name server will serve. The format is:

```
[label] [class] [type] [type-specific-data...]
```

If the `[label]` is not included, it is assumed to re-use the previous label. So the second line (`  IN NS ns.example.com`) uses the same label as `@ IN SOA ...`; the `@` symbol is a special symbol that means "only include the `$ORIGIN` string. Thus, both of those lines refer to the label `example.com.`.

The only classes that we will be dealing with in this project are `IN` (Internet) class records.

The `type`s that we will use include `A` and `AAAA` records; `CNAME`, `NS`, and `MX` records; `SOA` records, and others including `TXT` records. Your DNS server generally should not care about the actual contents of these records, with the slight exception of `CNAME` records which require some special processing.

You can choose to parse these files yourself, or you can use a library that will do it for you (e.g., `dnslib` in Python provides DNS zone file parsing; check out the `RR.fromZone` method, which you are welcome to use). [Ars Technica](https://arstechnica.com/gadgets/2020/08/understanding-dns-anatomy-of-a-bind-zone-file/) provides a good overview of the anatomy of DNS zone files.

### Serving Your Authoritative Domain

Your DNS server should listen for incoming UDP messages on its port and then process them. At the beginning, the only messages will be requests from clients. Eventually, some of the messages will be responses to queries that your DNS server has sent to other DNS servers. You are welcome to use a library to unpack the DNS responses (e.g., `dnslib` includes a `DNSRecord.parse` method that does this), or you can do it yourself (note that parsing DNS is a bit tricky, so this is very much not recommended). Once you have a response you wish to send to a client, again you can use a library to pack your DNS response (e.g., `dnslib` includes `DNSRecord.reply` to build a skeleton reply from an incoming query, and the `DNSRecord.pack` method for formatting the DNS response).

There are a few notes about how exactly your DNS server should put together its response when serving requests that it is authoritative for.

- For `NS` queries, you can choose whether to include responses with `NS` records in either the `ANSWER` or `AUTHORITY` fields (different servers take different approaches here). You should include the corresponding `A` record as an `ADDITIONAL` record (this is called a "glue record").

- For `CNAME` queries, you should include the corresponding the `A` record as an additional `ANSWER` if you are authoritative for that `A` record. If you are not, you should not include it.

- Your code should also properly handle `TXT` records that may exist in your zone file.

- If you do not have an entry for the requested DNS name and it is under your domain, you should return a response that includes the status `NXDOMAIN`.

## Recursive DNS Resolver

Your DNS server will also be responsible for serving as a recursive DNS resolver for local clients, meaning you will serve their requests for both your authoritative domain as well as for other domains. To do so, you will regularly need to contact other DNS servers to obtain answers for your clients. For example, if you receive a client request for the `A` record for `www.google.com`, you only need to:

1. Ask the authoritative server for `google.com.`, as it will have the record and you can simply give that the the client. Unfortunately, you don't know who the authoritative server is. But fear not! You can...

2. Ask the authoritative server for `com.` who the authoritative server for `google.com.` is. Unfortunately you don't know who that is either. But keep the faith! You can...

3. Ask the authoritative server for `.` (root) who the authoritative server for `com.` is. Luckily you do know who that is, as we give you the IP address of the root DNS server.

Thus, to handle this client's request, your server will:

1. Send a query to the root nameserver for the `NS` record for `com.`.

2. Send a query to that server for the `NS` record for `google.com.`.

3. Send a query to that server for the `A` record for `www.google.com.`. Take the result that comes back, and send it back to the client.

**IMPORTANT NOTE**: Different DNS server implementations differ in where in the DNS response message they put answers that are `NS` records: some put them in the `AUTHORITY` section and others put them in the `ANSWER` section. To avoid this ambiguity, in this project, we will allow you to put `NS` records in either location and will consider answers equivalent either way. Similarly, some DNS servers will include the `AUTHORITY` records received from the authoritative server and others won't. Outside of requests for `NS` records, including these records is up to you.

### Caching

Your DNS server is required to aggressively cache DNS responses obtained from other servers whenever possible. For example, in the `www.google.com` example above, any other request for a `google.com` domain would require the same first two steps; if your server can remember the responses it got back, it can avoid much of this work in the future. In fact, DNS responses often include additional information in the Authority and Additional sections; **you should aggressively cache those as long as they are in the responding server's bailiwick.**

However, note that DNS records each come with a TTL (time to live), which says how many seconds you can cache them for. Your server must respect this value. Note that your server should not cache error responses (`NXDOMAIN`, `SERVFAIL`, etc).

### Handling Errors

Your client must implement bailiwick checking, ensuring that you only accept `ANSWER`, `AUTHORITY`, and `ADDITIONAL` records that are under the bailiwick of the responding server. In other words, a server for `foo.com` could send you records for `foo.com`, `bar.foo.com`, `baz.blah.foo.com`, etc, but you should reject any records that are included from `bar.com` or `com.foo`.

Your client should gracefully ignore any malformed packets that happen to arrive.

## Allowed Libraries

As noted above, you are only allowed to use DNS libraries based on the functionality they provide. For example, for Python, you are allowed to use the `dnslib` library (`pip3 install dnslib`), which provides packing and unpacking of DNS records as well as parsing of DNS zone files. Any library in any language that provides more functionality than this is **illegal**. Use of unapproved libraries or modules of approved libraries that support illegal functionality will be severely penalized.

<!--
## CS 5700-only Requirements

In this project, we are adding additional requirements for students enrolled in CS 5700. If you are enrolled in CS 4700, you do not need to pass these tests, but are welcome to implement that functionality if desired (though for no extra points). These additional requirements are:

1. Your client is expected to handle packets that may be dropped, retransmititng them as necessary.  Speficially, your `4700dns` is expected to retransmit requests that go unanswered after 1 second, and should do that up to 5 times (i.e., should send a given request up to 6 times to the server, including the original transmission).  If your `4700dns` is never able to receive a reply for a query after all four requests, you should reply to the client with a status code of `SERVFAIL`.

2. Note that between steps 1 and 2, as well as 2 and 3 above, your DNS server is waiting for the response from the other servers.  But this one client's request is not the only thing your DNS server is responsible for; you've got other clients too.  Thus, your DNS server should be architected so that it can handle multiple requests in parallel.  We encourage you to implement an event-driven server (i.e., avoid the use of threads or multiple processes).  This will make your server both more scalable and avoid annoying synchronization issues.
-->

## Implementation Strategy

The starter code provides a simple DNS server that simply sends back the same response for all incoming queries. A suggested order of feature implementation would be:

1.  Write the code necessary to parse the authoritative zone file your program is provided on the command line (if you are using `dnslib` in python, note that it has the ability to do this for you; you are welcome to use that functionality). You should use the SOA record in the zone file to extract the domain that you are the authoritative DNS server for.

2.  Write the code to be able to, using your authoritative zone file data, serve incoming queries for names under your authoritative domain. You should be able to then serve you local zone (don't worry about queries for other domains). Start by being able to serve responses for `A` records, then add support for other record types in your zone. At the end of this step, you should be able to pass the tests with a level less than 10.

3.  Write the code that is able to, in response to an incoming query, make a DNS request to another server, wait and receive the reply, and the forward that reply back to the requesting client. You should start by only being able to request things from the root DNS server (IP address provided to your code on startup). You will also need to properly handle some of the DNS packet flags, including `AA`. At this point, you should be able to pass the level 10 and 11 tests.

4.  Write the code to implement recursive DNS lookups. In other words, your code will need to take in a query, potentially make requests for multiple servers (starting from the root, etc), and then respond back to the original client. Once this is working, you should be able to pass the level 12 and 13 tests.

5.  Then modify this code so that it is able to run multiple client requests in parallel. In other words, you may be waiting for a server to respond on one query, but you can handle other queries in the meantime. After getting this working, you should be able to pass the level 14 test.

6.  Implement robust support for different DNS record types, in combination with all of the features above. You should then be able to pass the level 15 and 16 tests.

7.  Next, implement bailiwick checking by removing from the response, authority, and additional records any records that are outside of the responding server's bailiwick. After getting this working, you should be able to pass the level 17 test.

8.  Implement caching of DNS answers, as well as authority and additional records. Be careful to always respect the TTL, purging items from your cache when they have expired. You should then be able to pass the level 18 test.

9.  Ensure that all of the additional features are working. Be sure you always return an `A` record if an `A` record was requested (e.g., you find there's a `CNAME` you need to resolve). This includes when you receive requests to your authoritative domain. Be sure to set flags appropriately in your DNS responses, and handle cases where you don't get back what you asked (A record for root of subdomain where you get an `NS` record).

<!--
10.  If you are in CS 5700, write the code that can handle timeouts, retransmitting a request to a DNS server multiple times if necessary.  Retry every second, up to 5 times.  You'll also need to make sure your code is able to run multiple client requests in parallel.  In other words, you may be waiting for a server to respond on one query, but you can handle other queries in the meantime.  At this point, you should be able to pass the level 20 and 21 tests.
-->

## Submission

To submit your work, you should submit your (thoroughly documented) code along with a plain-text (no Word or PDF) `README.md` file. In this file, you should describe your high-level approach, the challenges you faced, a list of properties/features of your design that you think is good, and an overview of how you tested your code.

You should submit your source code on Gradescope to the Project 6 project. Be sure to indicate who your teammate is, otherwise, they will not get any credit!

## Grading

The grading in this project will consist of

| Item                | Percentage of Grade |
| ------------------- | ------------------- |
| Program correctness | 100%                |

By definition, you are going to be graded on how gracefully you handle errors; your code should never print out or respond with incorrect data. You should always assume that everyone is trying to break your program. To paraphrase John F. Woods, "Always code as if the [remote machine youâ€™re communicating with] will be a violent psychopath who knows where you live." All student code will be scanned by plagiarism detection software to ensure that students are not copying code from the internet or each other. We will be performing spot-checks on submissions to check for compliance with the project specification and to look for signs of LLM use. An accurate README and well-documented, readable code will be essential for these spot-checks.
