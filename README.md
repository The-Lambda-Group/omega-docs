# OmegaDB
Omega, also known as OmegaDB, is a logic database that represents rules, facts, and JSON documents in a distributed filesystem. Omega can evaluate statements composed of rules and facts about its data allowing programmers to build complex software systems simply without the use of any algorithms.

## Syntax

### Documents

The most basic Omega data types are documents. These are JSON objects of any valid JSON form.

```clj
[1, 2, "Three", {"four": "five"}]
456
12.6778
{"true": false}
```

### Terms

Terms represent statements that are either true or false. By convention, the object of the term will be the first.

```clj
(blue {"name": "Sky"})
(blue {"name": "Car", "owner": "Dave"})
(even 2)
(odd 2)
(- 6 1 5)
```

For instance, the first term reads as, the object `{"blue": "Sky"}` is `blue`. The rules and facts stored in Omega are intended to enforce the meaning of the symbol `blue`, but the users are free to assign any rules to these symbols as they wish. The last term, `(- 6 1 5)` by convention should read as `6 - 1 = 5`, or the *the difference between 6 and 1 is 5*. This is the order any terms should be read be default, but this is not enforced anywhere in the database engine, but idiomatically by the users of the query langauge.

Terms may either be true or false, for instance `(even 2)` we, would expect to be a true term, while `(odd 2)` we expect to be a false term, because we would read these conventionally as *2 is even* and *2 is odd*. The first statement is true, and the second is false, so the database engine should evaluate these terms as true and false respectively.

### Logical Connectives

Omega can process logic using the logical connectives `and`, `or`, `not`, `every`, and `exists`.

```clj
(and (= 1 X)
     (+ X 2 Y))

(and (or (= 1 X)
         (= 2 X))
     (+ X 1 Y))

(and (or (= X 1)
         (= X 2))
     (not (= X 2)
          (+ X 1 X)
          (= X 1)))

(and (or (= X 2)
         (= X 4))
     (every (even X))
     (+ X 1 Y))

(and (or (= X 2)
         (= X 3))
     (exists (even X))
     (+ X 1 Y))

```

### Rules

Rules can be asserted into the runtime with the `(assert! ...Rules)` functor. Anything passed into this functor will be asserted as a true fact into the current context of the query. In this query, the rule `(:- (mortal X) (man X))` should be read as *if X is a man, then X is mortal*. After the rules and facts have been asserted into the query, the term `(mortal Socrates)` will then be evaluated as true in that query.

```clj
(assert! 
 (:- (mortal X)
     (man X))
     
 (man Socrates))

(mortal Socrates)
```

## Omega Host API

Omega me be hosted at any URL. For instance, `https://subdomain.domain.com/path/to/host` could be the URL for a host. Hosts are a JSON REST API.

### Query Method

The query method is located at `oql/query` on an Omega host you wish to query. It's a post method where the OQL query text is passed as the form body and an optional `PublicKey` header may be passed in with a public RSA key that will be used to authenticate the identity of the user.

```bash
curl \
    -X POST \
    -H "PublicKey: {public-key}" \
    -d @queries/query.oql \
    https://subdomain.domain.com/path/to/host/oql/query
```

## Datastores

Datastores in Omega are located at the `ds` path on the host. For instance, `https://subdomain.domain.com/path/to/host/ds` is the root datastore for the `https://subdomain.domain.com/path/to/host` host. All other datastores will be located in paths under the root.

### Methods

```bash
curl \
    -X POST \
    -d '{"datasore": "albums"}' \
    https://subdomain.domain.com/path/to/host/oql/functors

{
    "result": {
        "type": "json-array",
        "value": [
            {
                "type": "reference-value",
                "id": "XXXXX",
                "rev": "YYYYY",
                "value": {
                    "type": "functor",
                    "name": "even-integer",
                    "arity": 1
                }
            }
        ]
    }
}
```

All data will return serialized OQL data. As an OQL query, this would be:

```clj
(datastore AlbumsDS
           "https://subdomain.domain.com/path/to/host/ds/albums")

(AlbumsDS/query
 (functor Functor Name Arity))

(contains Functors Functor)

(return Functors)
```

### Viewing / Editing Datastores

The `(datastore Symbol Url)` functor binds a symbol to the datastore at a given URL.  

```clj
(datastore AlbumsDS
           "https://subdomain.domain.com/path/to/host/ds/albums")

(AlbumsDS/query 
 (destruct Album {"artist-name": "The Eagles"}))

(merge Album
       {"artist-is-hall-of-fame": true}
       Revision)

(AlbumsDS/revise! _ Album Revision)

(AlbumsDS/insert!
 _
 {"artist-name": "The Beatles",
  "year": 1969,
  "artist-is-hall-of-fame": true,
  "album-name": "Abbey Road"})

(destruct Album
          {"year" 2005})

(AlbumsDS/delete! _ Album)
```

Once a symbol is bound by a datastore, facts and rules in that datastore may be accessed as a path on that symbol such as `AlbumsDS/insert!`. Datastores all implement the functors `(insert! Binding Value)`, `(revise! NewBinding OldBinding Revision)`, `(query ...Terms)`, and `(delete! NewBinding OldBinding)`. Together these make up how datastores are viewed, and edited.

### Serializing

Omega objects all serialize into a final JSON form. This is the actual representaion in the database itself. Serialization can be accessed in either direction with the `(serialize Deserialized Serialized)` functor.

```clj
(datastore DS "https://subdomain.domain.com/path/to/host/ds/abc123")

(serialize {"name": "Dave"}
           {"type": "json-object",
            "value": {"name": {"type": "String",
                               "value": "Dave"}}})

(serialize DS
           {"type": "datastore",
            "url": "https://subdomain.domain.com/path/to/host/ds/abc123"})

(DS/insert! Four 4)

(serialize Four
           {"type": "reference-value",
            "id": "XXXXXXXXXXX",
            "revision": "YYYYYYYYY",
            "value": {"type": "long",
                      "value": 4}})

(serialize (= 1 2)
           {"type": "term",
            "functor": {"type": "functor",}
                        "name": "=",
                        "arity": 2})
```

### Pulling and Pushing

You can pull a datastore locally for more convenient manipulation before ultimately pushing to the upstream.

```clj
(datastore Origin "https://subdomain.domain.com/path/to/host/ds/abc123")
(datastore Local "/abc123")
(Local/pull! Origin)
(Local/insert! _ 4)
(Local/insert! _ 5)
(Local/push! Origin)
```

This code pulls a copy of `https://subdomain.domain.com/path/to/host/ds/abc123` locally, writes two objects, then pushes up the changes in batch.

### Stream Pulling

An alternative to a single pull or even a push are streams. Adding a stream into a datastore will constantly insert new records as they come into the stream.

```clj
(datastore Origin "https://subdomain.domain.com/path/to/host/ds/abc123")
(datastore Local "/abc123")
(Local/stream Upstream)
(Origin/stream Downstream)
(Local/insert! Upstream)
(Origin/insert! Downstream)
```

In this example, any changes to the remote or the origin will automatically be reflected in eachother.

## Atomic Transactions

Transactions that should be atomic can use the atomic keyword to be added all together or not at all.


## Docker Client

### Host

You can run an Omega host locally with.

```
docker run -p 6422:6422 omegadb/omega host
```

You can connect to an Omega host with a remote console using.

```
docker run -p 6422:6422 omegadb/omega client
```

# See Also

- [LeetCode Problems](LeetCode.md)
- [Einstein's Riddle](Einstein.md)
