<!--stackedit_data:&#10;eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19&#10;-->

## Introduction to SQL/JSON

SQL-2016 standard introduced SQL/JSON data model and path language used by certain SQL/JSON functions to query JSON.   SQL/JSON data model is a sequences of items, each of which is consists of SQL scalar values with an additional SQL/JSON null value,  and composite data structures using JSON arrays and objects. 

PostgreSQL has two JSON  data types - the textual json data type to store an exact copy of the input text and the jsonb data type -   the binary storage for  JSON data converted to PostgreSQL types, according  mapping in   [json primitive types and corresponding  PostgreSQL types](https://www.postgresql.org/docs/current/static/datatype-json.html).  SQL/JSON data model adds datetime type to these primitive types, but it only used for comparison operators in path expression and stored on disk as a string.  Thus, jsonb data is already conforms to SQL/JSON data model, while json should be converted according the mapping.  SQL-2016 standard describes two sets of SQL/JSON functions: constructor functions (JSON_OBJECT, JSON_OBJECTAGG, JSON_ARRAY, and JSON_ARRAYAGG) and query functions (JSON_VALUE, JSON_TABLE, JSON_EXISTS, and JSON_QUERY). Constructor functions use values of SQL types and produce JSON values (JSON objects or JSON arrays) represented in SQL character or binary string types. Query functions evaluate SQL/JSON path language expressions against JSON values, producing values of SQL/JSON types, which are converted to SQL types. 

## SQL/JSON Path language 

The main task of the path language is to specify  the parts (the projection)  of JSON data to be retrieved by path engine for SQL/JSON query functions.  The language is designed to be flexible enough to meet the current needs and to be adaptable to the future use cases. Also, it is integratable into SQL engine, i.e., the semantics of predicates and operators generally follow SQL.  To be friendly to JSON users, the language resembles  JavaScript - dot(```.```)  used for member access and [] for array access, arrays starts from zero (SQL arrays starts from 1).

Example of two-floors house:
```sql
CREATE TABLE house(js) AS SELECT jsonb '
{
  "info": {
    "contacts": "Postgres Professional\n+7 (495) 150-06-91\ninfo@postgrespro.ru",
    "dates": ["01-02-2015", "04-10-1957 19:28:34 +00", "12-04-1961 09:07:00 +03"]
  },
  "address": {
    "country": "Russia",
    "city": "Moscow",
    "street": "117036, Dmitriya Ulyanova, 7A"
  },
  "lift": false,
  "floor": [
    {
      "level": 1,
      "apt": [
        {"no": 1, "area": 40, "rooms": 1},
        {"no": 2, "area": 80, "rooms": 3},
        {"no": 3, "area": null, "rooms": 2}
      ]
    },
    {
      "level": 2,
      "apt": [
        {"no": 4, "area": 100, "rooms": 3},
        {"no": 5, "area": 60, "rooms": 2}
      ]
    }
  ]
}
';
```

 Consider the following path expression:
```sql
'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)'
```
The result of this path expression will be information about apartments with rooms, which area is  in specified range. Dollar sign ```$```  designates a __context item__ or the whole JSON document, which describes a house with floors ( ```floor[]```), apartments (```apt[]```)  with rooms and room has attribute ```area```.  Expression ```$.floor[*].apt[*]``` in described context means  __any__ room, which filtered by a filter expression (in parentheses).  At sign ```@``` in filter designates the __current item__ in filter.   Path expression could have more filters (applied  left to right) and each of them may be nested. 
```sql
'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90) ? (@.rooms > 2)'
```
It's possible to use the __path variables__ in path expression, whose values are set in __PASSING__ clause of invoked SQL/JSON function. For example (js is a column of type JSON):
```sql
SELECT JSON_QUERY(js, '$.floor[*].apt[*] ? (@.area > $min && @.area < $max)' PASSING 40 AS min, 90 AS max WITH WRAPPER) FROM house;
                               json_query
------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
```
__WITH WRAPPER__ clause  is used to wrap the results into array, since  JSON_QUERY output should be  JSON text.  If minimal and maximal values are stored in table ```area (min integer, max integer)```, then it is possible to pass them to the path expression:
```sql
SELECT JSON_QUERY(house.js, '$.floor[*].apt[*] ? (@.area > $min && @.area < $max)' PASSING area.min AS min, area.max AS max WITH WRAPPER) FROM house, area;
```
Example of using several filters in json path expression, which returns room number (integer) and the room should satisfies the several conditions: the one is checking floor level and another - its area.
```sql
SELECT JSON_VALUE(js, '$.floor[*] ? (@.level > 1).apt[*] ? (@.area > 40 && @.area < 90).no' RETURNING int) FROM house;
 json_value
------------
          5
(1 row)
```

Path expression may  contains several  __item methods__ (out of eight predefined functions), which applies to the result of preceding path expression. For example,  item method ```.double()``` converts ```area``` into a double number.
```sql
'$.floor[0].apt[1].area.double()'
```
More complex example with __keyvalue()__ method, which outputs an array of  room numbers. 
```sql
SELECT JSON_QUERY( js , '$.floor[*].apt[*].keyvalue() ? (@.key == "no").value' WITH WRAPPER) FROM house;
   json_query
-----------------
 [1, 2, 3, 4, 5]
(1 row)
```

## JSONPATH in PostgreSQL

In PostgreSQL the SQL/JSON path language is implemented as  **JSONPATH**  data type - the binary representation of parsed SQL/JSON path expression to effective query JSON data.  Path expression is an optional  path mode (strict | lax), followed by a  path, which is a  sequence of path elements,  started from path  variable, path literal or  expression in parentheses and zero or more operators ( json accessors) .  It  is possible to specify arithmetic or boolean  (*PostgreSQL extension*) expression on the path. *Path can be enclosed in brackets to return an array similar to WITH WRAPPER clause in SQL/JSON query functions. This is a PostgreSQL extension )*.  

