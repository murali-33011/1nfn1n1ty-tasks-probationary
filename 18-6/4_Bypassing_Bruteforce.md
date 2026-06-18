
## Lab 4: Bypassing GraphQL Brute Force Protections
 
**Technique:** GraphQL alias batching to send multiple login attempts in a single request
**Target:** Find carlos's password
**Status:** Solved
 
### Context
 
The application rate-limited login attempts per request or per time window. A normal brute force would trigger the limit after a small number of attempts. GraphQL aliases let you send multiple operations in a single request by giving each one a different name. Most rate-limiting implementations count requests at the HTTP level, not at the individual operation level within a GraphQL body. Sending 100 login attempts in one request counts as one request.
 
### What I Did
 
I intercepted the login request and confirmed it was using a GraphQL mutation. The mutation took a username and password and returned a `success` field. I confirmed rate limiting was active by sending multiple requests rapidly and getting blocked.
 
I built a batched mutation using aliases, one alias per password from the provided list:
 
```graphql
mutation {
  attempt0: login(input: {username: "carlos", password: "abc123"}) {
    success
  }
  attempt1: login(input: {username: "carlos", password: "password"}) {
    success
  }
  attempt2: login(input: {username: "carlos", password: "letmein"}) {
    success
  }
  ...
}
```
 
Each alias was a separate login attempt with a different password. I sent the entire list in a single HTTP request. The rate limiter saw one request and let it through. In the response I scanned for whichever alias returned `"success": true`, identified the corresponding password, and used it to log in as carlos.
 
### Why It Works
 
Rate limiting at the HTTP layer counts HTTP requests. One HTTP request with 200 GraphQL operations is still one HTTP request from the perspective of a naive rate limiter. GraphQL aliases were designed to let clients fetch the same type of data with different arguments in one round trip. Used here they become a single-packet brute force. The fix requires rate limiting at the operation level within the GraphQL middleware, counting each aliased operation as a separate attempt.
 
