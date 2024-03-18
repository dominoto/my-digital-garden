---
{"dg-publish":true,"permalink":"/resources/failed-to-start-server-on-port-1234/","created":"2024-01-21","updated":"2024-03-18"}
---


**Topics:** ==[[LM Studio\|LM Studio]]==

While playing with local LLMs (Large Language Models) in [[LM Studio\|LM Studio]] on Windows, I got a following error while trying to start LLM server:

```json
{ "cause": "{\"code\":\"EACCES\",\"errno\":-4092,\"syscall\":\"listen\",\"address\":\"0.0.0.0\",\"port\":1234}", "data": { "memory": { "ram_capacity": "33.95 GB", "ram_unused": "13.21 GB" }, "gpu": { "type": "NvidiaCuda", "vram_recommended_capacity": "8.59 GB", "vram_unused": "7.48 GB" }, "os": { "platform": "win32", "version": "10.0.22631", "supports_avx2": true }, "app": { "version": "0.2.10", "downloadsDir": "C:\\Users\\user\\.cache\\lm-studio\\models" }, "model": {} }, "title": "Failed to start server on port 1234" }
```

The solution was to restart **WinNAT** service in the following way:
- Open CMD (command line) as **admin** and run:
	1. `net stop winnat`
	2. `net start winnat`

[Source](https://stackoverflow.com/a/65698209)

### What is WinNAT?

Windows NAT (WinNAT) is a network address translation (NAT) service designed to provide connectivity between virtual machines (VMs) and the external network in a Hyper-V environment. It acts as a router and enables VMs to access external resources using a NAT IP address.