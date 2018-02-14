---


---

<h2 id="sqljson-data-model">SQL/JSON Data model</h2>
<p>tbw</p>
<h2 id="jsonpath-introduction">Jsonpath introduction</h2>
<p>SQL-2016 standard introduced SQL/JSON data model and path language used by certain SQL/JSON functions to query JSON.  The main task of the path language is to specify  the parts (the projection)  of JSON data to be retrieved by path engine for that functions.  The language is designed to be flexible enough to meet the current needs and to be adaptable to the future use cases. Also, it is integratable into SQL engine, i.e., the semantics of predicates and operators generally follow SQL.  To be friendly to JSON users, the language resembles  JavaScript - dot(<code>.</code>)  used for member access and [] for array access, arrays starts from zero (SQL arrays starts from 1).</p>
<p>Example of two-floors house:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> house <span class="token keyword">AS</span>
<span class="token keyword">SELECT</span> jsonb <span class="token string">'{
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
}'</span> js<span class="token punctuation">;</span>
</code></pre>
<p>For example,  the result of this path expression will be information about apartments with rooms, which area is  in specified range.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span>
</code></pre>
<p>Dollar sign <code>$</code>  designates a <strong>context item</strong> or the whole JSON document, which describes a house with floors ( <code>floor[]</code>), apartments (<code>apt[]</code>)  with rooms and room has attribute <code>area</code>.  Expression <code>$.floor[*].apt[*]</code> in described context means  <strong>any</strong> room, which filtered by a filter expression (in parentheses).  At sign <code>@</code> in filter designates the <strong>current item</strong> in filter.   Path expression could have more filters (applied  left to right) and each of them may be nested.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90) ? (@.rooms &gt; 2)'</span>
</code></pre>
<p>It’s possible to use the <strong>path variables</strong> in path expression, whose values are set in <strong>PASSING</strong> clause of invoked SQL/JSON function. For example (js is a column of type JSON):</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; $min &amp;&amp; @.area &lt; $max)'</span> PASSING <span class="token number">40</span> <span class="token keyword">AS</span> min<span class="token punctuation">,</span> <span class="token number">90</span> <span class="token keyword">AS</span> max <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
                               json_query
<span class="token comment">------------------------------------------------------------------------</span>
 <span class="token punctuation">[</span>{<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}<span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p><strong>WITH WRAPPER</strong>  is used to wrap the results into array, since  JSON_QUERY output should be   JSON text.  If minimal and maximal values are stored in table <code>area (min integer, max integer)</code>, then it is possible to pass them to the path expression:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span>house<span class="token punctuation">.</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; $min &amp;&amp; @.area &lt; $max)'</span> PASSING area<span class="token punctuation">.</span>min <span class="token keyword">AS</span> min<span class="token punctuation">,</span> area<span class="token punctuation">.</span>max <span class="token keyword">AS</span> max <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">,</span> area<span class="token punctuation">;</span>
</code></pre>
<p>Example of using several filters in json path expression, which returns room number (integer) and the room should satisfies the several conditions: the one is checking floor level and another - its area.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*] ? (@.level &gt; 1).apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90).no'</span> RETURNING <span class="token keyword">int</span><span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
 json_value
<span class="token comment">------------</span>
          <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Path expression may  contains several  <strong>item methods</strong> (out of eight predefined functions), which applies to the result of preceding path expression. For example,  item method <code>.double()</code> converts <code>area</code> into a double number.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor[0].apt[1].area.double()'</span>
