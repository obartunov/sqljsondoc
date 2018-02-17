---


---

<!--stackedit_data:&amp;#10;eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19&amp;#10;-->
<h2 id="introduction-to-sqljson">Introduction to SQL/JSON</h2>
<p>SQL-2016 standard introduced SQL/JSON data model and path language used by certain SQL/JSON functions to query JSON.   SQL/JSON data model is a sequences of items, each of which is consists of SQL scalar values with an additional SQL/JSON null value,  and composite data structures using JSON arrays and objects.</p>
<p>PostgreSQL has two JSON  data types - the textual json data type to store an exact copy of the input text and the jsonb data type -   the binary storage for  JSON data converted to PostgreSQL types, according  mapping in   <a href="https://www.postgresql.org/docs/current/static/datatype-json.html">json primitive types and corresponding  PostgreSQL types</a>.  SQL/JSON data model adds datetime type to these primitive types, but it only used for comparison operators in path expression and stored on disk as a string.  Thus, jsonb data is already conforms to SQL/JSON data model, while json should be converted according the mapping.  SQL-2016 standard describes two sets of SQL/JSON functions: constructor functions (JSON_OBJECT, JSON_OBJECTAGG, JSON_ARRAY, and JSON_ARRAYAGG) and query functions (JSON_VALUE, JSON_TABLE, JSON_EXISTS, and JSON_QUERY). Constructor functions use values of SQL types and produce JSON values (JSON objects or JSON arrays) represented in SQL character or binary string types. Query functions evaluate SQL/JSON path language expressions against JSON values, producing values of SQL/JSON types, which are converted to SQL types.</p>
<h2 id="sqljson-path-language">SQL/JSON Path language</h2>
<p>The main task of the path language is to specify  the parts (the projection)  of JSON data to be retrieved by path engine for SQL/JSON query functions.  The language is designed to be flexible enough to meet the current needs and to be adaptable to the future use cases. Also, it is integratable into SQL engine, i.e., the semantics of predicates and operators generally follow SQL.  To be friendly to JSON users, the language resembles  JavaScript - dot(<code>.</code>)  used for member access and [] for array access, arrays starts from zero (SQL arrays starts from 1).</p>
<p>Example of two-floors house:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> house<span class="token punctuation">(</span>js<span class="token punctuation">)</span> <span class="token keyword">AS</span> <span class="token keyword">SELECT</span> jsonb <span class="token string">'
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
'</span><span class="token punctuation">;</span>
</code></pre>
<p>Consider the following path expression:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span>
</code></pre>
<p>The result of this path expression will be information about apartments with rooms, which area is  in specified range. Dollar sign <code>$</code>  designates a <strong>context item</strong> or the whole JSON document, which describes a house with floors ( <code>floor[]</code>), apartments (<code>apt[]</code>)  with rooms and room has attribute <code>area</code>.  Expression <code>$.floor[*].apt[*]</code> in described context means  <strong>any</strong> room, which filtered by a filter expression (in parentheses).  At sign <code>@</code> in filter designates the <strong>current item</strong> in filter.   Path expression could have more filters (applied  left to right) and each of them may be nested.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90) ? (@.rooms &gt; 2)'</span>
</code></pre>
<p>It’s possible to use the <strong>path variables</strong> in path expression, whose values are set in <strong>PASSING</strong> clause of invoked SQL/JSON function. For example (js is a column of type JSON):</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; $min &amp;&amp; @.area &lt; $max)'</span> PASSING <span class="token number">40</span> <span class="token keyword">AS</span> min<span class="token punctuation">,</span> <span class="token number">90</span> <span class="token keyword">AS</span> max <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
                               json_query
<span class="token comment">------------------------------------------------------------------------</span>
 <span class="token punctuation">[</span>{<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}<span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p><strong>WITH WRAPPER</strong> clause  is used to wrap the results into array, since  JSON_QUERY output should be  JSON text.  If minimal and maximal values are stored in table <code>area (min integer, max integer)</code>, then it is possible to pass them to the path expression:</p>
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
<p>More complex example with <strong>keyvalue()</strong> method, which outputs an array of  room numbers.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span> js <span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*].keyvalue() ? (@.key == "no").value'</span> <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
   json_query
<span class="token comment">-----------------</span>
 <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">,</span> <span class="token number">4</span><span class="token punctuation">,</span> <span class="token number">5</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h2 id="jsonpath-in-postgresql">JSONPATH in PostgreSQL</h2>
