---
layout: post  
tittle: A nginx server proxy issue   
description:  When app use the nginx as proxy server, there maybe exist response issue of big data from proxyed server
category: operation
---

## How to fix it

Upstream sent too big header while reading response header from upstream. It shows that some file maybe not be complete transmission to client. So we could update the default size of proxy parameters in your app config

    #It's for Terry test
	proxy_buffers           4 32k;
	proxy_busy_buffers_size 64k;
	#End of test