</code></pre>
<p>More complex example with <strong>keyvalue()</strong> method, which outputs an array of values of keys <code>"a","b"</code>.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span> <span class="token string">'{"a": 123, "b": 456, "c": 789}'</span><span class="token punctuation">,</span> <span class="token string">'$.keyvalue() ? (@.key == "a" || @.key == "c").value'</span> <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_query
<span class="token comment">------------</span>
 <span class="token punctuation">[</span><span class="token number">123</span><span class="token punctuation">,</span> <span class="token number">789</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h2 id="jsonpath-in-postgresql">JSONPATH in PostgreSQL</h2>
<p>In PostgreSQL the SQL/JSON path language is implemented as  <strong>JSONPATH</strong>  data type - the binary representation of parsed SQL/JSON path expression to effective query JSON data.  Path expression is an optional  path mode (strict | lax), followed by a  path, which is a  sequence of path elements,  started from path  variable, path literal or  expression in parentheses and zero or more operators ( json accessors) .  It  is possible to specify arithmetic or boolean  (<em>PostgreSQL extension</em>) expression on the path.</p>
<p>Examples of vaild jsonpath:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor'</span>::jsonpath
<span class="token string">'($+1)'</span>::jsonpath<span class="token punctuation">;</span>
<span class="token string">'($.floor[*].apt[*].area &gt; 10)'</span>
</code></pre>
<p><em>Path can be enclosed in brackets to return an array similar to WITH WRAPPER clause in SQL/JSON query functions. This is a PostgreSQL extension )</em>.<br>
An <a href="#how-path-expression-works">Example</a> of how path expression works.</p>
<h3 id="jsonpath-operators">Jsonpath operators</h3>
<p>To accelerate JSON path queries using existing indexes for jsonb  PostgreSQL introduced several boolean operators for json[b] and jsonpath data types.</p>
<ul>
<li><code>json[b] @? jsonpath</code> -  exists  operator, returns bool</li>
<li><code>json[b] @~ jsonpath</code> - match operator, returns bool</li>
<li><code>json[b] @* jsonpath</code> - query operator,  returns setof json[b]</li>
</ul>
<p>Examples:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> js @?  <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span> <span class="token keyword">from</span> house<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?
<span class="token comment">----------</span>
 t
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> js @?  <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 100 &amp;&amp; @.area &lt; 90)'</span> <span class="token keyword">from</span> house<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?
<span class="token comment">----------</span>
 <span class="token number">f</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="path-modes">Path modes</h3>