<p>In PostgreSQL the SQL/JSON path language is implemented as  <strong>JSONPATH</strong>  data type - the binary representation of parsed SQL/JSON path expression to effective query JSON data.  Path expression is an optional  path mode (strict | lax), followed by a  path, which is a  sequence of path elements,  started from path  variable, path literal or  expression in parentheses and zero or more operators ( json accessors) .  It  is possible to specify arithmetic or boolean  (<em>PostgreSQL extension</em>) expression on the path. <em>Path can be enclosed in brackets to return an array similar to WITH WRAPPER clause in SQL/JSON query functions. This is a PostgreSQL extension )</em>.</p>
<p>Examples of vaild jsonpath:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor'</span>
<span class="token string">'($+1)'</span>
<span class="token string">'$+1'</span>
<span class="token comment">-- boolean predicate in path, extension</span>
<span class="token string">'($.floor[*].apt[*].area &gt; 10)'</span>
<span class="token string">'$.floor[*].apt[*] ? (@.area == null).no'</span>
</code></pre>
<p>Syntactical errors in jsonpath are reported</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> <span class="token string">'$a. &gt;1'</span>::jsonpath<span class="token punctuation">;</span>
ERROR:  <span class="token number">bad</span> jsonpath representation at character <span class="token number">8</span>
DETAIL:  syntax error<span class="token punctuation">,</span> unexpected GREATER_P at <span class="token operator">or</span> near <span class="token string">"&gt;"</span>
</code></pre>
<p>An <a href="#how-path-expression-works">Example</a> of how path expression works.</p>
<h3 id="jsonpath-operators">Jsonpath operators</h3>
<p>To accelerate JSON path queries using existing indexes for jsonb  (GIN index using built-in  <code>jsonb_ops</code> or <code>jsonb_path_ops</code>)  PostgreSQL extends the standard with two  boolean operators for json[b] and jsonpath data types.</p>
<ul>
<li><code>json[b] @? jsonpath</code> -  exists  operator, returns bool.  Check that path expression returns non-empty SQL/JSON sequence.</li>
<li><code>json[b] @~ jsonpath</code> - match operator, returns the result of boolean predicate (<em>PostgreSQL extension</em>).</li>
</ul>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> js @?  <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?
<span class="token comment">----------</span>
 t
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> js @<span class="token operator">~</span>  <span class="token string">'$.floor[*].apt[*].area &lt;  20'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?
<span class="token comment">----------</span>
 <span class="token number">f</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p><code>jsonb @? jsonpath</code> and <code>jsonb @~ jsonpath</code> are as fast as <code>jsonb @&gt; jsonb</code>  (for equality operation),  but  jsonpath supports more complex expressions, for example:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> <span class="token function">count</span><span class="token punctuation">(</span><span class="token operator">*</span><span class="token punctuation">)</span> <span class="token keyword">FROM</span> house <span class="token keyword">WHERE</span>   js @<span class="token operator">~</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH")  &gt; "1945-03-09".datetime()'</span><span class="token punctuation">;</span>
 count
<span class="token comment">-------</span>
     <span class="token number">1</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Query operators:</p>
