
## Lab 3: Finding a Hidden GraphQL Endpoint
 
**Technique:** Endpoint discovery, introspection filter bypass via newline injection, user deletion via mutation
**Target:** Delete carlos
**Status:** Solved
 
### Context
 
This lab did not expose the GraphQL endpoint at a standard path and had introspection disabled with a regex-based filter. Both of these are common attempts to secure a GraphQL API. Neither was sufficient here. The endpoint was discoverable through common path guessing and the introspection filter was bypassable because regex filters on GraphQL queries are fragile.
 
### What I Did
 
I manually tested common paths. Sending a request to `/api` returned a GraphQL-related error message rather than a 404, which confirmed something was there. I verified it was a live GraphQL endpoint with the minimal type probe:
 
```graphql
query{__typename}
```
 
Got back `{"data":{"__typename":"query"}}`. Endpoint confirmed.
 
I tried a standard introspection query next and got a blocked response. The error indicated the server was rejecting requests matching a pattern. My assumption was it was a simple string match or regex on `__schema`. I tested adding a newline character after `__schema`:
 
```graphql
{
  __schema
  \n
  {
    types {
      name
    }
  }
}
```
 
The modified query bypassed the filter and returned the full schema. The newline broke the pattern the regex was matching against while the GraphQL parser itself ignored the whitespace and processed the query normally.
 
I saved the schema to the site map and went through the available queries. Found a `getUser` query which I used to enumerate user IDs until I identified carlos at ID 3. I then found the `deleteOrganizationUser` mutation:
 
```graphql
mutation {
  deleteOrganizationUser(input: {id: 3}) {
    user {
      username
    }
  }
}
```
 
Sent it. Carlos was deleted.
 
### The Filter Bypass
 
Regex-based filters on GraphQL are fundamentally unreliable because the query language allows arbitrary whitespace and formatting while remaining semantically identical. A filter that looks for `__schema{` will miss `__schema\n{`. A filter that looks for the word `introspection` misses the actual introspection syntax entirely. The only reliable way to block introspection is to disable it at the parser level in the GraphQL server configuration, not to filter the incoming query string.
 