<p>The path engine has two modes, strict and lax, the latter is   default, that is,  the standard tries to facilitate matching of the  (sloppy) document structure and path expression.</p>
<p>In <strong>strict</strong> mode any structural errors (  an attempt to access a non-existent member of an object or element of an array)  raises an error (it is up to JSON_XXX function to actually report it, see <code>ON ERROR</code> clause).<br>
For example:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'strict $.a'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">-- returns ERROR:  SQL/JSON member not found</span>
</code></pre>
<p>Notice,  JSON_VALUE function needs <code>ERROR ON ERROR</code>  to report the error  , since default behaviour  <code>NULL ON ERROR</code> suppresses error reporting and returns <code>null</code>.</p>
<p>In <strong>strict</strong> mode  using an array accessor on a scalar value  or  object triggers error handling.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'strict $[0]'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">-- returns ERROR:  SQL/JSON array not found</span>
<span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'strict $.a'</span> ERROR <span class="token keyword">ON</span> EMPTY ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">-- returns ERROR:  SQL/JSON member not found</span>
</code></pre>
<p>In <strong>lax</strong> mode the path engine suppresses the structural errors and  converts them to the empty SQL/JSON sequences.  Depending on <code>ON EMPTY</code> clause  the empty SQL/JSON sequences  will be  interpreted as <code>null</code> by default ( <code>NULL ON EMPTY</code>) or raise an ERROR ( <code>ERROR ON EMPTY</code>).</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'lax $.a'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">-- returns null</span>
<span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'lax $.a'</span> ERROR <span class="token keyword">ON</span> EMPTY ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">-- returns ERROR:  no SQL/JSON item</span>
</code></pre>
<p>Also,  in <strong>lax</strong> mode arrays of size 1 is interchangeable with the singleton.</p>
<p>Example of automatic array wrapping in lax mode:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'lax $[0]'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">-- returns 1</span>
</code></pre>
<h3 id="path-elements">Path Elements</h3>
<dl>
<dt><strong>path literal</strong></dt>
<dd>
<p>JSON primitive types : unicode text, numeric, true, false, null</p>
</dd>
<dt><strong>path variable</strong></dt>
<dd>
<p><code>$</code> – context item</p>
</dd>
<dd>
<p><code>$var</code> – named variable, value is set in PASSING clause (<em>may be of   datetime type</em>)</p>
</dd>
<dd>
<p><code>@</code> – value of the current item in a filter</p>
</dd>
<dd>
<p><code>last</code> - JSON last subscript of an array</p>
</dd>
<dt><strong>expression in parentheses</strong></dt>
<dd>
<p><code>'($a + 2)'</code></p>
</dd>
<dt><strong>Path elements</strong></dt>
<dd>
<p><strong>member accessor</strong>  – <code>.name</code> or <code>."$name"</code>.  It is used to access a member of an object by key name.  If the key name does not begin with a dollar sign and meets the JavaScript rules of an Identifier, then the member name can be written in clear text, else it can be written as a character string literal. For example: <code>$.color, "$."color of a pen"</code>.  In <strong>strict</strong> mode, every SQL/JSON item in the SQL/JSON sequence must be an object with a member having the specified key name. If this condition is not met, the result is an error.  In <strong>lax</strong> mode, any SQL/JSON array in the SQL/JSON sequence is unwrapped. Unwrapping only goes <strong>one deep</strong>; that is, if there is an array of arrays, the outermost array is unwrapped, leaving the inner arrays alone.</p>
</dd>
<dd>
<p><strong>wildcard member accessor</strong>  –  <code>.*</code>, the values of all attributes of the current  object.  In <strong>strict</strong> mode, every SQL/JSON item in the SQL/JSON sequence must be an object. If this condition is not met, the result is an error. In <strong>lax</strong> mode, any SQL/JSON array in the SQL/JSON sequence is unwrapped.</p>
</dd>
<dd>
<p><strong>array element accessor</strong>  – <code>[1,5 to LAST]</code>, the second and the six-th to the last array elements of the array . Since jsonpath follows javascript conventions rather than SQL, <code>[0]</code> accesses is the first element in the array. <code>LAST</code> is the special variable to handle arrays of unknown length, its value is the size of the array minus 1.   In the <strong>strict</strong> mode, subscripts must be singleton numeric values between 0 and last; in <strong>lax</strong> mode, any subscripts that are out of bound are simply ignored. In both <strong>strict</strong> and <strong>lax</strong> mode, non-numeric subscripts are an error.</p>
</dd>
<dd>
<p><strong>wildcard array element accessor</strong> – <code>[*]</code>, all array elements. In <strong>strict</strong> mode, the operand must be an array, in <strong>lax</strong> mode, if the operand is not an array, then one is provided by wrapping it in an array before unwrapping, <code>$[*]</code> is the same as <code>$[0 to last]</code>.  The latter is not valid in <strong>strict</strong> mode, since <code>$[0 to last]</code> requires at least one array element and raise an error if <code>$</code> is the empty array , while <code>$[*]</code>  returns <code>null</code> in that case.</p>
</dd>
<dd>
<p><strong>filter expression</strong> –  ? (expression) , the result of filter expression may be <code>unknown</code>, <code>true</code>,  <code>false</code>.</p>
</dd>
<dd>
<p><strong>item method</strong> – is the function, that operate on an SQL/JSON item and return an SQL/JSON item.<br>
It denotes as  <code>.</code> and could be one of the 8 methods:</p>
</dd>
<dd>
<p><strong>type()</strong> - returns a character string that names the type of the SQL/JSON item ( <code>"null"</code>, <code>"boolean"</code>, <code>"number"</code>, <code>"string"</code>, <code>"array"</code>, <code>"object"</code>, <code>"date"</code>, <code>"time without time zone"</code>, <code>"time with time zone"</code>, <code>"timestamp without time zone"</code>, <code>"timestamp with time zone"</code>).</p>
</dd>
<dd>
<p><strong>size()</strong> -  returns the size of an SQL/JSON item, which is the number of elements in the array or 1 for SQL/JSON object or scalar in <strong>lax</strong> mode  and error <code>ERROR: SQL/JSON array not found</code> in <strong>strict</strong> mode.</p>
</dd>
</dl>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'$.size()'</span> RETURNING <span class="token keyword">int</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_value
<span class="token comment">------------</span>
          <span class="token number">3</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>In more complex case,  we can wrap SQL/JSON sequence into an array and apply <code>.size()</code> to the result:</p>
