# Records, Values, and Refs

| Word       | Definition                             |
| ---------- | -------------------------------------- |
| **Value**  | A distinct element of data.            |
| **Ref**    | An identifier.                         |
| **Record** | A value with an associated identifier. |

## Anatomy of a Ref
For instance, a value might be the number `4` or a record `{"name": "Jordan", "age": 4}`. The reference itself cannot be much more than some form of identifier. In Omega, the standard identifier is a uuid and a revision number. The revisions are intended to keep track of conflicting information. For instance, if two different nodes were to attempt to create a revision `n` of the same `id`, then the engine would detect a conflict of information.
```json
{
	"type": "record",
	"ref": {
		"type": "ref",
		"id": "878e0a7a-dfaf-4277-b44f-2cbe6fea94e8",
		"rev": 0,
	},
	"data": {
		"type": "long",
		"value": 4
	}
}
```

## Records inherit the logic of their values
Refs can be created with `ref` and dereferenced with `deref`. Two refs can only be considered equal if they share the same id, rev, and data. Refs and values are never considered equal. They are different data types.

## Revisions of a ref
```clojure
(deref Ref V)
(+ V 1 Revision)
(revise! Ref Revision)
```

New revisions can be created with `revise!`.