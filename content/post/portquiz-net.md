---
title: "PortQuiz.net"
date: 2020-04-19T13:14:57+03:00
draft: false
---

Recently, I was trying to connect to an Azure SQL database from a client's
Windows Server. The Windows admin was telling me he can connect to any website
from server but not my SQL database. It was evident that only 80 and 443
outbound ports are allowed but I had to prove this to convince him to open a
ticket to network team. I was looking for a public server that listens on a non
regular port. Should I run `nc` on one of my machines? Wouldn't it be nice if
there was a server that listens on all ports?

As a matter of fact, there is: [PortQuiz.net](portquiz.net). It listens on all
TCP ports, so you can test outbound connections with `telnet`, `curl` or `nc`.

```
$ telnet portquiz.net 666
Trying 52.47.209.216...
Connected to portquiz.net.
Escape character is '^]'.

$ curl http://portquiz.net:666
Port 666 test successful!
Your IP: 46.196.19.241

$ nc -v portquiz.net 666
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Connected to 52.47.209.216:666.
```
