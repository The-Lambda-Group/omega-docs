# Values and Refs
**Ref:** An identifier for a value. \
**Value:** A distinct element of data.

## Anatomy of a Ref
For instance, a value might be the number `4` or a record `{"name": "Jordan", "age": 4}`. The reference itself cannot be much more than some form of identifier. In Omega, the standard identifier is a uuid and a revision number. The revisions are intended to keep track of conflicting information. For instance, if two different nodes were to attempt to create a revision `n` of the same `id`, then the engine would detect a conflict of information.
```json
{
	"type": "ref",
	"id": "878e0a7a-dfaf-4277-b44f-2cbe6fea94e8",
	"rev": 0,
	"data": {
		"type": "long",
		"value": 4
	}
}
```

## Refs are not Values
```clojure
(assert!
 (new-ref {"name": "John"} John)
 (= {"name": "John"} OtherJohn))

(!= John OtherJohn)
(deref John JohnValue)
(= OtherJohn JohnValue)
```
Refs can be created with `new-ref` and dereferenced with `deref`. Two refs can only be considered equal if they share the same id, rev, and data. Refs and values are never considered equal. They are different data types.

## Revisions of a ref
```clojure
(deref Ref V)
(+ V 1 Revision)
(revise! Ref Revision)
```

New revisions can be created with `revise!`.