
## Lab 2: Accidental Exposure of Private GraphQL Fields
 
**Technique:** GraphQL introspection to find exposed credential fields, then user ID enumeration
**Target:** Retrieve admin credentials and delete carlos
**Status:** Solved
 
### Context
 
This lab involved a GraphQL API where sensitive fields were unintentionally exposed through the schema. The developer probably intended for some fields to be internal only but left them queryable. Combined with the ability to enumerate user IDs, this gave me a path to administrator credentials without needing to brute force anything.
 
### What I Did
 
I went to the login page and attempted to log in with test credentials. Intercepted the request in Burp and saw it was a GraphQL mutation. I sent it to Repeater and ran an introspection query on the endpoint to map out what was available.
 
I sent the introspection query response to the site map for easier reading. Going through the available operations I found a `getUser` query that returned a user object. Looking at the fields on the user type I found both `username` and `password` were exposed.
 
I built a query to test user IDs:
 
```graphql
query {
  getUser(id: 1) {
    username
    password
  }
}
```
 
ID 1 came back as the administrator account with the password in plaintext in the response. I used those credentials to log in through the normal login form and then navigated to the admin panel and deleted carlos.
 
### Why It Works
 
The `password` field on the user type should never have been included in the schema in a form clients could query. Once introspection reveals it exists, there is no further barrier to requesting it. The pattern of checking user ID 1 for an admin account is a reasonable starting assumption since most applications create the admin account first.
 
