# Query Templates
a new class of cloud service

The following ingredients ought to be sufficient for defining a serverless function

1. The expected input, in the form of OpenAPI, JSON Schema, OData, XSD etc.
2. A database query
3. A templating system such as Slick
4. Database result to XML/JSON mapping, as defined by the [SQL](https://en.wikipedia.org/wiki/SQL/XML) [specification](https://en.wikipedia.org/wiki/SQL:2016)

No general purpose backend language should be necessary.

This example should speak for itself:

```yaml
swagger: "2.0"
info:
  description: "A serverless function for looking up who has acted in a given film"
  version: "1.0.0"
  title: "Actor Lookup"
paths:
  /actors/{film}:
    get:
      summary: "Finds actors by film"
      produces:
      - "application/json"
      parameters:
      - name: "film"
        in: "path"
        description: "film name"
        required: true
        type: "string"
      responses:
        "200":
          description: "successful operation"
          schema:
            type: "array"
            items:
              $ref: "#/definitions/Actor"
        "400":
          description: "Invalid film"
definitions:
  Actor:
    type: "object"
    properties:
      first_name:
        type: "string"
      last_name:
        type: "string"
      f:
        $ref: "#/definitions/Films"
  Films:
    type: array
    items:
      $ref: "#/definitions/Film"
  Film:
    type: object
    properties:
      title:
        type: "string"
```

```sql
SELECT JSON_ARRAYAGG(JSON_OBJECT(
  KEY 'actor' VALUE JSON_OBJECT(
    KEY 'first_name' VALUE actor.first_name,
    KEY 'last_name' VALUE actor.last_name
  ),
  KEY 'TITLE' VALUE title
  ABSENT ON NULL
))
FROM (
  SELECT
    a.first_name AS "actor.first_name", 
    a.last_name AS "actor.last_name", 
    f.title
  FROM actor a
  JOIN film_actor fa ON a.actor_id = fa.actor_id
  JOIN film f ON fa.film_id = f.film_id
  WHERE f.name = $film
  ORDER BY 1, 2, 3
) t
```

which will return
```json
[
    {
        "first_name": "NICK",
        "last_name": "WAHLBERG",
        "f": [
            {
                "title": "SMILE EARRING"
            },
            {
                "title": "WARDROBE PHANTOM"
            }
        ]
    },
    {
        "first_name": "PENELOPE",
        "last_name": "GUINESS",
        "f": [
            {
                "title": "ACADEMY DINOSAUR"
            },
            {
                "title": "ANACONDA CONFESSIONS"
            }
        ]
    }
]
```

query credit: https://blog.jooq.org/2020/05/05/using-sql-server-for-xml-and-for-json-syntax-on-other-rdbms-with-jooq/

## Design questions

* Keep endpoint spec and query in the same file? (Template language is the host language, api spec lives in a comment)
  * Pro: files change together, convenience of passing around a single file, easier for tooling to validate
  * Con: Harder for typical tooling to work with. A full IDE like IntelliJ can do language injection, but basic text editors or code review tools don't

## Implementation TODOs

* Parse API endpoint spec for available parameters
* Create canonical format for available parameters: URI path + URI query params + JSON/XML in POST request body ?
  * flatten?
* Parse template for used parameters
* Validate API endpoint spec against template parameters (subset)
* Validate api spec against actual returned results (warning when running in dev mode is probably fine, static analysis probably overkill)
* build container image from Query Template

## Ideas for version 2.0

* External result mapping for a query language such as ksql that doesn't define its own result mapping