<pre><code> SELECT JSON_VALUE(JSON_QUERY('[1,2,3]', '$[*] ? (@ &gt; 1)' WITH WRAPPER), '$.size()' RETURNING int);
 json_value
------------
          2
(1 row)
</code></pre>
<p>Or use our automatic wrapping extension to the path expression</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'[$[*] ? (@ &gt; 1)].size()'</span> RETURNING <span class="token keyword">int</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
?<span class="token keyword">column</span>?
<span class="token comment">----------</span>
       <span class="token number">2</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

</code></pre>
<dl>
<dt>~  <strong>ceiling()</strong> - the same as <code>CEILING</code> in SQL</dt>
<dd><strong>double()</strong> - converts a string or numeric to an approximate numeric value.</dd>
<dd><strong>floor()</strong> - the same as <code>FLOOR</code> in SQL</dd>
<dd><strong>abs()</strong>  - the same as <code>ABS</code> in SQL</dd>
<dd><strong>datetime()</strong> - converts a character string to an SQL datetime type, optionally using a conversion template.</dd>
</dl>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span><span class="token string">'"10-03-2017"'</span><span class="token punctuation">,</span><span class="token string">'$.datetime("dd-mm-yyyy")'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_value
<span class="token comment">------------</span>
 <span class="token number">2017</span><span class="token operator">-</span><span class="token number">03</span><span class="token operator">-</span><span class="token number">10</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span><span class="token string">'"12:34:56"'</span><span class="token punctuation">,</span><span class="token string">'$.datetime().type()'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
       json_value
