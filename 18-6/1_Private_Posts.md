 
## Lab 1: Accessing Private GraphQL Posts
 
**Technique:** GraphQL introspection and field enumeration
**Target:** Retrieve password from hidden blog post
**Status:** Solved
 
### Context
 
GraphQL is a query language for APIs that lets clients request exactly the fields they need. Unlike REST where endpoints are fixed, GraphQL exposes a single endpoint and the client specifies the shape of the data it wants. This flexibility is also a security concern because GraphQL supports introspection, a built-in feature that lets you query the schema itself to discover all available types, fields, and operations. If introspection is left enabled on a production API, an attacker can enumerate the entire data model.
 
### What I Did
 
I loaded the blog page and opened Burp HTTP History. The blog posts were being fetched through a GraphQL query. I noticed the post IDs coming back were sequential but there was a gap: IDs 1, 2, and 4 were returned but ID 3 was missing. That gap was the tell that something was there but deliberately hidden from the public listing.
 
I sent the GraphQL request to Repeater and ran an introspection query to explore the schema:
 
```graphql
{
  __schema {
    types {
      name
      fields {
        name
      }
    }
  }
}
```
 
Going through the schema I found the `BlogPost` type and noticed it had a field called `postPassword` that was not being requested in the original query. That field was not shown in the public post listing but it existed in the schema and there was nothing preventing me from asking for it.
 
I modified the original query to target ID 3 and added `postPassword` to the fields being requested:
 
```graphql
query getBlogPost($id: Int!) {
  getBlogPost(id: $id) {
    id
    title
    body
    postPassword
  }
}
```
 
Set the variable to `{"id": 3}` and sent it. The response returned the hidden post along with its password. Submitted it to complete the lab.
 
### Why It Works
 
The server was filtering post ID 3 out of the listing query but the `getBlogPost` query by ID had no such restriction. The field `postPassword` existed in the schema and was never marked as restricted. Introspection revealed it existed and nothing prevented it from being queried directly. This is a case where the frontend hid information that the API never actually protected.
 
