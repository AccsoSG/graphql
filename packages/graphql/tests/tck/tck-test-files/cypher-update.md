## Cypher Update

Tests Update operations.

Schema:

```schema
type Actor {
    name: String
    movies: [Movie] @relationship(type: "ACTED_IN", direction: "OUT")
}

type Movie {
    id: ID
    title: String
    actors: [Actor]! @relationship(type: "ACTED_IN", direction: "IN")
}
```

---

### Simple Update

**GraphQL input**

```graphql
mutation {
    updateMovies(where: { id: "1" }, update: { id: "2" }) {
        movies {
            id
        }
    }
}
```

**Expected Cypher output**

```cypher
MATCH (this:Movie)
WHERE this.id = $this_id
SET this.id = $this_update_id

RETURN this { .id } AS this
```

**Expected Cypher params**

```cypher-params
{
    "this_id": "1",
    "this_update_id": "2"
}
```

---

### Single Nested Update

**GraphQL input**

```graphql
mutation {
    updateMovies(
        where: { id: "1" }
        update: {
            actors: [
                { where: { name: "old name" }, update: { name: "new name" } }
            ]
        }
    ) {
        movies {
            id
        }
    }
}
```

**Expected Cypher output**

```cypher
MATCH (this:Movie)
WHERE this.id = $this_id
WITH this
OPTIONAL MATCH (this)<-[:ACTED_IN]-(this_actors0:Actor)
WHERE this_actors0.name = $this_actors0_name
CALL apoc.do.when(this_actors0 IS NOT NULL,
  "
    SET this_actors0.name = $this_update_actors0_name
    RETURN count(*)
  ",
  "",
  {this:this, this_actors0:this_actors0, this_update_actors0_name:$this_update_actors0_name}) YIELD value as _

RETURN this { .id } AS this
```

**Expected Cypher params**

```cypher-params
{
    "this_id": "1",
    "this_actors0_name": "old name",
    "this_update_actors0_name": "new name"
}
```

---

### Double Nested Update

**GraphQL input**

```graphql
mutation {
    updateMovies(
        where: { id: "1" }
        update: {
            actors: [
                {
                    where: { name: "old actor name" }
                    update: {
                        name: "new actor name"
                        movies: [
                            {
                                where: { id: "old movie title" }
                                update: { title: "new movie title" }
                            }
                        ]
                    }
                }
            ]
        }
    ) {
        movies {
            id
        }
    }
}
```

**Expected Cypher output**

```cypher
MATCH (this:Movie)
WHERE this.id = $this_id
WITH this
OPTIONAL MATCH (this)<-[:ACTED_IN]-(this_actors0:Actor)
WHERE this_actors0.name = $this_actors0_name
CALL apoc.do.when(this_actors0 IS NOT NULL, "
    SET this_actors0.name = $this_update_actors0_name

    WITH this, this_actors0
    OPTIONAL MATCH (this_actors0)-[:ACTED_IN]->(this_actors0_movies0:Movie)
    WHERE this_actors0_movies0.id = $this_actors0_movies0_id
    CALL apoc.do.when(this_actors0_movies0 IS NOT NULL, \"
        SET this_actors0_movies0.title = $this_update_actors0_movies0_title
        RETURN count(*)
    \",
      \"\",
      {this_actors0:this_actors0, this_actors0_movies0:this_actors0_movies0, this_update_actors0_movies0_title:$this_update_actors0_movies0_title}) YIELD value as _

    RETURN count(*)
  ",
  "",
  {this:this, this_actors0:this_actors0, this_update_actors0_name:$this_update_actors0_name,this_actors0_movies0_id:$this_actors0_movies0_id,this_update_actors0_movies0_title:$this_update_actors0_movies0_title}) YIELD value as _

RETURN this { .id } AS this
```

**Expected Cypher params**

```cypher-params
{
    "this_id": "1",
    "this_actors0_name": "old name",
    "this_actors0_movies0_id": "old movie title",
    "this_actors0_name": "old actor name",
    "this_update_actors0_movies0_title": "new movie title",
    "this_update_actors0_name": "new actor name"
}
```

