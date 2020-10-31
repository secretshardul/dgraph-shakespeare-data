# Complete works of Shakespeare: GraphQL schema and DGraph RDF dump
![](https://media-fastly.hackerearth.com/media/hackathon/slash-sprint/images/7af8fd52f7-cover_image_1.png)

This repository holds the GraphQL schema and RDF file needed to setup a Shakespeare database on Slash. For a sample app built with this data, look at my [Complete Shakespeare Alexa skill on Github](https://github.com/secretshardul/complete-shakespeare).

![Schema visualized](https://lucid.app/publicSegments/view/e6bbf02e-864b-4ce4-85f2-5dc13a843c5a/image.png)

## Steps to push data on Slash
1. [Create a new instance on Slash](https://slash.dgraph.io/).
2. Set the backend mode to flexible and generate an admin API key.
3. Copy schema from [`schema.graphql`](/schema.graphql) and save it in the Slash console.
4. Upload `sql.rdf` to Slash using **live loader**. The command should have the absolute path to this folder.

    ```sh
    docker run -it --rm -v /path/to/dgraph-shakespeare-data/sql.rdf:/tmp/sql.rdf dgraph/dgraph:v20.07-slash \
        dgraph live --slash_grpc_endpoint=<grpc-endpoint>:443 -f /tmp/sql.rdf -t <api-token>
    ```

## How was the data obtained?
These steps are for documentatary purposes. The final `schema.graphql` and `sql.rdf` files have already been prepared.

1. Run a MySQL instance on localhost using XAMPP and PHPMyAdmin.
2. Create new database and import data from `shakespeare.sql`. This is an improved version of the dump provided by Open Source Shakespeare. Unnecessary tables were removed, fields were renamed and foreign keys were added.
3. Run the **SQL to Dgraph migration tool** to obtain `schema.txt` and `sql.rdf` files. Add necessary credentials in `config.properties` file.

    ```sh
    dgraph migrate --config config.properties --output_schema schema.txt --output_data sql.rdf --host 192.168.64.2
    ```

4. By looking at the database, design a `schema.graphql` file as per needs. Add types, authorization, inverse relations and search as necessary.
5. Some regular expression transformations were performed on the generated `sql.rdf` file to support all access patterns- adding `<dgraph.type>`, reverse relations etc. For complete steps, look at this [blog post](TODO).

## Future work
1. In the SQL data, Paragraph table does not have a foreign key for Chapter table. `Paragraph` and `Chapter` graphQL types are yet to be linked.
2. `Quotes` table was separate and not linked to `Works`, `Character` or any other table. The graphQL types are not linked yet.

## Credits
1. [Open source Shakespeare by George Mason University](https://www.opensourceshakespeare.org/downloads/): For making their SQL database open source.
2. [Terence Eden](https://github.com/edent/Open-Source-Shakespeare) and [Richard Morrison](): For improvements in the original SQL dump.
3. [DGraph Labs and DGraph developer community](https://dgraph.io/)
