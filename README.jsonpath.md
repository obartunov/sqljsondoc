---


---

<h2 id="jsonpath-introduction">Jsonpath introduction</h2>
<p>SQL-2016 standard introduced SQL/JSON data model and path language used by certain SQL/JSON functions to query JSON.  The main task of the path language is to specify  the parts (the projection)  of JSON data to be retrieved by path engine for that functions.  The language is designed to be flexible enough to meet the current needs and to be adaptable to the future use cases. Also, it is integratable into SQL engine, i.e., the semantics of predicates and operators generally follow SQL.  To be friendly to JSON users, the language resembles  JavaScript - dot(.)  used for member access and [] for array access, arrays starts from zero (SQL arrays starts from 1).</p>
<p>Example of two-floors house:</p>
<pre><code>CREATE TABLE house AS
SELECT jsonb '{
  "address": {
    "city": "Moscow",
    "street": "Ulyanova, 7A"
  },
  "lift": false,
  "floor": [
    {
      "level": 1,
      "apt": [
        {"no": 1, "area": 40, "rooms": 1},
        {"no": 2, "area": 80, "rooms": 3},
        {"no": 3, "area": 50, "rooms": 2}
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
}' js;
</code></pre>
<p>For example,  the result of this path expression will be information about apartments with rooms, which area is  in specified range.</p>
<pre><code>'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'
</code></pre>
<p>Dollar sign <code>$</code>  designates a <strong>context item</strong> or the whole JSON document, which describes a house with floors ( <code>floor[]</code>), apartments (<code>apt[]</code>)  with rooms and room has attribute <code>area</code>.  Expression <code>$.floor[*].apt[*]</code> in described context means  <strong>any</strong> room, which filtered by a filter expression (in parentheses).  At sign <code>@</code> in filter designates the <strong>current item</strong> in filter.   Path expression could have more filters (applied  left to right) and each of them may be nested.</p>
<pre><code>'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90) ? (@.rooms &gt; 2)'
</code></pre>
<p>It’s possible to use the <strong>path variables</strong> in path expression, whose values are set in <strong>PASSING</strong> clause of invoked SQL/JSON function. For example (js is a column of type JSON):</p>
<pre><code>SELECT JSON_QUERY(js, '$.floor[*].apt[*] ? (@.area &gt; $min &amp;&amp; @.area &lt; $max)' PASSING 40 AS min, 90 AS max WITH WRAPPER) FROM house;
                                                 ?column?
-----------------------------------------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 3, "area": 50, "rooms": 2}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
</code></pre>
<p><strong>WITH WRAPPER</strong>  is used to wrap the results into array, since  JSON_QUERY output should be   JSON text.  If minimal and maximal values are stored in table <code>area (min integer, max integer)</code>, then it is possible to pass them to path expression:</p>
<pre><code>SELECT JSON_QUERY(house.js, '$.floor[*].apt[*] ? (@.area &gt; $min &amp;&amp; @.area &lt; $max)' PASSING area.min AS min, area.max AS max WITH WRAPPER) FROM house, area;
</code></pre>
<p>Example of using several filters in json path expression, which returns room number (integer) and the room should satisfies the several conditions: the one is checking floor level and another - its area.</p>
<pre><code>SELECT JSON_VALUE(js, '$.floor[*] ? (@.level &gt; 1).apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90).no' RETURNING int) FROM house;
 ?column?
----------
        5
(1 row)
</code></pre>
<p>Path expression may  contains several  <strong>item methods</strong> (out of eight predefined functions), which applies to the result of preceding path expression. For example,  item method <code>.double()</code> converts <code>area</code> into a double number.</p>
<pre><code>'$.floor[0].apt[1].area.double()'
</code></pre>
<p>More complex example with <strong>keyvalue()</strong> method, which outputs an array of values of keys <code>"a","b"</code>.</p>
<pre><code>SELECT JSON_QUERY( '{"a": 123, "b": 456, "c": 789}', '$.keyvalue() ? (@.key == "a" || @.key == "c").value' WITH WRAPPER);
  ?column?
------------
 [123, 789]
(1 row)
</code></pre>
<h2 id="jsonpath-in-postgresql">JSONPATH in PostgreSQL</h2>
<p>In PostgreSQL the SQL/JSON path language is implemented as  <strong>JSONPATH</strong>  data type - the binary representation of parsed SQL/JSON path expression to effective query JSON data.  Path expression is a path mode (strict | lax), followed by a  path, which is a  sequence of path elements,  started from path  variable, path literal or  expression in parentheses.</p>
<dl>
<dt><strong>path literal</strong></dt>
<dd>JSON primitive types : unicode text, numeric, true, false, null</dd>
<dt><strong>path variable</strong></dt>
<dd>$ – context item</dd>
<dd>$var – named variable, value is set in PASSING clause</dd>
<dd>@ – value of the current item in a filter</dd>
<dd>last - JSON last subscript of an array</dd>
<dt><strong>expression in parentheses</strong></dt>
<dd>‘($a + 2)’</dd>
<dt><strong>Path elements</strong></dt>
<dd><strong>member accessor</strong>  – <code>.color</code>, the value of the <code>color</code> attribute</dd>
<dd><strong>wildcard member accessor</strong>  –  <code>.*</code>, the values of all attributes</dd>
<dd><strong>array accessor</strong>  – <code>[1,5 to LAST]</code>, the second and the six-th to the last array elements of the array</dd>
<dd><strong>wildcard array accessor</strong> – <code>[*]</code>, all array elements. In <strong>strict</strong> mode, the operand must be an array, in <strong>lax</strong> mode, if the operand is not an array, then one is provided by wrapping it in an array before unwrapping, <code>$[*]</code> is the same as <code>$[0 to last]</code>.  The latter is not valid in <strong>strict</strong> mode, since <code>$[0 to last]</code> requires at least one array element and raise an error if <code>$</code> is the empty array , while <code>$[*]</code>  returns <code>null</code> in that case.</dd>
<dd><strong>filter expression</strong> –  ? (expression) , the result of filter expression may be <code>unknown</code>, <code>true</code>,  <code>false</code>.</dd>
<dd>
<dl>
<dt><strong>item method</strong> –  <code>.</code> and one of the 8 methods:</dt>
<dd>type()</dd>
<dd>size()</dd>
<dd>ceiling()</dd>
<dd>double()</dd>
<dd>floor()</dd>
<dd>abs()</dd>
<dd>datetime()</dd>
<dd>keyvalue()</dd>
</dl>
</dd>
</dl>
<h3 id="path-modes">Path modes</h3>
<p>The path engine has two modes, strict and lax, the latter is   default, that is,  the standard tries to facilitate matching of the [sloppy] document structure and path expression.</p>
<p>In <strong>strict</strong> mode any structural errors (  an attempt to access a non-existent member of an object or element of an array)  raises an error (it is up to JSON_XXX function to actually report it, see <code>ON ERROR</code> clause).<br>
For example:</p>
<pre><code>SELECT JSON_VALUE(jsonb '1', 'strict $.a' ERROR ON ERROR); -- returns ERROR:  SQL/JSON member not found
</code></pre>
<p>Notice,  JSON_VALUE function needs <code>ERROR ON ERROR</code>  to report the error  , since default behaviour  <code>NULL ON ERROR</code> suppresses error reporting and returns <code>null</code>.</p>
<p>In <strong>strict</strong> mode  using an array accessor on a scalar value  or  object triggers error handling.</p>
<pre><code>SELECT JSON_VALUE(jsonb '1', 'strict $[0]' ERROR ON ERROR); -- returns ERROR:  SQL/JSON array not found
</code></pre>
<p>In <strong>lax</strong> mode the path engine supresses the structural errors and  converts them to the empty SQL/JSON sequences.  Depending on <code>ON EMPTY</code> clause  the empty SQL/JSON sequences  will be  interpreted as <code>null</code> by default ( <code>NULL ON EMPTY</code>) or raise an ERROR ( <code>ERROR ON EMPTY</code>).</p>
<pre><code>SELECT JSON_VALUE(jsonb '1', 'lax $.a' ERROR ON ERROR); -- returns null
SELECT JSON_VALUE(jsonb '1', 'lax $.a' ERROR ON EMPTY ERROR ON ERROR); -- returns ERROR:  SQL/JSON member not found
</code></pre>
<p>Also,  in <strong>lax</strong> mode arrays of size 1 is interchangeable with the singleton.</p>
<p>Example of automatic array wrapping in lax mode:</p>
<pre><code>SELECT JSON_VALUE(jsonb '1', 'lax $[0]' ERROR ON ERROR); -- returns 1
</code></pre>
<h3 id="member-accessors">Member accessors</h3>
<h3 id="filter-expession">Filter expession</h3>
<h3 id="links">Links</h3>
<ul>
<li>Github Postgres Professional repository<br>
<a href="https://github.com/postgrespro/sqljson">https://github.com/postgrespro/sqljson</a></li>
<li>WEB-interface to play with SQL/JSON<br>
<a href="http://sqlfddle.postgrespro.ru/#!21/0/1819">http://sqlfddle.postgrespro.ru/#!21/0/1819</a></li>
<li>Technical Report (SQL/JSON) - available for free<br>
<a href="http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip">http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip</a></li>
</ul>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

