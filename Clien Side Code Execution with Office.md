# Client Side Code Execution with Office
Gaining remote code execution can be done with a variety of methods. One of those methods is taking advantage of vulnerable software running on the endpoint. Though, it is hard to find vulnerabilities since patches are continuously being ruled out, and even if we find a vulnerability we have to develop a working exploit in order to successfully gain a foothold.  

So, to gain remote code execution without relying on software  vulnerabilities we will attempt abuse common features and functionalities provided by software running on the endpoint. Specifically, the goal of this module is to exploit the Microsoft Office suite to execute malicious code.

## The Client-Side Attack Scenario
To initiate a client-side attack, an attacker often delivers a _Trojan_ (for example, in the form of an innocent looking Word or PDF document). Traditional droppers embed an entire payload into the Trojan, but almost all malware families are tending to rely on complex _Droppers_ that use a _staged_ payload that connects back to a C2 to download the second stage.

Once the code has been delivered, it may be written directly to the hard disk or run directly from memory (This is what knows as **In-Memory Execution**).  

### Staged vs Non-staged Payload
There are two types of payloads, they are as follows -
- **Non-staged payloads** is a self-contained of code instruction that serve to perform some actions. For example the Metasploit `windows/tcp_reverse_shell` is non-staged payload to open up a reverse command shell and expose to the attacker's machine. This type of payloads are really just fire-and-forget.

- On the other hand, **Staged payloads** allow for reusability using different stages. These payloads contain a minimal amount of code that perform a _callback_, then retrieves the remaining code. This is usually the preferred type for malware developers since it allows for reusability and helps to avoid detection since it is smaller compared to a non-staged payload.

To generate a non-staged payload in Metasploit, we use:
```
windows/x64/meterpreter_reverse_https
```
and to generate a staged payload, 
```
windows/x64/meterpreter/reverse_https
```

**Note**: "`_`" for non-staged, and "`/`" for staged payloads.

### Building our first Dropper





