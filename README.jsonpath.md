---


---

<h2 id="jsonpath-introduction">Jsonpath introduction</h2>
<p>SQL-2016 standard introduced SQL/JSON data model and path language used by certain SQL/JSON functions to query JSON.  The main task of the path language is to specify  the parts (the projection)  of JSON data to be retrieved by path engine for that functions.  The language is designed to be flexible enough to meet the current needs and to be adaptable to the future use cases. Also, it is integratable into and SQL engine, i.e., the semantics of predicates and operators generally follow SQL.  To be friendly to JSON users, the language resembles  JavaScript - dot(.)  used for member access and [] for array access, arrays starts from zero (SQL arrays starts from 1).</p>
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
<p>Dollar sign <code>$</code>  designates a <strong>context item</strong> or the whole JSON document, which describes a house with floors (array <code>floor</code>), apartments (array <code>apt</code>)  with rooms and room has attribute <code>area</code>.  Expression <code>$.floor[*].apt[*]</code> in described context means  <strong>any</strong> room, which filtered by a filter expression (in parentheses).  At sign <code>@</code> in filter designates the <strong>current item</strong> in filter.   Path expression could have more filters (applied  left to right) and each of them may be nested.</p>
<pre><code>'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90) ? (@.rooms &gt; 2)'
</code></pre>
<p>It’s possible to use the <strong>path variables</strong> in path expression, whose values are set in <strong>PASSING</strong> clause of invoked SQL/JSON function. For example (js is a column of type JSON):</p>
<pre><code>SELECT JSON_QUERY(js, '$.floor[*].apt[*] ? (@.area &gt; $min &amp;&amp; @.area &lt; $max)' PASSING 40 AS min, 90 AS max WITH WRAPPER) FROM house;
</code></pre>
<p><strong>WITH WRAPPER</strong>  is used to wrap the results into array to satisfy the JSON_QUERY  requirements, which expects JSON text.  If minimal and maximal values are stored in table <code>area (min integer, max integer)</code>, then it is possible to pass them to path expression:</p>
<pre><code>SELECT JSON_QUERY(house.js, '$.floor[*].apt[*] ? (@.area &gt; $min &amp;&amp; @.area &lt; $max)' PASSING area.min AS min, area.max AS max WITH WRAPPER) FROM house, area;
</code></pre>
<p>Path expression may  contains  <strong>item methods</strong> (out of eight predefined functions), for example, tem method <code>.double()`` converts</code>area``` into a double number.</p>
<pre><code>'$.floor[0].apt[1].area.double()'
</code></pre>
<p>The path engine has two modes, strict and lax (default mode).  They used to facilitate user working with JSON data, which often has a sloppy schema. In <strong>strict</strong> mode any structural errors ( data doesn’t strictly match a path expression)  raises an error, while in <strong>lax</strong> mode errors will be converted to empty SQL/JSON sequences.  For example:</p>
<pre><code>SELECT JSON_VALUE(jsonb '1', 'strict $.a' ERROR ON ERROR); -- returns ERROR:  SQL/JSON member not found
SELECT JSON_VALUE(jsonb '1', 'lax $.a' ERROR ON ERROR); -- returns null
</code></pre>
<p>Also,  in lax mode arrays of size 1 is interchangeable with the singleton.  Example of automatic array wrapping in lax mode:</p>
<pre><code>SELECT JSON_VALUE(jsonb '1', 'strict $[0]' ERROR ON ERROR); -- returns ERROR:  SQL/JSON array not found
SELECT JSON_VALUE(jsonb '1', 'lax $[0]' ERROR ON ERROR); -- returns 1
</code></pre>
<h2 id="jsonpath-in-postgresql">JSONPATH in PostgreSQL</h2>
<p>In PostgreSQL the SQL/JSON path language is implemented as  <strong>JSONPATH</strong>  data type - the binary representation of parsed SQL/JSON path expression to effective query JSON data.  Path expression is a path mode (strict | lax), followed by a   sequence of path elements,  started from path  variable, path literal or  expression in parentheses.</p>
<dl>
<dt><strong>path literal</strong></dt>
<dd>JSON primitive types : unicode text, numeric, true, false, null</dd>
<dt><strong>path variable</strong></dt>
<dd>$ – context item</dd>
<dd>$var – named variable, value is set in PASSING clause</dd>
<dd>@ – value of the current item in a filter</dd>
<dd>last - JSON last subscript of an array</dd>
<dt><strong>expression in parentheses</strong></dt>
<dd>‘($.a.b &gt; 2)’</dd>
<dt><strong>Path elements</strong></dt>
<dd>member accessor  – .color</dd>
<dd>wildcard member accessor  –  .*</dd>
<dd>array accessor  – [1,2,3]</dd>
<dd>wildcard array accessor – [*]</dd>
<dd>filter expression –  ? (…)</dd>
<dd>
<dl>
<dt>item method –  <code>.</code> and one of the 8 methods:</dt>
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
<h3 id="member-accessors">Member accessors</h3>
<h3 id="filter-expession">Filter expession</h3>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