Examples of vaild jsonpath:
```sql
'$.floor'
'($+1)'
'$+1'
-- boolean predicate in path, extension
'($.floor[*].apt[*].area > 10)'
'$.floor[*].apt[*] ? (@.area == null).no'
```
Syntactical errors in jsonpath are reported
```sql
SELECT '$a. >1'::jsonpath;
ERROR:  bad jsonpath representation at character 8
DETAIL:  syntax error, unexpected GREATER_P at or near ">"
```
An [Example](#how-path-expression-works) of how path expression works.

### Jsonpath operators

To accelerate JSON path queries using existing indexes for jsonb  (GIN index using built-in  ```jsonb_ops``` or ```jsonb_path_ops```)  PostgreSQL extends the standard with two  boolean operators for json[b] and jsonpath data types.

* ```json[b] @? jsonpath``` -  exists  operator, returns bool.  Check that path expression returns non-empty SQL/JSON sequence.
* ```json[b] @~ jsonpath``` - match operator, returns the result of boolean predicate (*PostgreSQL extension*).

```sql
SELECT js @?  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
 ?column?
----------
 t
(1 row)

SELECT js @~  '$.floor[*].apt[*].area <  20' FROM house;
 ?column?
----------
 f
(1 row)
```
```jsonb @? jsonpath``` and ```jsonb @~ jsonpath``` are as fast as ```jsonb @> jsonb```  (for equality operation),  but  jsonpath supports more complex expressions, for example:
```sql
SELECT count(*) FROM house WHERE   js @~ '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH")  > "1945-03-09".datetime()';
 count
-------
     1
(1 row)
```

Query operators:

* ```json[b] @* jsonpath``` - set-query operator,  returns setof json[b].
* ```json[b] @# jsonpath``` - singleton-query operator,  returns a single json[b].
The results is depends on the size of the resulted SQL/JSON sequence:
--  empty sequence - returns SQL NULL;
-- single item - returns the item;
-- more items - returns array of items.
Notice, that this  behaviour differs from ```WITH CONDITIONAL WRAPPER```, since the latter wraps into array a single scalar value, but not the single object or an array.

```sql
SELECT js @*  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
             ?column?
-----------------------------------
 {"no": 2, "area": 80, "rooms": 3}
 {"no": 5, "area": 60, "rooms": 2}
(2 rows)

SELECT js @#  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
                                ?column?
------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
```
Operator ```@#``` can be used to index jsonb:
```sql
CREATE INDEX bookmarks_oper_path_idx ON bookmarks USING gin((js @# '$.tags.term') jsonb_path_ops);
EXPLAIN ( ANALYZE, COSTS OFF) SELECT COUNT(*) FROM bookmarks WHERE js @# '$.tags.term' @> '"NYC"';
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Aggregate (actual time=1.136..1.136 rows=1 loops=1)
   ->  Bitmap Heap Scan on bookmarks (actual time=0.213..1.098 rows=285 loops=1)
         Recheck Cond: ((js @# '$."tags"."term"'::jsonpath) @> '"NYC"'::jsonb)
         Heap Blocks: exact=285
         ->  Bitmap Index Scan on bookmarks_oper_path_idx (actual time=0.148..0.148 rows=285 loops=1)
               Index Cond: ((js @# '$."tags"."term"'::jsonpath) @> '"NYC"'::jsonb)
 Planning time: 0.228 ms
 Execution time: 1.309 ms
(8 rows)
```

### Path modes

The path engine has two modes, strict and lax, the latter is   default, that is,  the standard tries to facilitate matching of the  (sloppy) document structure and path expression.

In __strict__ mode any structural errors (  an attempt to access a non-existent member of an object or element of an array)  raises an error (it is up to JSON_XXX function to actually report it, see ```ON ERROR``` clause).
For example: 

```sql
SELECT JSON_VALUE(jsonb '1', 'strict $.a' ERROR ON ERROR); -- returns ERROR:  SQL/JSON member not found
```
Notice,  JSON_VALUE function needs ```ERROR ON ERROR```  to report the error  , since default behaviour  ```NULL ON ERROR``` suppresses error reporting and returns ```null```. 

In __strict__ mode  using an array accessor on a scalar value  or  object triggers error handling.
```sql
SELECT JSON_VALUE(jsonb '1', 'strict $[0]' ERROR ON ERROR);
ERROR:  SQL/JSON array not found
SELECT JSON_VALUE(jsonb '1', 'strict $.a' ERROR ON EMPTY ERROR ON ERROR);
ERROR:  SQL/JSON member not found
```

 In __lax__ mode the path engine suppresses the structural errors and  converts them to the empty SQL/JSON sequences.  Depending on ```ON EMPTY``` clause  the empty SQL/JSON sequences  will be  interpreted as ```null``` by default ( ```NULL ON EMPTY```) or raise an ERROR ( ```ERROR ON EMPTY```).

```sql
SELECT JSON_VALUE(jsonb '1', 'lax $.a' ERROR ON ERROR);
 json_value
------------
 (null)
(1 row)
SELECT JSON_VALUE(jsonb '1', 'lax $.a' ERROR ON EMPTY ERROR ON ERROR);
ERROR:  no SQL/JSON item
```

Also,  in __lax__ mode arrays of size 1 is interchangeable with the singleton. 

Example of automatic array wrapping in lax mode:
```sql
SELECT JSON_VALUE(jsonb '1', 'lax $[0]' ERROR ON ERROR);
 json_value
------------
 1
(1 row)
```

### Path Elements

__path literal__
~ JSON primitive types : unicode text, numeric, true, false, null

__path variable__
~ ```$``` -- context item
~ ```$var``` -- named variable, value is set in PASSING clause (*may be of   datetime type*)
~ ```@``` -- value of the current item in a filter
~ ```last``` - JSON last subscript of an array

__expression in parentheses__
~  ```($a + 2)```

__Path elements__ 
 ~ __member accessor__  -- ```.name``` or ```."$name"```.  It is used to access a member of an object by key name.  If the key name does not begin with a dollar sign and meets the JavaScript rules of an Identifier, then the member name can be written in clear text, else it can be written as a character string literal. For example: ```$.color, "$."color of a pen"```.  In __strict__ mode, every SQL/JSON item in the SQL/JSON sequence must be an object with a member having the specified key name. If this condition is not met, the result is an error.  In __lax__ mode, any SQL/JSON array in the SQL/JSON sequence is unwrapped. Unwrapping only goes **one deep**; that is, if there is an array of arrays, the outermost array is unwrapped, leaving the inner arrays alone.

 
 ~ __wildcard member accessor__  --  ```.*```, the values of all attributes of the current  object.  In __strict__ mode, every SQL/JSON item in the SQL/JSON sequence must be an object. If this condition is not met, the result is an error. In __lax__ mode, any SQL/JSON array in the SQL/JSON sequence is unwrapped.
 
 ~ __array element accessor__  -- ```[1,5 to LAST]```, the second and the six-th to the last array elements of the array . Since jsonpath follows javascript conventions rather than SQL, ```[0]``` accesses is the first element in the array. ```LAST``` is the special variable to handle arrays of unknown length, its value is the size of the array minus 1.   In the __strict__ mode, subscripts must be singleton numeric values between 0 and last; in __lax__ mode, any subscripts that are out of bound are simply ignored. In both __strict__ and __lax__ mode, non-numeric subscripts are an error.
 
 ~ __wildcard array element accessor__ -- ```[*]```, all array elements. In __strict__ mode, the operand must be an array, in __lax__ mode, if the operand is not an array, then one is provided by wrapping it in an array before unwrapping, ```$[*]``` is the same as ```$[0 to last]```.  The latter is not valid in __strict__ mode, since ```$[0 to last]``` requires at least one array element and raise an error if ```$``` is the empty array , while ```$[*]```  returns ```null``` in that case.
 ~ __filter expression__ --  ? (expression) , the result of filter expression may be ```unknown```, ```true```,  ```false```. 
 ~ __item method__ -- is the function, that operate on an SQL/JSON item and return an SQL/JSON item. 
It denotes as  ```.``` and could be one of the 8 methods:

  ~ __type()__ - returns a character string that names the type of the SQL/JSON item ( ```"null"```, ```"boolean"```, ```"number"```, ```"string"```, ```"array"```, ```"object"```, ```"date"```, ```"time without time zone"```, ```"time with time zone"```, ```"timestamp without time zone"```, ```"timestamp with time zone"```).
   
  ~ __size()__ -  returns the size of an SQL/JSON item, which is the number of elements in the array or 1 for SQL/JSON object or scalar in __lax__ mode  and error ```ERROR:  SQL/JSON array not found``` in __strict__ mode.  
```sql
SELECT JSON_VALUE('[1,2,3]', '$.size()' RETURNING int);
 json_value
------------
          3
(1 row)
```
In more complex case,  we can wrap SQL/JSON sequence into an array and apply ```.size()``` to the result:
```
 SELECT JSON_VALUE(JSON_QUERY('[1,2,3]', '$[*] ? (@ > 1)' WITH WRAPPER), '$.size()' RETURNING int);
 json_value
------------
          2
(1 row)
 ```
 Or use our automatic wrapping extension to the path expression
 ```sql
SELECT JSON_VALUE('[1,2,3]', '[$[*] ? (@ > 1)].size()' RETURNING int);
 json_value
------------
          2
(1 row)
 ```
 
   ~  __ceiling()__ - the same as ```CEILING``` in SQL
   ~ __double()__ - converts a string or numeric to an approximate numeric value.
   ~ __floor()__ - the same as ```FLOOR``` in SQL
   ~ __abs()__  - the same as ```ABS``` in SQL
   ~ __datetime()__ - converts a character string to an SQL datetime type, optionally using a conversion template ( [templates examples](https://www.postgresql.org/docs/current/static/functions-formatting.html)).
```sql
SELECT JSON_VALUE('"10-03-2017"','$.datetime("dd-mm-yyyy")');
 json_value
------------
 2017-03-10
(1 row)
SELECT JSON_VALUE('"12:34:56"','$.datetime().type()');
       json_value
------------------------
 time without time zone
(1 row)
-- all datetime examples are obtained with timezone 'W-SU'
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ < "2000-01-01".datetime())' FROM house;
          ?column?           
-----------------------------
 "1957-10-04T19:28:34+00:00"
 "1961-04-12T06:07:00+00:00"
(2 rows)
-- datetime cannot compared to string
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ < "2000-01-01")' FROM house;
 ?column? 
----------
(0 rows)
-- rejected items in previous query
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? ((@ < "2000-01-01") is unknown)' FROM house;
          ?column?           
-----------------------------
 "2015-02-01T00:00:00+00:00"
 "1957-10-04T19:28:34+00:00"
 "1961-04-12T06:07:00+00:00"
(3 rows)
-- 
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ > "1961-04-12".datetime())' FROM house;
          ?column?
-----------------------------
 "2015-02-01T00:00:00+03:00"
 "1961-04-12T09:07:00+03:00"
(2 rows)
-- set timezone = 'UTC';
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ > "1961-04-12".datetime())' FROM house;
          ?column?
-----------------------------
 "2015-02-01T00:00:00+00:00"
 "1961-04-12T06:07:00+00:00"
(2 rows)
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy") ? (@ > "1961-04-12".datetime())' FROM house;
   ?column?
--------------
 "2015-02-01"
(1 row)
```
   ~ __keyvalue()__ - transforms json to an SQL/JSON sequence of objects with a known schema. Example:
   ```sql
   SELECT JSON_QUERY( '{"a": 123, "b": 456, "c": 789}', '$.keyvalue()' WITH WRAPPER);
                                       ?column?
--------------------------------------------------------------------------------------
 [{"key": "a", "value": 123}, {"key": "b", "value": 456}, {"key": "c", "value": 789}]
(1 row)
  ```
   
 **PostgreSQL extensions**:
   1.  __recursive wildcard member accessor__ -- ```.**```,  recursively applies wildcard member accessor ```.*``` to all levels of hierarchy and returns   the values of **all** attributes of the current object regardless of the level of the hierarchy.
Examples:
Wildcard member accessor returns the values of all elements without looking deep.
```sql
SELECT jsonb '{"a":{"b":[1,2]}, "c":1}' @* '$.*';
   ?column?
---------------
 {"b": [1, 2]}
 1
(2 rows)
```
Recursive wildcard member accessor "unwraps"  all objects and arrays
```sql
SELECT jsonb '{"a":{"b":[1,2]}, "c":1}' @* '$.**';
           ?column?
------------------------------
 {"a": {"b": [1, 2]}, "c": 1}
 {"b": [1, 2]}
 [1, 2]
 1
 2
 1
(6 rows)
 ```
 
  2.   __automatic wrapping__  - ```[path]``` is equivalent to ```WITH WRAPPER``` in SQL_XXX function.
   ```sql
   SELECT JSON_QUERY('[1,2,3]', '[$[*]]');
 json_query
------------
 [1, 2, 3]
(1 row)
SELECT JSON_QUERY('[1,2,3]', '$[*]' WITH WRAPPER);
 json_query
------------
 [1, 2, 3]
(1 row)
  ```

   
 ### Filter expression
A filter expression is similar to a WHERE clause in SQL, it is used to remove SQL/JSON items from an SQL/JSON sequence if they do not satisfy a predicate. The syntax uses a question mark ```?``` followed by a parenthesized predicate. In __lax__ mode, any SQL/JSON arrays in the operand are automatically unwrapped. The predicate is evaluated for each SQL/JSON item in the SQL/JSON sequence.  Predicate returns ```Unknown``` (SQL NULL) if any error occured during evaluation of its operands and execution. The result is those SQL/JSON items for which the predicate resulted in ```True```, ```False``` and ```Unknown``` are rejected. 

Within a filter, the special variable @ is used to reference the current SQL/JSON item in the SQL/JSON sequence.

The SQL/JSON path language has the following predicates:

 - ```exists``` predicate, to test if a path expression has a non-empty result.
- Comparison predicates ==, !=, <>, <, <=, >, and >=.
- ```like_regex``` for string pattern matching.  Optional parameter```flag``` can be combination of ```i,s,m,x```, default value is ```s```.
- ```starts with``` to test for an initial substring (prefix).
- ```is unknown``` to test for ```Unknown``` results. Its operand should be in parentheses.

JSON literals ```true, false``` are parsed into the SQL/JSON model as the SQL boolean values ```True``` and ```False```. 
```sql
SELECT JSON_VALUE(jsonb 'true','$ ? (@ == true)') FROM house;
 json_value
------------
 true
(1 row)
SELECT jsonb '{"g": {"x": 2}}' @* '$.g ? (exists (@.x))';
 ?column?
----------
 {"x": 2}
(1 row)
```
JSON literal ```null``` is parsed into the special SQL/JSON value ```null```, which differs from SQL NULL, for example, SQL JSON ```null``` is equal to itself, so the result of ```null == null``` is ```True```.
```sql
SELECT json_query('1', '$ ? (null == null)');
 json_query
------------
 1
(1 row)
SELECT json_query('1', '$ ? (null != null)');
 json_query
------------
 (null)
(1 row)
``` 
Prefix search with ```starts with``` example:
```sql
SELECT js @* '$.** ? (@ starts with "11")' FROM house;
            ?column?
---------------------------------
 "117036, Dmitriya Ulyanova, 7A"
(1 row)
```
Regular expression search:
```sql
-- case insensitive
SELECT js @* '$.** ? (@ like_regex "O(w|v)" flag "i")' FROM house;
            ?column?
---------------------------------
 "Moscow"
 "117036, Dmitriya Ulyanova, 7A"
(2 rows)
-- ignore spaces in query, flag "x"
SELECT js @* '$.** ? (@ like_regex "O w|o V" flag "ix")' FROM house;
            ?column?
---------------------------------
 "Moscow"
 "117036, Dmitriya Ulyanova, 7A"
(2 rows)
-- single-line mode, flag "s".
SELECT js @* '$.** ? (@ like_regex "^info@" flag "is")' FROM house;
 ?column?
----------
(0 rows)
-- multi-line mode, flag "m"
SELECT js @* '$.** ? (@ like_regex "^info@" flag "im")' FROM house;
                             ?column?
------------------------------------------------------------------
 "Postgres Professional\n+7 (495) 150-06-91\ninfo@postgrespro.ru"
(1 row)
```


Predicate ```is unknown``` can be used to filter "problematic" SQL/JSON items. Notice, that filter expression automatically unwraps array ```apt```, since default mode is ```lax```.
```sql
SELECT js @* '$.floor.apt ? ((@.area / @.rooms > 0))' FROM house;
              ?column?
------------------------------------
 {"no": 1, "area": 40, "rooms": 1}
 {"no": 2, "area": 80, "rooms": 3}
 {"no": 4, "area": 100, "rooms": 3}
 {"no": 5, "area": 60, "rooms": 2}
(4 rows)
-- what item was rejected by a filter ?
SELECT js @* '$.floor.apt ? ((@.area / @.rooms > 0) is unknown)' FROM house;
              ?column?
-------------------------------------
 {"no": 3, "area": null, "rooms": 2}
(1 row)
```

### How path expression works

Path expression is intended to produce an SQL/JSON sequence ( an ordered list of zero or more SQL/JSON items) and completion code for  SQL/JSON query functions or operator, whose job is to process that result using the particular SQL/JSON query operator. The path engine process path expression step by step, each of which produces SQL/JSON sequence for following step. For example,  path expression  
```sql
'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)'
```
at first step produces SQL/JSON sequence of length 1, which is simply json itself - the context item, denoted as ```$```.  Next step is member accessor ```.floor``` - the result is SQL/JSON sequence of length 1 - an array ```floor```, which then unwrapped by  wildcard  array element accessor  ```[*]``` to SQL/JSON sequence of length 2, containing an array of two objects . Next, ```.apt```  produces two arrays of objects and ```[*]``` extracts  SQL/JSON sequence of length 5 ( five objects - appartments), each of which filtered by a filter expression ```(@.area > 40 && @.area < 90)```, so a result of the whole path expression  is a sequence of two SQL/JSON items.

```
SELECT JSON_QUERY(js,'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' WITH WRAPPER) FROM house;
                               json_query
------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
```
The result by jsonpath set-query operator:
```sql
SELECT js @*  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
             ?column?
-----------------------------------
 {"no": 2, "area": 80, "rooms": 3}
 {"no": 5, "area": 60, "rooms": 2}
(2 rows)
```

### SQL/JSON conformance

- ```like_regex``` supports posix regular expressions,  while standard requires xquery regexps.
-  Not supported (due to unresolved conflicts in SQL grammar):
   - expr FORMAT JSON IS [NOT] JSON
   - JSON_OBJECT(KEY key VALUE value, ...)
   - JSON_ARRAY(SELECT ... FORMAT JSON ...)
   - JSON_ARRAY(SELECT ... (ABSENT|NULL) ON NULL ...)
  - Only error codes are returned for the failed arithmetic operations inside jsonpath, error messages are lost
-  Use boolean  expression on the path, PostgreSQL extension
- ```.**```  - recursive wildcard member accessor, PostgreSQL extension
- json[b] op jsonpath - PostgreSQL extension
- [path] - wrap SQL/JSON sequences into an array - PostgreSQL extension


 ### Links
* Github Postgres Professional repository
https://github.com/postgrespro/sqljson
*  WEB-interface to play with SQL/JSON
http://sqlfiddle.postgrespro.ru/#!21/0/2580
* Technical Report (SQL/JSON) - available for free
http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip
* Jsonb roadmap - talk at PGConf.eu, 2018
http://www.sai.msu.su/~megera/postgres/talks/sqljson-pgconf.eu-2017.pdf


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19
-->
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTgwMjQzOTRdfQ==
-->