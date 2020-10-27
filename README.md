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

## Upload to Dgraph
1. Use Docker to get unreleased Dgraph version which has Live Loader support for Slash.
2. Path to schema and RDF files has to be absolute.
3. Alter URL: add `.grpc` to domain, append port `443` and remove `https://`. Eg. `my-endpoint.grpc.ap-south-1.aws.cloud.dgraph.io:443`.
4. Change Slash mode to flexible.
```sh
# local ingestion
dgraph live -z localhost:5080 -a localhost:9080 --files sql.rdf --format=rdf --schema schema.txt

# On Slash
docker run -it --rm -v /Users/hp/Documents/dgraph-hackathon/SQL-dump/dgraph-migration:/tmp/ dgraph/dgraph:v20.07-slash \
  dgraph live --slash_grpc_endpoint=rural-money.grpc.ap-south-1.aws.cloud.dgraph.io:443 -f /tmp/sql.rdf  --schema /tmp/schema.txt -t <API-KEY>

docker run -it --rm -v /Users/hp/Documents/dgraph-hackathon/SQL-dump/live-loader-schema-test:/tmp/ dgraph/dgraph:v20.07-slash \
  dgraph live --slash_grpc_endpoint=fluent-breath.grpc.ap-south-1.aws.cloud.dgraph.io:443 -f /tmp/sql.rdf -t <API-KEY>
```

## Admin endpoints
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
11. Remove `engine=myISAM`. This engine does not support foreign keys, that's why connections were not being generated.
12. Change `CharID` `Decius Brutus` to `Decius-Brutus`

## Generate schema after feeding data
```
<Chapters.Chapter>: int .
<Chapters.ChapterID>: int .
<Chapters.Description>: string .
<Chapters.Section>: int .
<Chapters.WorkID>: [uid] .
<Characters.Abbrev>: string .
<Characters.CharID>: string .
<Characters.CharName>: string .
<Characters.Description>: string .
<Characters.SpeechCount>: int .
<Characters.Works>: string .
<Genres.GenreName>: string .
<Genres.GenreType>: string .
<Paragraphs.Chapter>: int .
<Paragraphs.CharCount>: int .
<Paragraphs.CharID>: [uid] .
<Paragraphs.ParagraphID>: int .
<Paragraphs.ParagraphNum>: int .
<Paragraphs.ParagraphType>: string .
<Paragraphs.PhoneticText>: string .
<Paragraphs.PlainText>: string .
<Paragraphs.Section>: int .
<Paragraphs.StemText>: string .
<Paragraphs.WordCount>: int .
<Paragraphs.WorkID>: [uid] .
<Quotations.Location>: string .
<Quotations.QuotationID>: int .
<Quotations.QuotationText>: string .
<Works.Date>: int .
<Works.GenreType>: [uid] .
<Works.LongTitle>: string .
<Works.ShortTitle>: string .
<Works.Source>: string .
<Works.Title>: string .
<Works.TotalParagraphs>: int .
<Works.TotalWords>: int .
<Works.WorkID>: string .
<dgraph.cors>: [string] @index(exact) @upsert .
<dgraph.graphql.schema>: string .
<dgraph.graphql.xid>: string @index(exact) @upsert .
type <dgraph.graphql> {
	dgraph.graphql.schema
	dgraph.graphql.xid
}


type <Genres> {
    Genres.GenreName
    Genres.GenreType
}
```

# Access live loaded data in Slash
1. Create schema in Slash. Use `@id` for primary key
```graphql
type Student {
    studentId: String! @id
    name: String
}
```
2. Add `<dgraph.type>` predicate to `sql.rdf`. Eg.
```rdf
_:Student.1 <Student.studentId> "1" .
_:Student.1 <Student.name> "Jack" .
_:Student.1 <dgraph.type> "Student" .
_:Student.2 <Student.studentId> "2" .
_:Student.2 <Student.name> "John" .
_:Student.2 <dgraph.type> "Student" .
```

## Character to Work foreign key issue
```
Create  table `shakespeare_clean`.`Character` with foreign key constraint failed. There is no index in the referenced table where the referenced columns appear as the first columns near 'FOREIGN KEY (`workId`) REFERENCES `Work`(`workId`)
```

```
SQL query

ALTER TABLE `Character`
  ADD FOREIGN KEY (`workId`) REFERENCES `Work` (`workId`)

MySQL said: Documentation

#1452 - Cannot add or update a child row: a foreign key constraint fails (`shakespeare_clean`.`#sql-2a4_22c`, CONSTRAINT `#sql-2a4_22c_ibfk_1` FOREIGN KEY (`workId`) REFERENCES `Work` (`workId`))
```