<span class="token comment">------------------------</span>
 time without time zone
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> JSON_QUERY <span class="token punctuation">(</span>
<span class="token string">'["2017-03-10", "2017-03-11", "2017-03-09",         "12:34:56", "01:02:03 +04", "2017-03-10 00:00:00", "2017-03-10 12:34:56", "2017-03-10 01:02:03 +04", "2017-03-10 03:00:00 +03"]'</span><span class="token punctuation">,</span>
<span class="token string">'$[*].datetime() ? (@ &lt;  "10.03.2017".datetime("dd.mm.yyyy"))'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  json_query
<span class="token comment">--------------</span>
 <span class="token string">"2017-03-09"</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>~ <strong>keyvalue()</strong> - transforms json to an SQL/JSON sequence of objects with a known schema. Example:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span> <span class="token string">'{"a": 123, "b": 456, "c": 789}'</span><span class="token punctuation">,</span> <span class="token string">'$.keyvalue()'</span> <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span><span class="token punctuation">;</span>
                                    ?<span class="token keyword">column</span>?
<span class="token comment">--------------------------------------------------------------------------------------</span>
<span class="token punctuation">[</span>{<span class="token string">"key"</span>: <span class="token string">"a"</span><span class="token punctuation">,</span> <span class="token string">"value"</span>: <span class="token number">123</span>}<span class="token punctuation">,</span> {<span class="token string">"key"</span>: <span class="token string">"b"</span><span class="token punctuation">,</span> <span class="token string">"value"</span>: <span class="token number">456</span>}<span class="token punctuation">,</span> {<span class="token string">"key"</span>: <span class="token string">"c"</span><span class="token punctuation">,</span> <span class="token string">"value"</span>: <span class="token number">789</span>}<span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p><strong>PostgreSQL extensions</strong>:</p>
<ol>
<li><strong>recursive wildcard member accessor</strong> – <code>.**</code>,  recursively applies wildcard member accessor <code>.*</code> to all levels of hierarchy and returns   the values of <strong>all</strong> attributes of the current object regardless of the level of the hierarchy.<br>
Examples:<br>
Wildcard member accessor returns the values of all elements without looking deep.</li>
</ol>
<pre class=" language-sql"><code class="prism  language-sql"> <span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span><span class="token string">'{"a":{"b":[1,2]}, "c":1}'</span><span class="token punctuation">,</span><span class="token string">'$.*'</span> <span class="token keyword">WITH</span> CONDITIONAL WRAPPER<span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
     json_query
<span class="token comment">--------------------</span>
 <span class="token punctuation">[</span>{<span class="token string">"b"</span>: <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span>}<span class="token punctuation">,</span> <span class="token number">1</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Recursive wildcard member accessor “unwraps”  all objects and arrays</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span><span class="token string">'{"a":{"b":[1,2]}, "c":1}'</span><span class="token punctuation">,</span><span class="token string">'$.a.**'</span> <span class="token keyword">WITH</span> CONDITIONAL WRAPPER<span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
          json_query
<span class="token comment">-------------------------------</span>
 <span class="token punctuation">[</span>{<span class="token string">"b"</span>: <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span>}<span class="token punctuation">,</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span><span class="token punctuation">,</span> <span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<ol start="2">
<li><strong>automatic wrapping</strong>  - <code>[path]</code> is equivalent to <code>WITH WRAPPER</code> in SQL_XXX function</li>
</ol>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'[$[*]]'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
?<span class="token keyword">column</span>?
<span class="token comment">-----------</span>
<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'$[*]'</span> <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span><span class="token punctuation">;</span>
?<span class="token keyword">column</span>?
<span class="token comment">-----------</span>
<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="filter-expression">Filter expression</h3>
<p>A filter expression is similar to a WHERE clause in SQL — it is used to remove SQL/JSON items from an SQL/JSON sequence if they do not satisfy a predicate. The syntax uses a question mark <code>?</code> followed by a parenthesized predicate. In <strong>lax</strong> mode, any SQL/JSON arrays in the operand are unwrapped. The predicate is evaluated for each SQL/JSON item in the SQL/JSON sequence. The result is those SQL/JSON items for which the predicate resulted in <code>True</code>.</p>
<h3 id="how-path-expression-works">How path expression works</h3>
<p>Path expression is intended to produce an SQL/JSON sequence ( an ordered list of zero or more SQL/JSON items) and completion code for  SQL/JSON query functions or operator, whose job is to process that result using the particular SQL/JSON query operator. The path engine process path expression step by step, each of which produces SQL/JSON sequence for following step. For example,  path expression</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span>
</code></pre>
<p>at first step produces SQL/JSON sequence of length 1, which is simply json itself - the context item, denoted as <code>$</code>.  Next step is member accessor <code>.floor</code> - the result is SQL/JSON sequence of length 1 - an array <code>floor</code>, which then unwrapped by  wildcard  array element accessor  <code>[*]</code> to SQL/JSON sequence of length 2, containing an array of two objects . Next, <code>.apt</code>  produces two arrays of objects and <code>[*]</code> extracts  SQL/JSON sequence of length 5 ( five objects - appartments), each of which filtered by a filter expression <code>(@.area &gt; 40 &amp;&amp; @.area &lt; 90)</code>, so a result of the whole path expression  is a sequence of two SQL/JSON items.</p>
<p>The final result by jsonpath query operator:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> js @<span class="token operator">*</span>  <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span> <span class="token keyword">from</span> house<span class="token punctuation">;</span>
             ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------------</span>
 {<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}
 {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="links">Links</h3>
<ul>
<li>Github Postgres Professional repository<br>
<a href="https://github.com/postgrespro/sqljson">https://github.com/postgrespro/sqljson</a></li>
<li>WEB-interface to play with SQL/JSON<br>
<a href="http://sqlfiddle.postgrespro.ru/#!21/0/2580">http://sqlfiddle.postgrespro.ru/#!21/0/2580</a></li>
<li>Technical Report (SQL/JSON) - available for free<br>
<a href="http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip">http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip</a></li>
</ul>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

