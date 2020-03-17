## Magic Link

### 1A

Basic flow:
```
- user enters his login and requests a magic link. With this request, we will send a flag with the client id. This will allow us to send a correct link and prevent interception. We can also send a code challenge to verify it later with the link that we receive. This will require us to store the code challenge and verification in the app.
- a magic link is sent to his email address
- going to the link in the web app logs user in.
```

Let's define some security requirements first:

1. We probably should store hashed tokens instead of plain ones for the case if a perpetrator gets access to the database.
2. Token living time should be long enough for the user to comfortably log in and short enough so that it's not possible to crack it in short amount of time. This way we can also use a lower number of salt rounds for bcrypt so that we can compute more hashes per second.

Now that we send a token that we don't store, there comes a question: how can we verify it? We obviously cannot check against every hash in the db.
We must add a piece of information to the token that will help us find a corresponding record in the database. Now, it could be user id or an arbitrary id, unique for every record.
I personally don't see why not attach the record id. So we'll be sending a link of this sort:

`protocol://magic-link/${record_id}${token}`

Now, our database schema can look like this:
```
record_id — string of fixed length, supposedly hexadecimal.
hash — string
client_id - string
code_challenge - string, sent with the request to create the link
user_id
is_used - boolean
is_invalidated - boolean (for cases where this link needs to be explicitly revoked)
expires - timestamp
```

### 1B

Given that we basically verify against the record id *and* a token, we won't have any collision. This means we don't need a large entropy. Just enough to not be cracked in a reasonable amount of time.

### 1C

I would suppose our query would look like this:
```
record_id = ${record_id}
  and expires > now()
  and is_used = false
  and is_invalidated = false
```
So I suppose our index configuration could be done in 4 simple indices:
```
unique(record_id)
is_used
is_invalidated
expires
```

### 1D
We can issue signed tokens like JWT, and only store signing key. 
There's a public information and we can basically put everything we need there: user_id, expiration date, etc.
Unfortunately, we cannot track if the token has been used or not. But we can make the living time very short.

### 1E

We can use app link scheme, like
`myapp://...`

That would require us to use PKCE in order to prevent interception:
we would send a code challenge with the request to create a magic link.

In order to stay secure, we want to open a browser window with the web app page where the user can ask for the link himself. When a webview with this link is open, a client id and a code challenge are passed as a query. This allows to verify that the user makes this request on their behalf.


## Javascript promises and garbage collection

### 2A

As fetch call returns
`Promise<Response>`, the output of the function the way that it is, is going to be:
```
a 1
b NaN
```
If we add `.then((response) => response.json())` in the middle, we will get:
```
a 1
b 124
```
It doesn't matter if the `content-type: application/json` header is set.

I'm going to reason about why `b` comes last in **2C**

### 2B

The `d` variable is allocated when the `x` is called. Everything that is defined inside `x` has access to `d`. Therefore it is only released when it is unreachable from the root.

The `d` in this example is referenced by `onFulfilled` callback and therefore by the promise produced by fetch. As we don't store the promise anywhere, it will be ready to be sweeped as soon as it's resolved: in this case, once `onFulfilled` callback returns.

Also, we do not need to care about `console.log` calls because `d` is  a number and thus a value type, not a reference type.

### 2C

The expected output will be:
```
a 1
b 235
```

The order will always be the same, because the Promise spec always tells us to not execute `onFulfilled` until the execution context contains platform code only. Ie, it must always be scheduled on the next event loop turn. The VM implementation should be always irrelevant, because there's always at most one thread per agent.