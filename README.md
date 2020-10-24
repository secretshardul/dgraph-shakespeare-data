# Steps
1. Install Xampp and start PHPMyAdmin.
2. Edit `httpd-xampp.conf` and restart to allow access from localhost
    ```conf
    # since XAMPP 1.4.3
    <Directory "/opt/lampp/phpmyadmin">
        AllowOverride AuthConfig Limit
        Order allow,deny
        Allow from all
        Require all granted
        ErrorDocument 403 /error/XAMPP_FORBIDDEN.html.var
    </Directory>
    ```
3. Create database `leobinus_ossfdt` and import data from dump.
4. Create new user `dgraph_user` with with host `%`. This allows us to access database from command line.
5. Get Dgraph schema and data using migration tool
```sh
dgraph migrate --config config.properties --output_schema schema.txt --output_data sql.rdf --host 192.168.64.2
```
6. Feed data to Dgraph
```sh
dgraph live -z localhost:5080 -a localhost:9080 --files sql.rdf --format=rdf --schema schema.txt
```

## Upload to slash
1. Feed schema
2. Feed JSON using live loader to GRPC endpoint
```
_:Works.12night <Works.Title> "Twelfth Night" .
_:Works.hamlet <Works.Title> "Hamlet" .
```

```sh
docker run -it --rm -v /path/to/g01.json.gz:/tmp/g01.json.gz dgraph/dgraph:v20.07-slash \
  dgraph live --slash_grpc_endpoint=<grpc-endpoint>:443 -f /tmp/g01.json.gz -t <api-token>

# Add .grpc to domain and remove https://
docker run -it --rm -v /Users/hp/Documents/dgraph-hackathon/SQL-dump/:/tmp/ dgraph/dgraph:v20.07-slash \
  dgraph live --slash_grpc_endpoint=xenophobic-anger.grpc.ap-south-1.aws.cloud.dgraph.io:443 -f /tmp/sql.rdf  --schema /tmp/schema.txt -t <api-key>
```

## How to feed into Slash?
1. Upload rdf data to `/mutate` endpoint
```sh
curl -H "Content-Type: application/rdf" -H "x-auth-token: <api-key>" -X POST "<graphql-endpoint>/mutate?commitNow=true" -d $'
{
 set {
    _:x <Person.name> "John" .
    _:x <Person.age> "30" .
    _:x <Person.country> "US" .
 }
}'
```
2. Upload schema: Make mutation to `/admin`. Pass `$sch` as parameter.
```
mutation($sch: String!) {
  updateGQLSchema(input: { set: { schema: $sch}})
  {
    gqlSchema {
      schema
      generatedSchema
    }
  }
}
```

# SQL to Dgraph conversion rules
- SQL data is converted into N-Quad format with the format `<subject> <predicate> object .` where
    1. Subject is UID
    2. Predicate is attribute name (column name)
    3. Object is attribute value
    4. `.` is delimiter
- For example the following table becomes

    ```sh
    # comments
    | Id | PostId | Text                                  | UserId |
    |----|--------|---------------------------------------|--------|
    |  4 |      9 | Of what fabric are the blankets made? |     15 |
    ```

    ```rdf
    _:comments.4 <comments.Id> "4" .
    _:comments.4 <comments.Text> "Of what fabric are the blankets made?" .
    ```
- During data conversion 3 rules are followed
    1. Subject `_:comments.4` includes
        1. `-:` or blank node label: It means object is yet to be created.
        2. Table name
        3. UID
    2. Predicate includes
        1. Table name
        2. Column name of attribute
        Here foreign key predicates like `UserId` are not considered. They will be represented on edges between nodes.

    3. Representing relations: Foreign keys are replaced with blank node labels. The relation between the comment node with post and user nodes is represented as
        ```
        _:comments.4 <comments.PostId> _:posts.9
        _:comments.4 <comments.UserId> _:users.15
        ```
        Here the foreign keys are replaced with blank node labels of a `posts` node and a `users` node.

- Schema generation:
    1. Plain attributes are converted to GraphQL scalar types like `String`, `Float` etc. The row's UID is converted into `Int` format. In RDF types start with lowercase letters.
        ```
        comments.Id: int .
        comments.Text: string .
        ```
    2. Foreign keys are assigned type `[uid]`. It means following predicate leads to another set of objects:
        ```
        comments.PostId: [uid] .
        comments.UserId: [uid] .
        ```

# Migration
1.

## Migration issues and fixes
1. `blob` type column `Notes` in `Works` table: It's empty, drop it.
2. `char` type column `ParagraphType` in `Paragraphs`: Change to text type
3. `mediumInt` type in `TotalParagraphs` and `TotalWords` in `Works`: Change to int
4. `mediumInt` type in `Occurences` in `WordForms`
5. `mediumInt` type in `SpeechCount` in `Characters`
6. `UserAccounts` and `UserAccessLevels` tables: Delete
7. `MediaTypes` and `MediaObjects` tables
8. `mediumInt` type in `WordFormID` in `WordForms`
9. `Works_Searches` table: Delete
10. `WordForms` table: No issue but unnecessary so delete

Cleaned SQL data exported as `leobinus_ossfdt.sql`.