# Database Binder

Class to help with binding to `PDOStatement` objects. Mostly syntax sugar, but not only.

## Binding specific values

Most of the functions are aimed at binding one value and follow the usage pattern like

```php
\Simbiat\Database\Bind::function_name(\PDOStatement $sql, string $binding, mixed $value);
```

where `function_name` is name of the function (listed below), `$sql` is respective `\PDOStatement` object (normally result of `PDO`'s `prepare` statement), `$binding` is a string representing the named parameter to bind to, and `$value` is the value to attempt to bind. `$value` can be of any type, casting will be attempted in respective functions.

### Generic binders

Some binders are just a way to replace use of `\PDO::PARAM_*` variables to allow use of "words" representing the types:

- `bindString` - binds a value as string.
- `bindInteger` - binds a value as integer.
- `bindBoolean` - binds a value as boolean.
- `bindBinary` - binds a value as binary with automatic length calculation (as required by `PDO`).
- `bindNull` - binds a value as null (or rather it ignores the value entirely).

### Custom binders

There are a bunch of binders that provide extra flavor:

- `bindYear`, `bindDate`, `bindTime` and `bindDateTime` - if you have `\Simbiat\SandClock` [class](https://github.com/Simbiat/sand-clock) in your project will use its `format()` method to attempt conversion of the value provided to year (`Y`), date (`Y-m-d`), time (`H:i:s.u`) or datetime (`Y-m-d H:i:s.u`) respectively (DB engines normally recognize these formats as respective data types). Otherwise, will just treat the value as a string.
- `bindBytes`, `bindBits` - if you have `\Simbiat\CuteBytes` [class](https://github.com/Simbiat/cute-bytes) in your project will use its `bytes()` method to convert the value to bytes or bits respectively. Otherwise, will treat the value as a string.
- `bindLike` - same as `bindString`, but will wrap the string in `%` symbols. Useful for `LIKE` queries.
- `bindMatch` - same as `bindString`, but with extra processing and sanitization to allow use of the value in `MATCH` clause in `FULLTEXT` search. `MATCH` has very specific rules and some characters in it mean specific things, so this will attempt to "prepare" the string to a format that will not break the search.

## Binding multiple values

Quite often you do not have just one value to bind. For these cases you can use this:

```php
\Simbiat\Database\Bind::bindMultiple(\PDOStatement $sql, array $bindings = []);
```

This will go through the array of bindings (format explained below), skip those bindings that are not present in the prepared query (works only for named identifiers) and then bind the values using the methods described earlier. Strings will be run through `mb_scrub` before binding.

### Array format

```php
$bindings = [
    #Every key is the name of the parameter identifier that we are binding to
    ':table' =>
        [
            #The first element is the value that we are binding. For "in" expected to be an array. If not an array, will wrap it in one
            $table,
            #The second element is the data type of the element that we want to bind as. For `IN` we should use "in". More details on the types below.
            'in',
            #The third element is semi-required for "in", and is the data type that will be applied to all array elements. If omitted, the value will be treated as a `string`
            'string'
        ],
    ':schema' =>
        [
            $schema,
            'string'
        ]
];
```

### Data types mapping

The second element in the `$bindings` array expects certain values, which are used to map to respective functions as below:

```php
public const array BINDING_HANDLERS = [
    'year' => 'bindYear',
    'date' => 'bindDate',
    'time' => 'bindTime',
    'datetime' => 'bindDateTime',
    'timestamp' => 'bindDateTime',
    'bool' => 'bindBoolean',
    'boolean' => 'bindBoolean',
    'null' => 'bindNull',
    'int' => 'bindInteger',
    'integer' => 'bindInteger',
    'number' => 'bindInteger',
    'limit' => 'bindInteger',
    'offset' => 'bindInteger',
    'str' => 'bindString',
    'string' => 'bindString',
    'text' => 'bindString',
    'float' => 'bindString',
    'varchar' => 'bindString',
    'varchar2' => 'bindString',
    'bytes' => 'bindBytes',
    'bits' => 'bindBits',
    'match' => 'bindMatch',
    'like' => 'bindLike',
    'lob' => 'bindBinary',
    'large' => 'bindBinary',
    'object' => 'bindBinary',
    'blob' => 'bindBinary',
];
```

If a value is not found in the mappings and is an integer, it will be considered to be a `\PDO::PARAM_*` constant and used directly in `bindValue()`. If it's not integer, the first element will be cast into string before being passed to `bindValue()`.

## Binding for `IN` clause

```php
\Simbiat\Database\Bind::unpackIN(string $sql, array $bindings);
```

Can be used to "unpack" a `$bindings` array and prepare each value for future binding by "cloning" the original parameter identifier. Since this requires **changing** of the original SQL, function **requires** passing it a string (by reference), and thus **before** you have prepared it using `prepare()`. It also currently requires the list of **all** bindings to be passed (by reference) because they also need to be modified and because I simply do not see a good way of not doing that.

The query is expected to have the `IN` clause in it, and `$bindings` is expected to have a format same as for `bindMultiple()`. Only named identifiers are supported because otherwise the function can break the order, especially since `+` operator is used instead of `array_merge`. Here's a practical example:
```php
$query = 'SELECT COUNT(*) as `count` FROM `information_schema`.`TABLES` WHERE `TABLE_NAME` IN(:table) AND `TABLE_SCHEMA` IN(:schema);';
$table = ['table', 'table2'];
$schema = 'schema';
$bindings = [
    ':table' =>
        [
            $table,
            is_string($table) ? 'string' : 'in',
            'string'
        ],
    ':schema' =>
        [
            $schema,
            is_string($schema) ? 'string' : 'in',
            'string'
        ]
];
\Simbiat\Database\Bind::unpackIN($sql, $bindings);
echo $sql;
```

This will output the following query:

```sql
SELECT COUNT(*) as `count` FROM `information_schema`.`TABLES` WHERE `TABLE_NAME` IN(:table_0, :table_1) AND `TABLE_SCHEMA` IN(:schema);
```

and bindings would be:

```php
[
':schema' =>
    [
        'schema',
        'string',
        'string'
    ],
':table_0' =>
    [
        'table',
        'string'
    ],
':table_1' =>
    [
        'table2',
        'string'
    ]
];
```

After the bindings are processed and `PDO`'s `prepare()` is run, you can use the `bindMultiple()` method as explained earlier.