<ul>
<li><code>json[b] @* jsonpath</code> - set-query operator,  returns setof json[b].</li>
<li><code>json[b] @# jsonpath</code> - singleton-query operator,  returns a single json[b].<br>
The results is depends on the size of the resulted SQL/JSON sequence:<br>
–  empty sequence - returns SQL NULL;<br>
– single item - returns the item;<br>
– more items - returns array of items.<br>
Notice, that this  behaviour differs from <code>WITH CONDITIONAL WRAPPER</code>, since the latter wraps into array a single scalar value, but not the single object or an array.</li>
</ul>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> js @<span class="token operator">*</span>  <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
             ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------------</span>
 {<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}
 {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> js @<span class="token comment">#  '$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)' FROM house;</span>
                                ?<span class="token keyword">column</span>?
<span class="token comment">------------------------------------------------------------------------</span>
 <span class="token punctuation">[</span>{<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}<span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Operator <code>@#</code> can be used to index jsonb:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">CREATE</span> <span class="token keyword">INDEX</span> bookmarks_oper_path_idx <span class="token keyword">ON</span> bookmarks <span class="token keyword">USING</span> gin<span class="token punctuation">(</span><span class="token punctuation">(</span>js @<span class="token comment"># '$.tags.term') jsonb_path_ops);</span>
<span class="token keyword">EXPLAIN</span> <span class="token punctuation">(</span> <span class="token keyword">ANALYZE</span><span class="token punctuation">,</span> COSTS <span class="token keyword">OFF</span><span class="token punctuation">)</span> <span class="token keyword">SELECT</span> <span class="token function">COUNT</span><span class="token punctuation">(</span><span class="token operator">*</span><span class="token punctuation">)</span> <span class="token keyword">FROM</span> bookmarks <span class="token keyword">WHERE</span> js @<span class="token comment"># '$.tags.term' @&gt; '"NYC"';</span>
                                              QUERY <span class="token keyword">PLAN</span>
<span class="token comment">------------------------------------------------------------------------------------------------------</span>
 Aggregate <span class="token punctuation">(</span>actual time<span class="token operator">=</span><span class="token number">1.136</span><span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token number">1.136</span> <span class="token keyword">rows</span><span class="token operator">=</span><span class="token number">1</span> loops<span class="token operator">=</span><span class="token number">1</span><span class="token punctuation">)</span>
   <span class="token operator">-</span><span class="token operator">&gt;</span>  Bitmap Heap Scan <span class="token keyword">on</span> bookmarks <span class="token punctuation">(</span>actual time<span class="token operator">=</span><span class="token number">0.213</span><span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token number">1.098</span> <span class="token keyword">rows</span><span class="token operator">=</span><span class="token number">285</span> loops<span class="token operator">=</span><span class="token number">1</span><span class="token punctuation">)</span>
         Recheck Cond: <span class="token punctuation">(</span><span class="token punctuation">(</span>js @<span class="token comment"># '$."tags"."term"'::jsonpath) @&gt; '"NYC"'::jsonb)</span>
         Heap Blocks: exact<span class="token operator">=</span><span class="token number">285</span>
         <span class="token operator">-</span><span class="token operator">&gt;</span>  Bitmap <span class="token keyword">Index</span> Scan <span class="token keyword">on</span> bookmarks_oper_path_idx <span class="token punctuation">(</span>actual time<span class="token operator">=</span><span class="token number">0.148</span><span class="token punctuation">.</span><span class="token punctuation">.</span><span class="token number">0.148</span> <span class="token keyword">rows</span><span class="token operator">=</span><span class="token number">285</span> loops<span class="token operator">=</span><span class="token number">1</span><span class="token punctuation">)</span>
               <span class="token keyword">Index</span> Cond: <span class="token punctuation">(</span><span class="token punctuation">(</span>js @<span class="token comment"># '$."tags"."term"'::jsonpath) @&gt; '"NYC"'::jsonb)</span>
 Planning time: <span class="token number">0.228</span> ms
 Execution time: <span class="token number">1.309</span> ms
<span class="token punctuation">(</span><span class="token number">8</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="path-modes">Path modes</h3>
<p>The path engine has two modes, strict and lax, the latter is   default, that is,  the standard tries to facilitate matching of the  (sloppy) document structure and path expression.</p>
<p>In <strong>strict</strong> mode any structural errors (  an attempt to access a non-existent member of an object or element of an array)  raises an error (it is up to JSON_XXX function to actually report it, see <code>ON ERROR</code> clause).<br>
For example:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'strict $.a'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">-- returns ERROR:  SQL/JSON member not found</span>
</code></pre>
<p>Notice,  JSON_VALUE function needs <code>ERROR ON ERROR</code>  to report the error  , since default behaviour  <code>NULL ON ERROR</code> suppresses error reporting and returns <code>null</code>.</p>
<p>In <strong>strict</strong> mode  using an array accessor on a scalar value  or  object triggers error handling.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'strict $[0]'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
ERROR:  SQL<span class="token operator">/</span>JSON array <span class="token operator">not</span> found
<span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'strict $.a'</span> ERROR <span class="token keyword">ON</span> EMPTY ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
ERROR:  SQL<span class="token operator">/</span>JSON member <span class="token operator">not</span> found
</code></pre>
<p>In <strong>lax</strong> mode the path engine suppresses the structural errors and  converts them to the empty SQL/JSON sequences.  Depending on <code>ON EMPTY</code> clause  the empty SQL/JSON sequences  will be  interpreted as <code>null</code> by default ( <code>NULL ON EMPTY</code>) or raise an ERROR ( <code>ERROR ON EMPTY</code>).</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'lax $.a'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_value
<span class="token comment">------------</span>
 <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'lax $.a'</span> ERROR <span class="token keyword">ON</span> EMPTY ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
ERROR:  <span class="token keyword">no SQL</span><span class="token operator">/</span>JSON item
</code></pre>
<p>Also,  in <strong>lax</strong> mode arrays of size 1 is interchangeable with the singleton.</p>
<p>Example of automatic array wrapping in lax mode:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'lax $[0]'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_value
<span class="token comment">------------</span>
 <span class="token number">1</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
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
<p><code>($a + 2)</code></p>
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
<pre class=" language-sql"><code class="prism  language-sql"> <span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>JSON_QUERY<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'$[*] ? (@ &gt; 1)'</span> <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token string">'$.size()'</span> RETURNING <span class="token keyword">int</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_value
<span class="token comment">------------</span>
          <span class="token number">2</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Or use our automatic wrapping extension to the path expression</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'[$[*] ? (@ &gt; 1)].size()'</span> RETURNING <span class="token keyword">int</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
json_value
<span class="token comment">------------</span>
         <span class="token number">2</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<dl>
<dt>~ <strong>ceiling()</strong> - the same as <code>CEILING</code> in SQL</dt>
<dd><strong>double()</strong> - converts a string or numeric to an approximate numeric value.</dd>
<dd><strong>floor()</strong> - the same as <code>FLOOR</code> in SQL</dd>
<dd><strong>abs()</strong>  - the same as <code>ABS</code> in SQL</dd>
<dd><strong>datetime()</strong> - converts a character string to an SQL datetime type, optionally using a conversion template ( <a href="https://www.postgresql.org/docs/current/static/functions-formatting.html">templates examples</a>).</dd>
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
<span class="token comment">-- all datetime examples are obtained with timezone 'W-SU'</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ &lt; "2000-01-01".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
          ?<span class="token keyword">column</span>?           
<span class="token comment">-----------------------------</span>
 <span class="token string">"1957-10-04T19:28:34+00:00"</span>
 <span class="token string">"1961-04-12T06:07:00+00:00"</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- datetime cannot compared to string</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ &lt; "2000-01-01")'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>? 
<span class="token comment">----------</span>
<span class="token punctuation">(</span><span class="token number">0</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- rejected items in previous query</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? ((@ &lt; "2000-01-01") is unknown)'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
          ?<span class="token keyword">column</span>?           
<span class="token comment">-----------------------------</span>
 <span class="token string">"2015-02-01T00:00:00+00:00"</span>
 <span class="token string">"1957-10-04T19:28:34+00:00"</span>
 <span class="token string">"1961-04-12T06:07:00+00:00"</span>
<span class="token punctuation">(</span><span class="token number">3</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- </span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ &gt; "1961-04-12".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
          ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------</span>
 <span class="token string">"2015-02-01T00:00:00+03:00"</span>
 <span class="token string">"1961-04-12T09:07:00+03:00"</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- set timezone = 'UTC';</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ &gt; "1961-04-12".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
          ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------</span>
 <span class="token string">"2015-02-01T00:00:00+00:00"</span>
 <span class="token string">"1961-04-12T06:07:00+00:00"</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy") ? (@ &gt; "1961-04-12".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
   ?<span class="token keyword">column</span>?
<span class="token comment">--------------</span>
 <span class="token string">"2015-02-01"</span>
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
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> jsonb <span class="token string">'{"a":{"b":[1,2]}, "c":1}'</span> @<span class="token operator">*</span> <span class="token string">'$.*'</span><span class="token punctuation">;</span>
   ?<span class="token keyword">column</span>?
<span class="token comment">---------------</span>
 {<span class="token string">"b"</span>: <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span>}
 <span class="token number">1</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>Recursive wildcard member accessor “unwraps”  all objects and arrays</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> jsonb <span class="token string">'{"a":{"b":[1,2]}, "c":1}'</span> @<span class="token operator">*</span> <span class="token string">'$.**'</span><span class="token punctuation">;</span>
           ?<span class="token keyword">column</span>?
<span class="token comment">------------------------------</span>
 {<span class="token string">"a"</span>: {<span class="token string">"b"</span>: <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span>}<span class="token punctuation">,</span> <span class="token string">"c"</span>: <span class="token number">1</span>}
 {<span class="token string">"b"</span>: <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span>}
 <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">]</span>
 <span class="token number">1</span>
 <span class="token number">2</span>
 <span class="token number">1</span>
<span class="token punctuation">(</span><span class="token number">6</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<ol start="2">
<li><strong>automatic wrapping</strong>  - <code>[path]</code> is equivalent to <code>WITH WRAPPER</code> in SQL_XXX function.</li>
</ol>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'[$[*]]'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
json_query
<span class="token comment">------------</span>
<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span><span class="token string">'[1,2,3]'</span><span class="token punctuation">,</span> <span class="token string">'$[*]'</span> <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span><span class="token punctuation">;</span>
json_query
<span class="token comment">------------</span>
<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="filter-expression">Filter expression</h3>
<p>A filter expression is similar to a WHERE clause in SQL, it is used to remove SQL/JSON items from an SQL/JSON sequence if they do not satisfy a predicate. The syntax uses a question mark <code>?</code> followed by a parenthesized predicate. In <strong>lax</strong> mode, any SQL/JSON arrays in the operand are automatically unwrapped. The predicate is evaluated for each SQL/JSON item in the SQL/JSON sequence.  Predicate returns <code>Unknown</code> (SQL NULL) if any error occured during evaluation of its operands and execution. The result is those SQL/JSON items for which the predicate resulted in <code>True</code>, <code>False</code> and <code>Unknown</code> are rejected.</p>
<p>Within a filter, the special variable @ is used to reference the current SQL/JSON item in the SQL/JSON sequence.</p>
<p>The SQL/JSON path language has the following predicates:</p>
<ul>
<li><code>exists</code> predicate, to test if a path expression has a non-empty result.</li>
<li>Comparison predicates ==, !=, &lt;&gt;, &lt;, &lt;=, &gt;, and &gt;=.</li>
<li><code>like_regex</code> for string pattern matching.  Optional parameter<code>flag</code> can be combination of <code>i,s,m,x</code>, default value is <code>s</code>.</li>
<li><code>starts with</code> to test for an initial substring (prefix).</li>
<li><code>is unknown</code> to test for <code>Unknown</code> results. Its operand should be in parentheses.</li>
</ul>
<p>JSON literals <code>true, false</code> are parsed into the SQL/JSON model as the SQL boolean values <code>True</code> and <code>False</code>.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'true'</span><span class="token punctuation">,</span><span class="token string">'$ ? (@ == true)'</span><span class="token punctuation">)</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
 json_value
<span class="token comment">------------</span>
 <span class="token boolean">true</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> jsonb <span class="token string">'{"g": {"x": 2}}'</span> @<span class="token operator">*</span> <span class="token string">'$.g ? (exists (@.x))'</span><span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?
<span class="token comment">----------</span>
 {<span class="token string">"x"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>JSON literal <code>null</code> is parsed into the special SQL/JSON value <code>null</code>, which differs from SQL NULL, for example, SQL JSON <code>null</code> is equal to itself, so the result of <code>null == null</code> is <code>True</code>.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> json_query<span class="token punctuation">(</span><span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'$ ? (null == null)'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_query
<span class="token comment">------------</span>
 <span class="token number">1</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> json_query<span class="token punctuation">(</span><span class="token string">'1'</span><span class="token punctuation">,</span> <span class="token string">'$ ? (null != null)'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_query
<span class="token comment">------------</span>
 <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Prefix search with <code>starts with</code> example:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.** ? (@ starts with "11")'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
            ?<span class="token keyword">column</span>?
<span class="token comment">---------------------------------</span>
 <span class="token string">"117036, Dmitriya Ulyanova, 7A"</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Regular expression search:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token comment">-- case insensitive</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.** ? (@ like_regex "O(w|v)" flag "i")'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
            ?<span class="token keyword">column</span>?
<span class="token comment">---------------------------------</span>
 <span class="token string">"Moscow"</span>
 <span class="token string">"117036, Dmitriya Ulyanova, 7A"</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- ignore spaces in query, flag "x"</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.** ? (@ like_regex "O w|o V" flag "ix")'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
            ?<span class="token keyword">column</span>?
<span class="token comment">---------------------------------</span>
 <span class="token string">"Moscow"</span>
 <span class="token string">"117036, Dmitriya Ulyanova, 7A"</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- single-line mode, flag "s".</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.** ? (@ like_regex "^info@" flag "is")'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?
<span class="token comment">----------</span>
<span class="token punctuation">(</span><span class="token number">0</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- multi-line mode, flag "m"</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.** ? (@ like_regex "^info@" flag "im")'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
                             ?<span class="token keyword">column</span>?
<span class="token comment">------------------------------------------------------------------</span>
 <span class="token string">"Postgres Professional\n+7 (495) 150-06-91\ninfo@postgrespro.ru"</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Predicate <code>is unknown</code> can be used to filter “problematic” SQL/JSON items. Notice, that filter expression automatically unwraps array <code>apt</code>, since default mode is <code>lax</code>.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.floor.apt ? ((@.area / @.rooms &gt; 0))'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
              ?<span class="token keyword">column</span>?
<span class="token comment">------------------------------------</span>
 {<span class="token string">"no"</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">40</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">1</span>}
 {<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}
 {<span class="token string">"no"</span>: <span class="token number">4</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">100</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}
 {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">4</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- what item was rejected by a filter ?</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.floor.apt ? ((@.area / @.rooms &gt; 0) is unknown)'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
              ?<span class="token keyword">column</span>?
<span class="token comment">-------------------------------------</span>
 {<span class="token string">"no"</span>: <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="how-path-expression-works">How path expression works</h3>
<p>Path expression is intended to produce an SQL/JSON sequence ( an ordered list of zero or more SQL/JSON items) and completion code for  SQL/JSON query functions or operator, whose job is to process that result using the particular SQL/JSON query operator. The path engine process path expression step by step, each of which produces SQL/JSON sequence for following step. For example,  path expression</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span>
</code></pre>
<p>at first step produces SQL/JSON sequence of length 1, which is simply json itself - the context item, denoted as <code>$</code>.  Next step is member accessor <code>.floor</code> - the result is SQL/JSON sequence of length 1 - an array <code>floor</code>, which then unwrapped by  wildcard  array element accessor  <code>[*]</code> to SQL/JSON sequence of length 2, containing an array of two objects . Next, <code>.apt</code>  produces two arrays of objects and <code>[*]</code> extracts  SQL/JSON sequence of length 5 ( five objects - appartments), each of which filtered by a filter expression <code>(@.area &gt; 40 &amp;&amp; @.area &lt; 90)</code>, so a result of the whole path expression  is a sequence of two SQL/JSON items.</p>
<pre><code>SELECT JSON_QUERY(js,'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)' WITH WRAPPER) FROM house;
                               json_query
------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
</code></pre>
<p>The result by jsonpath set-query operator:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> js @<span class="token operator">*</span>  <span class="token string">'$.floor[*].apt[*] ? (@.area &gt; 40 &amp;&amp; @.area &lt; 90)'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
             ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------------</span>
 {<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}
 {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="sqljson-conformance">SQL/JSON conformance</h3>
<ul>
<li><code>like_regex</code> supports posix regular expressions,  while standard requires xquery regexps.</li>
<li>Not supported (due to unresolved conflicts in SQL grammar):
<ul>
<li>expr FORMAT JSON IS [NOT] JSON</li>
<li>JSON_OBJECT(KEY key VALUE value, …)</li>
<li>JSON_ARRAY(SELECT … FORMAT JSON …)</li>
<li>JSON_ARRAY(SELECT … (ABSENT|NULL) ON NULL …)</li>
</ul>
</li>
<li>Only error codes are returned for the failed arithmetic operations inside jsonpath, error messages are lost</li>
<li>Use boolean  expression on the path, PostgreSQL extension</li>
<li><code>.**</code>  - recursive wildcard member accessor, PostgreSQL extension</li>
<li>json[b] op jsonpath - PostgreSQL extension</li>
<li>[path] - wrap SQL/JSON sequences into an array - PostgreSQL extension</li>
</ul>
<h3 id="links">Links</h3>
<ul>
<li>Github Postgres Professional repository<br>
<a href="https://github.com/postgrespro/sqljson">https://github.com/postgrespro/sqljson</a></li>
<li>WEB-interface to play with SQL/JSON<br>
<a href="http://sqlfiddle.postgrespro.ru/#!21/0/2580">http://sqlfiddle.postgrespro.ru/#!21/0/2580</a></li>
<li>Technical Report (SQL/JSON) - available for free<br>
<a href="http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip">http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip</a></li>
<li>Jsonb roadmap - talk at <a href="http://PGConf.eu">PGConf.eu</a>, 2018<br>
<a href="http://www.sai.msu.su/~megera/postgres/talks/sqljson-pgconf.eu-2017.pdf">http://www.sai.msu.su/~megera/postgres/talks/sqljson-pgconf.eu-2017.pdf</a></li>
</ul>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>
<!--stackedit_data:&#10;eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19&#10;-->
<!--stackedit_data:&#10;eyJoaXN0b3J5IjpbNTgwMjQzOTRdfQ==&#10;-->

