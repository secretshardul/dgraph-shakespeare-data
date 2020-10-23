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
4. Import dump.

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