---

### Simple Update as Connect

**GraphQL input**

```graphql
mutation {
    updateMovies(
        where: { id: "1" }
        connect: { actors: [{ where: { name: "Daniel" } }] }
    ) {
        movies {
            id
        }
    }
}
```

**Expected Cypher output**

```cypher
MATCH (this:Movie)
WHERE this.id = $this_id
WITH this
OPTIONAL MATCH (this_connect_actors0:Actor)
WHERE this_connect_actors0.name = $this_connect_actors0_name
FOREACH(_ IN CASE this_connect_actors0 WHEN NULL THEN [] ELSE [1] END |
MERGE (this)<-[:ACTED_IN]-(this_connect_actors0)
)
RETURN this { .id } AS this
```

**Expected Cypher params**

```cypher-params
{
    "this_id": "1",
    "this_connect_actors0_name": "Daniel"
}
```

---

### Simple Update as Disconnect

**GraphQL input**

```graphql
mutation {
    updateMovies(
        where: { id: "1" }
        disconnect: { actors: [{ where: { name: "Daniel" } }] }
    ) {
        movies {
            id
        }
    }
}
```

**Expected Cypher output**

```cypher
MATCH (this:Movie)
WHERE this.id = $this_id
WITH this
OPTIONAL MATCH (this)<-[this_disconnect_actors0_rel:ACTED_IN]-(this_disconnect_actors0:Actor)
WHERE this_disconnect_actors0.name = $this_disconnect_actors0_name
FOREACH(_ IN CASE this_disconnect_actors0 WHEN NULL THEN [] ELSE [1] END |
DELETE this_disconnect_actors0_rel
)
RETURN this { .id } AS this
```

**Expected Cypher params**

```cypher-params
{
    "this_id": "1",
    "this_disconnect_actors0_name": "Daniel"
}
```

---

### Update an Actor while creating and connecting to a new Movie (via field level)

**GraphQL input**

```graphql
mutation {
    updateActors(
        where: { name: "Dan" }
        update: {
            movies: {
                create: [{ id: "dan_movie_id", title: "The Story of Beer" }]
            }
        }
    ) {
        actors {
            name
            movies {
                id
                title
            }
        }
    }
}
```

**Expected Cypher output**

```cypher
MATCH (this:Actor)
WHERE this.name = $this_name

WITH this
CREATE (this_movies0_create0:Movie)
SET this_movies0_create0.id = $this_movies0_create0_id
SET this_movies0_create0.title = $this_movies0_create0_title
MERGE (this)-[:ACTED_IN]->(this_movies0_create0)

RETURN this { .name, movies: [ (this)-[:ACTED_IN]->(this_movies:Movie)  | this_movies { .id, .title } ] } AS this
```

**Expected Cypher params**

```cypher-params
{
  "this_name": "Dan",
  "this_movies0_create0_id": "dan_movie_id",
  "this_movies0_create0_title": "The Story of Beer"
}
```

---

### Update an Actor while creating and connecting to a new Movie (via top level)

**GraphQL input**

```graphql
mutation {
    updateActors(
        where: { name: "Dan" }
        create: { movies: [{ id: "dan_movie_id", title: "The Story of Beer" }] }
    ) {
        actors {
            name
            movies {
                id
                title
            }
        }
    }
}
```

**Expected Cypher output**

```cypher
MATCH (this:Actor)
WHERE this.name = $this_name

CREATE (this_create_movies0:Movie)
SET this_create_movies0.id = $this_create_movies0_id
SET this_create_movies0.title = $this_create_movies0_title
MERGE (this)-[:ACTED_IN]->(this_create_movies0)

RETURN this { .name, movies: [ (this)-[:ACTED_IN]->(this_movies:Movie) | this_movies { .id, .title } ] } AS this
```

**Expected Cypher params**

```cypher-params
{
  "this_name": "Dan",
  "this_create_movies0_id": "dan_movie_id",
  "this_create_movies0_title": "The Story of Beer"
}
```

---