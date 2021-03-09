# Basic PHP-MySQLi templates

In this guide, the chosen format is MySQLi prepared statements and object-oriented style.

## Define database connection credentials

PHP constant are used. Other methods could be used, such as the [singleton pattern](https://en.wikipedia.org/wiki/Singleton_pattern) with a single app class instance.

```php
define('DB_HOST', 'localhost');
define('DB_USER', 'gesi');
define('DB_PASSWORD', 'gesigesi');
define('DB_NAME', 'gesidemo');
```

## Connect to the database

```php
try {
    $db_conn = new \mysqli($host, $user, $password, $name);
} catch (\mysqli_sql_exception $e) {
    throw new \Exception("Error connecting to the database.", 0, $e);
}
```

Optionally, set connection charset.

```php
try {
    $this->bbdd_con->set_charset("utf8mb4");
} catch (\mysqli_sql_exception $e) {
    throw new \Exception("Error configuring database charset.", 1);
}
```

## In-line operations

### Select data without parameters

The result, stored in `$records` will be an array of arrays with the selected properties.

```php
$query = <<< SQL
SELECT 
    property1,
    property2,
    property3,
    ...
FROM
    table
SQL;

$statement = $db_conn->prepare($query);
$statement->execute();
$resultado = $statement->get_result();
$records = array();

while ($record = $resultado->fetch_array()) {
    $records[] = $record;
}

$statement->close();
```

### Select data with parameters and limit to one result row

The result, stored in `$record` will be an array with the selected properties.

```php
$query = <<< SQL
SELECT 
    property1,
    property2,
    property3,
    ...
FROM
    table
WHERE
    propertyA = ?,
    propertyB = ?
LIMIT 1
SQL;

$statement = $db_conn->prepare($query);
$statement->bind_param('is', $paramA, $paramB);
$statement->execute();
$result = $statement->get_result();

$record = $result->fetch_array();

$statement->close();
```

### Check if data exists

The result, stored in `$exists`, will indicate if there is at least one row with the criteria specified.

```php
$query = <<< SQL
SELECT
    property1
FROM
    table
WHERE
    propertyA = ?
LIMIT 1
SQL;

$statement = $db_conn->prepare($query);
$statement->bind_param('i', $paramA);
$statement->execute();
$statement->store_result();

$exists = $statement->num_rows > 0;

$statement->close();
```

### Insert data

This function returns the identifier of the inserted row.

```php
$query = <<< SQL
INSERT
INTO
    table
    (
        property1,
        property2,
        property3,
        ...
    )
VALUES
    (?, ?, ?, ...)
SQL;

$statement = $db_conn->prepare($query);

$statement->bind_param(
    'sss...', 
    $property1,
    $property2,
    $property3,
    ...
);

$statement->execute();

$inserted_id = $bbdd->insert_id;

$statement->close();
```

## Object-oriented operations

### Object construction from `mysqli_result::fetch_object`

This will be used as a standarized function to construct objects from the result object of a MySQLi query.

```php
public static function constructFromMysqlFetch(stdClass $o): self
{
    return new self(
        $o->property1,
        $o->property2,
        $o->property3,
        ...
    );
}
```

### Select data without parameters

```php
public static function dbSelect(): array
{
    global $db_conn;

    $query = <<< SQL
    SELECT 
        property1,
        property2,
        property3,
        ...
    FROM
        table
    SQL;

    $statement = $db_conn->prepare($query);
    $statement->execute();
    $resultado = $statement->get_result();
    $records = array();
    
    while ($record = $resultado->fetch_object()) {
        $records[] = self::constructFromMysqlFetch($record);
    }
    
    $statement->close();

    return $records;
}
```

### Select data with parameters and limit to one result row

```php
public static function dbSelect(int $paramA, string $paramB): self
{
    global $db_conn;

    $query = <<< SQL
    SELECT 
        property1,
        property2,
        property3,
        ...
    FROM
        table
    WHERE
        propertyA = ?,
        propertyB = ?
    LIMIT 1
    SQL;

    $statement = $db_conn->prepare($query);
    $statement->bind_param('is', $paramA, $paramB);
    $statement->execute();
    $result = $statement->get_result();
    
    $record = self::constructFromMysqlFetch($result->fetch_object());
    
    $statement->close();

    return $record;
}
```

### Check if data exists

```php
public static function dbExists(int $paramA): bool
{
    global $db_conn;

    $query = <<< SQL
    SELECT
        property1
    FROM
        table
    WHERE
        propertyA = ?
    LIMIT 1
    SQL;

    $statement = $db_conn->prepare($query);
    $statement->bind_param('i', $paramA);
    $statement->execute();
    $statement->store_result();
    
    $exists = $statement->num_rows > 0;
    
    $statement->close();

    return $exists;
}
```

### Insert data

This function returns the identifier of the inserted row.

```php
public function dbInsert(): int
{
    global $db_conn;

    $query = <<< SQL
    INSERT
    INTO
        table
        (
            property1,
            property2,
            property3,
            ...
        )
    VALUES
        (?, ?, ?, ...)
    SQL;

    $statement = $db_conn->prepare($query);
    
    // Extract the data from the implicit parameter, if necessary.
    $property1 = $this->getProperty1();
    $property2 = $this->getProperty2();
    $property3 = $this->getProperty3();
    // ...

    $statement->bind_param(
        'sss...', 
        $property1,
        $property2,
        $property3,
        ...
    );

    $statement->execute();
    $inserted_id = $bbdd->insert_id;
    $statement->close();
    
    // Inject the inserted identifier, if necessary.
    $this->id = $inserted_id;

    return $inserted_id;
}
```

### Data access object (DAO) pattern

When using object-oriented programming, using the data access object (DAO) pattern is recommended. In this case, we implement an interface, and all the different classes that access the database must implement it.

```php
interface DAO
{
    
    /*
     *
     * INSERT operations.
     *  
     */

    /**
     * Inserts and object in the database.
     * 
     * @param static $this Object to insert.
     * 
     * @return int         Identifier of the inserted object.
     */
    public function dbInsert(): int;

    /*
     *
     * SELECT operations.
     *  
     */

    /**
     * Retrieves an object from the database.
     * 
     * @param int $id Identifier of the object to retrieve.
     * 
     * @return static Retrieved object.
     */
    public static function dbRetrieve(int $id);

    /**
     * Checks if there is an object in the database with the specified identifier.
     * 
     * @param int $id Identifier of the object.
     * 
     * @return bool   True if the object exists, else false.
     */
    public static function dbExistsId(int $id): bool;

    /**
     * Retrieves all objects from the database.
     * 
     * @return array List with all the objects.
     */
    public static function dbGetAll(): array;

    /**
     * Checks if an object meets al the restrictions to be deleted, i. e.
     * it is not being referenced as foreign key in a different table.
     * 
     * @requires      The object exists.
     * 
     * @param int $id Identifier of the object.
     * 
     * @return array  Array containing the names of the tables which reference
     *                the object, if any. Else, empty array.
     */
    public static function dbCheckDeleteConstraints(int $id): array;

    /*
     *
     * UPDATE operations.
     *  
     */

    /**
     * Updates an object in the database.
     * 
     * @requires           The object exists.
     * 
     * @param static $this Object to update.
     * 
     * @return bool        Statement execution result.
     */
    public function dbUpdate(): bool;

    /*
     *
     * DELETE operations.
     *  
     */

    /**
     * Deltes an object from the database.
     * 
     * @requires      The object exists.
     * 
     * @param int $id Identifier of the object.
     * 
     * @return bool   Statement execution result.
     */
    public static function dbEliminar(int $id): bool;
}
```
