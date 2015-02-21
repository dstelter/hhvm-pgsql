## Postgres Extension for HipHop

This is an implementation of the `pgsql` and `pdo_pgsql` PHP extensions for
[HHVM][fb-hphp].

### Prerequisites

To run, this extension only requires the HipHop VM itself and the `libpq`
library distributed with Postgres. Building it requires the HHVM header files,
the HHVM CMake files and the `libpq` header files. These can usually be
installed with the `hhvm-dev` and `libpq-dev` packages.

### Pre-built versions

Pre-built versions of this extension are available in the
[releases][pr-releases] branch.

### Building & Installation

Building requires the `hhvm-dev` and `libpq-dev` packages to be installed. Once
they have been installed, the following commands will build the extensions.

~~~
$ cd /path/to/extension
$ hphpize
$ cmake .
$ make
~~~

This will produce a `pgsql.so` file, the dynamically-loadable extension.

To enable the extension, you need to have the following section in your hhvm
config file:

Hdf format:

~~~
DynamicExtensionPath = /path/to/hhvm/extensions
DynamicExtensions {
	* = pgsql.so
}
~~~

INI format:

~~~
hhvm.dynamic_extension_path = /path/to/hhvm/extensions
hhvm.dynamic_extensions[pgsql] = pgsql.so
~~~

Where `/path/to/hhvm/extensions` is a folder containing all HHVM extensions, and
`pgsql.so` is in it. This will cause the extension to be loaded when the virtual
machine starts up.

### PostgreSQL types

By default, all the data retrieved by pg\_fetch\_\* functions are strings. This
is the same behavior as in the standard Zend implementation. However, you can
change this by setting the `TypedResults` option.

~~~
PGSQL {
  TypedResults = false
}
~~~

If you set this option to true, then the type of each column will be an
equivalent of the original PostgreSQL type. So, for example:

```php
// The connection has already been established.

$ret = pg_query($connection, 'SELECT * FROM example');
$row = pg_fetch_assoc($ret);
var_dump($row);

// => It outputs the following:
//
// array(3) {
//   ["id"]=>
//   int(1)
//   ["name"]=>
//   string(13) "A test string"
//   ["valid"]=>
//   bool(true)
// }
```

The supported types are the following (the types not listed below will be
converted to strings):

| PostgreSQL                     | PHP    |
|--------------------------------|--------|
| boolean                        | bool   |
| smallint, integer, bigint      | int    |
| decimal, numeric               | string |
| real, double precision         | float  |
| smallserial, serial, bigserial | int    |

Moreover, this also works when inserting/updating rows. A common pitfall is
doing the following:

```php
// The connection has already been established.

pg_prepare($connection, 'query', 'INSERT INTO test(id, valid) VALUES($1, $2)');
$res = pg_execute($connection, 'query', [1, false]);
var_dump($res); // => outputs: bool(false)
```

The previous example fails because converting `false` into a string results to
an empty string, which is not a valid boolean format in PostgreSQL. As
explained [here](https://bugs.php.net/bug.php?id=44791), this is the proper
behavior. However, this is not what we want if `TypedResults = true`. If this
option is set to true, then the boolean value will be converted as expected by
PostgreSQL. Therefore, the previous example executes successfully with this
option set to true.

### Hack Friendly Mode

If you are using Hack, then you can use the provided `pgsql.hhi` file to type
the functions. There is also a compile-time option that can be passed to cmake
that makes some minor adjustments to the API to make the Hack type checker more
useful with them. This mostly consists of altering functions that would normally
return `FALSE` on error and making them return `null` instead. This takes
advantage of the nullable types in Hack.

To enable Hack-friendly mode use this command instead of the `cmake` one above:

~~~
$ cmake -DHACK_FRIENDLY=ON .
~~~

### Running the tests

In order to run the test suite you just have to call the `test.sh` file. There
are three different ways to call this script:

~~~
$ ./test.sh
$ ./test.sh ExecuteTest
$ ./test.sh ExecuteTest#testNotPrepared
~~~

The first command will run the whole test suite. The second command will just
execute the tests under the `ExecuteTest` class. The third command will execute
the test named `testNotPrepared` that is inside the `ExecuteTest` class.


### Differences from Zend

There are a few differences from the standard Zend implementation.

* The connection resource is not optional.
* The following functions are not implemented for various reasons:
  * `pg_convert`
  * `pg_copy_from`
  * `pg_copy_to`
  * `pg_insert`
  * `pg_lo_close`
  * `pg_lo_create`
  * `pg_lo_export`
  * `pg_lo_import`
  * `pg_lo_open`
  * `pg_lo_read_all`
  * `pg_lo_read`
  * `pg_lo_seek`
  * `pg_lo_tell`
  * `pg_lo_unlink`
  * `pg_lo_write`
  * `pg_meta_data`
  * `pg_put_line`
  * `pg_select`
  * `pg_set_client_encoding`
  * `pg_set_error_verbosity`
  * `pg_trace`
  * `pg_tty`
  * `pg_untrace`
  * `pg_update`

There is a connection pool you can use with the `pg_pconnect` function.

The `$connection_type` parameter is ignored for both `pg_connect` and
`pg_pconnect`.

There are a few new function:

* `pg_connection_pool_stat`: It gives some information, eg. count of
connections, free connections, etc.
* `pg_connection_pool_sweep_free`: Closing all unused connection in all pool.

The `pg_pconnect` function creates a different connection pool for each
connection string.

The `pg_fetch_object` function only supports returning `stdClass` objects.

Otherwise, all functionality is (or should be) the same as the Zend
implementation.

As always, bugs should be reported to the issue tracker and patches are very
welcome.

