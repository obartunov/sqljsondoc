---


---

<!--stackedit_data:&amp;#10;eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19&amp;#10;-->
<h2 id="introduction-to-sqljson">Introduction to SQL/JSON</h2>
<p>SQL-2016 standard doesn’t describes the JSON data type, but instead it  introduced SQL/JSON data model (not  JSON data type like XML )  with string storage and path language used by certain SQL/JSON functions to query JSON.   SQL/JSON data model is a sequences of items, each of which is consists of SQL scalar values with an additional SQL/JSON null value,  and composite data structures using JSON arrays and objects.</p>
<p>PostgreSQL has two JSON  data types - the textual json data type to store an exact copy of the input text and the jsonb data type -   the binary storage for  JSON data converted to PostgreSQL types, according  mapping in   <a href="https://www.postgresql.org/docs/current/static/datatype-json.html">json primitive types and corresponding  PostgreSQL types</a>.  SQL/JSON data model adds datetime type to these primitive types, but it only used for comparison operators in path expression and stored on disk as a string.  Thus, jsonb data is already conforms to SQL/JSON data model (ORDERED and UNIQUE KEYS), while json should be converted according the mapping.  SQL-2016 standard describes two sets of SQL/JSON functions : constructor functions and query functions.  Constructor functions use values of SQL types and produce JSON values (JSON objects or JSON arrays) represented in SQL character or binary string types. Query functions evaluate SQL/JSON path language expressions against JSON values, producing values of SQL/JSON types, which are converted to SQL types.</p>
<h2 id="sqljson-path-language">SQL/JSON Path language</h2>
<p>The main task of the path language is to specify  the parts (the projection)  of JSON data to be retrieved by path engine for the  SQL/JSON query functions.  The language is designed to be flexible enough to meet the current needs and to be adaptable to the future use cases. Also, it is integratable into SQL engine, i.e., the semantics of predicates and operators generally follow SQL.  To be friendly to JSON users, the language resembles  JavaScript - dot(<code>.</code>)  used for member access and <code>[]</code> for array access, arrays starts from zero (SQL arrays starts from 1).</p>
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
<p>It can be illustrated as <a href="https://ic.pics.livejournal.com/obartunov/24248903/45289/45289_original.png">this picture</a>.</p>
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
<p>In PostgreSQL the SQL/JSON path language is implemented as  <strong>JSONPATH</strong>  data type - the binary representation of parsed SQL/JSON path expression to effective query JSON data.  <strong>Path expression</strong> is an optional  path mode (strict | lax), followed by a  path, which is a  sequence of path elements,  started from path  variable, path literal or  expression in parentheses and zero or more operators ( json accessors) .  It  is possible to specify arithmetic or boolean  (<em>PostgreSQL extension</em>) expression on the path. <em>Path can be enclosed in brackets to return an array similar to WITH WRAPPER clause in SQL/JSON query functions. This is a PostgreSQL extension )</em>.</p>
<p>Examples of vaild jsonpath:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$.floor'</span>
<span class="token string">'($+1)'</span>
<span class="token string">'$+1'</span>
<span class="token comment">-- boolean predicate in path, extension</span>
<span class="token string">'($.floor[*].apt[*].area &gt; 10)'</span>
<span class="token string">'$.floor[*].apt[*] ? (@.area == null).no'</span>
</code></pre>
<p>Jsonpath could be an expression:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token string">'$'</span> <span class="token operator">||</span> <span class="token string">'.'</span> <span class="token operator">||</span> <span class="token string">'a'</span>
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
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> <span class="token function">count</span><span class="token punctuation">(</span><span class="token operator">*</span><span class="token punctuation">)</span> <span class="token keyword">FROM</span> house <span class="token keyword">WHERE</span>   js @<span class="token operator">~</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy")  &gt; "1945-03-09".datetime()'</span><span class="token punctuation">;</span>
 count
<span class="token comment">-------</span>
     <span class="token number">1</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<p>Operators exists @? and match @~  can be speeded up by GIN index using built-in <code>jsonb_ops</code> or <code>jsonb_path_ops</code> opclasses.</p>
<pre><code>SELECT COUNT(*) FROM bookmarks 
WHERE jb @? '$.tags[*] ? (@.term == "NYC")';
                                       QUERY PLAN
------------------------------------------------------------------------------------------------
 Aggregate (actual time=0.529..0.529 rows=1 loops=1)
   -&gt;  Bitmap Heap Scan on bookmarks (actual time=0.080..0.502 rows=285 loops=1)
         Recheck Cond: (jb @? '$."tags"[*]?(@."term" == "NYC")'::jsonpath)
         Heap Blocks: exact=285
         -&gt;  Bitmap Index Scan on bookmarks_jb_path_idx (actual time=0.045..0.045 rows=285 loops=1)
               Index Cond: (jb @? '$."tags"[*]?(@."term" == "NYC")'::jsonpath)
 Planning time: 0.053 ms
 Execution time: 0.553 ms
(8 rows)

SELECT COUNT(*) FROM bookmarks 
WHERE jb @~ '$.tags[*].term == "NYC"';
                                        QUERY PLAN                                               -----------------------------------------------------------------------------------------------
Aggregate (actual time=0.930..0.930 rows=1 loops=1)
   -&gt;  Bitmap Heap Scan on bookmarks (actual time=0.133..0.884 rows=285 loops=1)
         Recheck Cond: (jb @~ '($."tags"[*]."term" == "NYC")'::jsonpath)
         Heap Blocks: exact=285
         -&gt;  Bitmap Index Scan on bookmarks_jb_path_idx (actual time=0.073..0.073 rows=285 loops=1)
               Index Cond: (jb @~ '($."tags"[*]."term" == "NYC")'::jsonpath)
 Planning time: 0.135 ms
 Execution time: 0.973 ms
(8 rows)
</code></pre>
<p>Query operators:</p>
<ul>
<li><code>json[b] @* jsonpath</code> - set-query operator,  returns setof json[b].</li>
<li><code>json[b] @# jsonpath</code> - singleton-query operator,  returns a single json[b].<br>
The results is depends on the size of the resulted SQL/JSON sequence:<br>
–  empty sequence - returns SQL NULL;<br>
– single item - returns the item;<br>
– more items - returns an array of items.<br>
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
<dd><strong>datetime()</strong> - converts a character string to an SQL datetime type, optionally using a conversion template ( <a href="https://www.postgresql.org/docs/current/static/functions-formatting.html">templates examples</a>) . Default template is ISO - “yyyy-dd-mm”.    Default timezone could be specified  as a second argument of <strong>datetime()</strong> function, it applied  only if  template contains <strong>TZH</strong> and there is no 	timezone in input data.  That helps to keep jsonpath operators and functions to be immutable and, hence, indexable.</dd>
</dl>
<p>PostgreSQL adds support of  conversion of UNIX epoch (double) to timestamptz.</p>
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

<span class="token comment">-- SQL Standard is strict about matching of data and template</span>
<span class="token comment">-- An error occurs if some data doesn't contains timezone</span>

<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ &lt; "2000-01-01".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
ERROR:  Invalid argument <span class="token keyword">for</span> SQL<span class="token operator">/</span>JSON <span class="token keyword">datetime</span> <span class="token keyword">function</span>

<span class="token comment">-- Identify "problematic" dates (no TZH specified)</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*] ? ((@.datetime("dd-mm-yy hh24:mi:ss TZH") &lt; "2000-01-01".datetime()) is unknown)'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
   ?<span class="token keyword">column</span>?
<span class="token comment">--------------</span>
 <span class="token string">"01-02-2015"</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token comment">-- Add default timezone</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH","+3") ? (@ &lt; "2000-01-01".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
          ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------</span>
 <span class="token string">"1957-10-04T19:28:34+00:00"</span>
 <span class="token string">"1961-04-12T09:07:00+03:00"</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- datetime cannot compared to string</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss") ? (@ &lt; "2000-01-01")'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>? 
<span class="token comment">----------</span>
<span class="token punctuation">(</span><span class="token number">0</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- rejected items in previous query</span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss") ? ((@ &lt; "2000-01-01") is unknown)'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
       ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------</span>
 <span class="token string">"2015-02-01T00:00:00"</span>
 <span class="token string">"1957-10-04T19:28:34"</span>
 <span class="token string">"1961-04-12T09:07:00"</span>
<span class="token punctuation">(</span><span class="token number">3</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token comment">-- </span>
<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss") ? (@ &gt; "1961-04-12".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
       ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------</span>
 <span class="token string">"2015-02-01T00:00:00"</span>
 <span class="token string">"1961-04-12T09:07:00"</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> js @<span class="token operator">*</span> <span class="token string">'$.info.dates[*].datetime("dd-mm-yy") ? (@ &gt; "1961-04-12".datetime())'</span> <span class="token keyword">FROM</span> house<span class="token punctuation">;</span>
   ?<span class="token keyword">column</span>?
<span class="token comment">--------------</span>
 <span class="token string">"2015-02-01"</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token comment">-- UNIX epoch to timestamptz</span>
<span class="token keyword">SELECT</span> jsonb <span class="token string">'0'</span> @<span class="token operator">*</span> <span class="token string">'$.datetime().type()'</span><span class="token punctuation">;</span>
          ?<span class="token keyword">column</span>?
<span class="token comment">----------------------------</span>
 <span class="token string">"timestamp with time zone"</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> jsonb <span class="token string">'0'</span> @<span class="token operator">*</span> <span class="token string">'$.datetime()'</span><span class="token punctuation">;</span>
          ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------</span>
 <span class="token string">"1970-01-01T00:00:00+00:00"</span>
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
<h2 id="sqljson-functions">SQL/JSON functions</h2>
<h3 id="introduction-to-sqljson-functions">Introduction to SQL/JSON functions</h3>
<p>The SQL/JSON construction functions use values of SQL types and produce JSON values (JSON objects or JSON arrays) represented in SQL character or binary string types. They are mostly the same as corresponding json[b] construction functions.</p>
<ul>
<li>JSON_OBJECT -  construct a JSON[b] object</li>
<li>JSON_ARRAY -  construct a JSON[b] array</li>
<li>JSON_ARRAYAGG -  aggregates values as JSON[b] array</li>
<li>JSON_OBJECTAGG - aggregates name/value pairs  as JSON[b] object</li>
</ul>
<p>The SQL/JSON retrieval functions evaluate SQL/JSON path language expressions against JSON values, producing values of SQL/JSON types, which are converted to SQL types.  The SQL/JSON query functions all need a path specification, the JSON value to be input to that path specification for querying and processing, and optional parameter values passed to the path specification.</p>
<ul>
<li>JSON_VALUE - Extract an SQL value of a predefined type from a JSON value.</li>
<li>JSON_QUERY - Extract a JSON text from a JSON text using an SQL/JSON path expression.</li>
<li>JSON_TABLE - Query a JSON text and present it as a relational table.</li>
<li>IS [NOT] JSON - test whether a string value is a JSON text.</li>
<li>JSON_EXISTS - determines whether a JSON value satisfies a search criterion.</li>
</ul>
<h3 id="common-syntax-elements">Common syntax elements:</h3>
<pre><code>json_value_expression ::=
    expression [ FORMAT json_representation ]
json_representation ::=
	JSON [ ENCODING { UTF8 | UTF16 | UTF32 } ]
json_output_clause ::=
	RETURNING data_type [ FORMAT json_representation ]
json_api_common_syntax ::=
	json_value_expression , json_path_specification
	[ PASSING { json_value_expression AS identifier }[,...] 	]
json_path_specification ::= jsonpath
</code></pre>
<p>Note:</p>
<ul>
<li>Only UTF8 encoding is supported</li>
<li>data_type in RETURNING could be one of json (default), jsonb, text,  bytea.  PostgreSQL extends the supported types to  records, arrays and domains.</li>
<li>json_representation could be only <code>json</code>, so FORMAT is implemented only for the standard compatibility.</li>
<li>The standard requires <code>character_string_literal</code> in json_path_specification,  PostgreSQL extends json_path_specification to a separate data type <code>jsonpath</code>.</li>
</ul>
<h3 id="error-handling">Error handling</h3>
<p>Since SQL standard describes using  strings for JSON  data and not  data type, parsing errors occurs not when casting string into the JSON type, so all corresponding SQL/JSON functions have special <strong>ON ERROR</strong>  clause, which specifies SQL/JSON function error behavior.</p>
<h3 id="json_object---construct-a-jsonb-object">JSON_OBJECT - construct a JSON[b] object</h3>
<p>Internally transformed into a json[b]_build_object call.<br>
Syntax:</p>
<pre><code>JSON_OBJECT (
  [ { expression {VALUE | ':'} json_value_expression }[,…] ]
  [ { NULL | ABSENT } ON NULL ]
  [ { WITH | WITHOUT } UNIQUE [ KEYS ] ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)

json_value_expression ::= expression [ FORMAT JSON ]
</code></pre>
<ul>
<li>RETURNING type:<br>
– jsonb_build_object() is used for  RETURNING jsonb<br>
– json_build_object() - for other types</li>
<li>Key uniqueness check: {WITH|WITHOUT} UNIQUES [KEYS]</li>
<li>ability to omit keys with NULL values: {ABSENT|NULL} ON NULL</li>
</ul>
<p>Examples:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'a'</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">'a'</span>: <span class="token number">2</span> <span class="token keyword">WITH</span> <span class="token keyword">UNIQUE</span> <span class="token keyword">KEYS</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
ERROR: duplicate JSON <span class="token keyword">key</span> <span class="token string">"a"</span>

<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'a'</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">'a'</span>: <span class="token number">2</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
      ?<span class="token keyword">column</span>?      
<span class="token comment">--------------------</span>
 {<span class="token string">"a"</span> : <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"a"</span> : <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'a'</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">'a'</span>: <span class="token number">2</span> RETURNING jsonb<span class="token punctuation">)</span><span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?      
<span class="token comment">----------</span>
 {<span class="token string">"a"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token comment">-- Omitting keys with NULL values (keys  are not allowed to be NULL):</span>
<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'a'</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">'b'</span>: <span class="token boolean">NULL</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
       ?<span class="token keyword">column</span>?        
<span class="token comment">-----------------------</span>
 {<span class="token string">"a"</span> : <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"b"</span> : <span class="token boolean">null</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'a'</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">'b'</span>: <span class="token boolean">NULL</span> ABSENT <span class="token keyword">ON</span> <span class="token boolean">NULL</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?  
<span class="token comment">-----------</span>
 {<span class="token string">"a"</span> : <span class="token number">1</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'area'</span> : <span class="token number">20</span> <span class="token operator">+</span> <span class="token number">30</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">'no'</span>: <span class="token number">5</span><span class="token punctuation">)</span> <span class="token keyword">AS</span> apt<span class="token punctuation">;</span>
                 apt                  
<span class="token comment">--------------------------------------</span>
 {<span class="token string">"area"</span> : <span class="token number">50</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"no"</span> : <span class="token number">5</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token comment">-- by default json type is returned</span>
<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'area'</span> : <span class="token number">20</span> <span class="token operator">+</span> <span class="token number">30</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">'no'</span>: <span class="token number">5</span><span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span><span class="token string">'no'</span> <span class="token keyword">AS</span> apt<span class="token punctuation">;</span>
 apt 
<span class="token comment">-----</span>
 <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token comment">-- jsonb: fields are ordered, duplicate fields removed</span>
<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'area'</span> : <span class="token number">20</span> <span class="token operator">+</span> <span class="token number">30</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">'no'</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">'area'</span> : <span class="token boolean">NULL</span> RETURNING jsonb<span class="token punctuation">)</span> <span class="token keyword">AS</span> apt<span class="token punctuation">;</span>
                apt                 
<span class="token comment">-------------------------------------</span>
 {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token comment">-- WITH UNIQUE</span>
<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'area'</span> : <span class="token number">20</span> <span class="token operator">+</span> <span class="token number">30</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">'no'</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">'area'</span> : <span class="token boolean">NULL</span> <span class="token keyword">WITH</span> <span class="token keyword">UNIQUE</span> <span class="token keyword">KEYS</span> RETURNING jsonb<span class="token punctuation">)</span> <span class="token keyword">AS</span> apt<span class="token punctuation">;</span>
ERROR:  duplicate JSON <span class="token keyword">key</span> <span class="token string">"area"</span>
<span class="token comment">-- ABSENT ON NULL (last "area" key is skipped)</span>
<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'area'</span> : <span class="token number">20</span> <span class="token operator">+</span> <span class="token number">30</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">'no'</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">'area'</span> : <span class="token boolean">NULL</span> ABSENT <span class="token keyword">ON</span> <span class="token boolean">NULL</span> RETURNING jsonb<span class="token punctuation">)</span> <span class="token keyword">AS</span> apt<span class="token punctuation">;</span>
                apt                
<span class="token comment">-----------------------------------</span>
 {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">50</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token comment">-- ABSENT ON NULL + WITH UNIQUE (key uniqueness is checked before NULLs are removed)</span>
<span class="token keyword">SELECT</span> JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'area'</span> : <span class="token number">20</span> <span class="token operator">+</span> <span class="token number">30</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">'no'</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">'area'</span> : <span class="token boolean">NULL</span> ABSENT <span class="token keyword">ON</span> <span class="token boolean">NULL</span> <span class="token keyword">WITH</span> <span class="token keyword">UNIQUE</span> <span class="token keyword">KEYS</span> RETURNING jsonb<span class="token punctuation">)</span> <span class="token keyword">AS</span> apt<span class="token punctuation">;</span>
ERROR:  duplicate JSON <span class="token keyword">key</span> <span class="token string">"area"</span>
</code></pre>
<h3 id="json_objectagg---aggregates-namevalue-pairs--as-jsonb-object">JSON_OBJECTAGG - aggregates name/value pairs  as JSON[b] object</h3>
<p>JSON_OBJECTAGG is transformed into a json[b]_object_agg depending on RETURNING type.<br>
Syntax:</p>
<pre><code>JSON_OBJECTAGG (
  expression { VALUE | ':' } expression [ FORMAT JSON ]
  [ { NULL | ABSENT } ON NULL ]
  [ { WITH | WITHOUT } UNIQUE [ KEYS ] ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)
</code></pre>
<p>Options and RETURNING clause are the same as in JSON_OBJECT.</p>
<p>Examples:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_OBJECTAGG<span class="token punctuation">(</span><span class="token string">'key'</span> <span class="token operator">||</span> i : <span class="token string">'val'</span> <span class="token operator">||</span>  i<span class="token punctuation">)</span>
   <span class="token keyword">FROM</span> generate_series<span class="token punctuation">(</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">)</span> i<span class="token punctuation">;</span>
                       ?<span class="token keyword">column</span>?
<span class="token comment">-------------------------------------------------------</span>
 { <span class="token string">"key1"</span> : <span class="token string">"val1"</span><span class="token punctuation">,</span> <span class="token string">"key2"</span> : <span class="token string">"val2"</span><span class="token punctuation">,</span> <span class="token string">"key3"</span> : <span class="token string">"val3"</span> }
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> JSON_OBJECTAGG<span class="token punctuation">(</span>k: v ABSENT <span class="token keyword">ON</span> <span class="token boolean">NULL</span><span class="token punctuation">)</span> <span class="token keyword">AS</span> apt
<span class="token keyword">FROM</span> <span class="token punctuation">(</span><span class="token keyword">VALUES</span> <span class="token punctuation">(</span><span class="token string">'no'</span><span class="token punctuation">,</span> <span class="token number">5</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token punctuation">(</span><span class="token string">'area'</span><span class="token punctuation">,</span> <span class="token number">50</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token punctuation">(</span><span class="token string">'rooms'</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token punctuation">(</span><span class="token string">'foo'</span><span class="token punctuation">,</span> <span class="token boolean">NULL</span><span class="token punctuation">)</span><span class="token punctuation">)</span> kv<span class="token punctuation">(</span>k<span class="token punctuation">,</span> v<span class="token punctuation">)</span><span class="token punctuation">;</span>
                  apt
<span class="token comment">----------------------------------------</span>
 { <span class="token string">"no"</span> : <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">50</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">2</span> }
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="json_array---construct-a-jsonb-array">JSON_ARRAY - construct a JSON[b] array</h3>
<p>Internally transformed into a json[b]_build_array() call.<br>
Syntax:</p>
<pre><code>JSON_ARRAY (
  [ { expression [ FORMAT JSON ] }[, ...] ]
  [ { NULL | ABSENT } ON NULL ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)

JSON_ARRAY (
  query_expression
  [ RETURNING data_type [ FORMAT JSON ] ]
)
</code></pre>
<p>Options and RETURNING clause are the same as in JSON_OBJECT.<br>
Note: ON NULL clause is not supported in subquery variant.</p>
<p>Examples:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_ARRAY<span class="token punctuation">(</span><span class="token string">'string'</span><span class="token punctuation">,</span> <span class="token number">123</span><span class="token punctuation">,</span> <span class="token boolean">TRUE</span><span class="token punctuation">,</span> ARRAY<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span><span class="token number">2</span><span class="token punctuation">,</span><span class="token number">3</span><span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">'{"a": 1}'</span>::jsonb<span class="token punctuation">,</span> <span class="token string">'[1, {"c": 3}]'</span> FORMAT JSON
<span class="token punctuation">)</span><span class="token punctuation">;</span>
                       ?<span class="token keyword">column</span>?
<span class="token comment">-----------------------------------------------------------</span>
 <span class="token punctuation">[</span><span class="token string">"string"</span><span class="token punctuation">,</span> <span class="token number">123</span><span class="token punctuation">,</span> <span class="token boolean">true</span><span class="token punctuation">,</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span><span class="token number">2</span><span class="token punctuation">,</span><span class="token number">3</span><span class="token punctuation">]</span><span class="token punctuation">,</span> {<span class="token string">"a"</span>: <span class="token number">1</span>}<span class="token punctuation">,</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> {<span class="token string">"c"</span>: <span class="token number">3</span>}<span class="token punctuation">]</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> JSON_ARRAY<span class="token punctuation">(</span><span class="token keyword">SELECT</span> <span class="token operator">*</span> <span class="token keyword">FROM</span> generate_series<span class="token punctuation">(</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?  
<span class="token comment">-----------</span>
 <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span> JSON_ARRAY<span class="token punctuation">(</span>
  json <span class="token string">'{ "no": 1, "area": 40, "rooms": 1 }'</span><span class="token punctuation">,</span>
  JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'no'</span>: <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">'area'</span>: <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">2</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
  JSON_OBJECT<span class="token punctuation">(</span><span class="token string">'no'</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">'area'</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">'rooms'</span>: <span class="token number">3</span> RETURNING <span class="token keyword">text</span><span class="token punctuation">)</span>
<span class="token punctuation">)</span> <span class="token keyword">AS</span> apts<span class="token punctuation">;</span>
                                                            apts                                                             
<span class="token comment">-----------------------------------------------------------------------------------------------------------------------------</span>
 <span class="token punctuation">[</span>{ <span class="token string">"no"</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">40</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">1</span> }<span class="token punctuation">,</span> {<span class="token string">"no"</span> : <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">2</span>}<span class="token punctuation">,</span> <span class="token string">"{\"no\" : 2, \"area\" : 80, \"rooms\" : 3}"</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="json_arrayagg---aggregates-values-as-jsonb-array">JSON_ARRAYAGG - aggregates values as JSON[b] array</h3>
<p>Internally transformed into a json[b]_agg() call.<br>
Syntax:</p>
<pre><code>JSON_ARRAYAGG (
  expression [ FORMAT JSON ]
  [ { NULL | ABSENT } ON NULL ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)
</code></pre>
<p>Examples:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_ARRAYAGG<span class="token punctuation">(</span>i<span class="token punctuation">)</span> <span class="token keyword">FROM</span> generate_series<span class="token punctuation">(</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">)</span> i<span class="token punctuation">;</span>
 ?<span class="token keyword">column</span>?  
<span class="token comment">-----------</span>
 <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token comment">-- transform json data into relational form</span>
<span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> house_apt <span class="token keyword">AS</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      <span class="token keyword">no</span> <span class="token keyword">int</span><span class="token punctuation">,</span>
      area <span class="token keyword">float</span><span class="token punctuation">,</span>
      rooms <span class="token keyword">int</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
  
<span class="token keyword">SELECT</span> <span class="token operator">*</span> <span class="token keyword">FROM</span> house_apt<span class="token punctuation">;</span>
 level <span class="token operator">|</span> <span class="token keyword">no</span> <span class="token operator">|</span>  area  <span class="token operator">|</span> rooms 
<span class="token comment">-------+----+--------+-------</span>
     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">1</span> <span class="token operator">|</span>     <span class="token number">40</span> <span class="token operator">|</span>     <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">2</span> <span class="token operator">|</span>     <span class="token number">80</span> <span class="token operator">|</span>     <span class="token number">3</span>
     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>     <span class="token number">2</span>
     <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">4</span> <span class="token operator">|</span>    <span class="token number">100</span> <span class="token operator">|</span>     <span class="token number">3</span>
     <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">5</span> <span class="token operator">|</span>     <span class="token number">60</span> <span class="token operator">|</span>     <span class="token number">2</span>
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span>
  JSON_OBJECT<span class="token punctuation">(</span>
    <span class="token string">'level'</span>: level<span class="token punctuation">,</span>
    <span class="token string">'apt'</span>: JSON_ARRAYAGG<span class="token punctuation">(</span>
      JSON_OBJECT<span class="token punctuation">(</span>
        <span class="token string">'no'</span>: <span class="token keyword">no</span><span class="token punctuation">,</span>
        <span class="token string">'area'</span>: area<span class="token punctuation">,</span>
        <span class="token string">'rooms'</span>: rooms
      <span class="token punctuation">)</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">AS</span> floor
<span class="token keyword">FROM</span> house_apt
<span class="token keyword">GROUP</span> <span class="token keyword">BY</span> level<span class="token punctuation">;</span>
                                                              floor                                                                    
<span class="token comment">---------------------------------------------------------------------------------------------------------------------------------------------</span>
 {<span class="token string">"level"</span> : <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"apt"</span> : <span class="token punctuation">[</span>{<span class="token string">"no"</span> : <span class="token number">4</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">100</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span> : <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">2</span>}<span class="token punctuation">]</span>}
 {<span class="token string">"level"</span> : <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"apt"</span> : <span class="token punctuation">[</span>{<span class="token string">"no"</span> : <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">40</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">1</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span> : <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span> : <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">2</span>}<span class="token punctuation">]</span>}
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- ABSENT ON NULL by default</span>
<span class="token keyword">SELECT</span> JSON_ARRAYAGG<span class="token punctuation">(</span>area<span class="token punctuation">)</span> <span class="token keyword">AS</span> areas
<span class="token keyword">FROM</span> house_apt<span class="token punctuation">;</span>
       areas       
<span class="token comment">-------------------</span>
 <span class="token punctuation">[</span><span class="token number">40</span><span class="token punctuation">,</span> <span class="token number">80</span><span class="token punctuation">,</span> <span class="token number">100</span><span class="token punctuation">,</span> <span class="token number">60</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token comment">-- NULL ON NULL</span>
<span class="token keyword">SELECT</span> JSON_ARRAYAGG<span class="token punctuation">(</span>area <span class="token boolean">NULL</span> <span class="token keyword">ON</span> <span class="token boolean">NULL</span><span class="token punctuation">)</span> <span class="token keyword">AS</span> areas
<span class="token keyword">FROM</span> house_apt<span class="token punctuation">;</span>
          areas          
<span class="token comment">-------------------------</span>
 <span class="token punctuation">[</span><span class="token number">40</span><span class="token punctuation">,</span> <span class="token number">80</span><span class="token punctuation">,</span> <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token number">100</span><span class="token punctuation">,</span> <span class="token number">60</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

<span class="token comment">-- record type serializing</span>
<span class="token keyword">SELECT</span> JSON_ARRAYAGG<span class="token punctuation">(</span>house_apt<span class="token punctuation">.</span><span class="token operator">*</span><span class="token punctuation">)</span> <span class="token keyword">AS</span> floor
<span class="token keyword">FROM</span> house_apt
<span class="token keyword">GROUP</span> <span class="token keyword">BY</span> level<span class="token punctuation">;</span>
                   floor                    
<span class="token comment">--------------------------------------------</span>
 <span class="token punctuation">[</span>{<span class="token string">"level"</span>:<span class="token number">2</span><span class="token punctuation">,</span><span class="token string">"no"</span>:<span class="token number">4</span><span class="token punctuation">,</span><span class="token string">"area"</span>:<span class="token number">100</span><span class="token punctuation">,</span><span class="token string">"rooms"</span>:<span class="token number">3</span>}<span class="token punctuation">,</span> <span class="token operator">+</span>
  {<span class="token string">"level"</span>:<span class="token number">2</span><span class="token punctuation">,</span><span class="token string">"no"</span>:<span class="token number">5</span><span class="token punctuation">,</span><span class="token string">"area"</span>:<span class="token number">60</span><span class="token punctuation">,</span><span class="token string">"rooms"</span>:<span class="token number">2</span>}<span class="token punctuation">]</span>
 <span class="token punctuation">[</span>{<span class="token string">"level"</span>:<span class="token number">1</span><span class="token punctuation">,</span><span class="token string">"no"</span>:<span class="token number">1</span><span class="token punctuation">,</span><span class="token string">"area"</span>:<span class="token number">40</span><span class="token punctuation">,</span><span class="token string">"rooms"</span>:<span class="token number">1</span>}<span class="token punctuation">,</span>  <span class="token operator">+</span>
  {<span class="token string">"level"</span>:<span class="token number">1</span><span class="token punctuation">,</span><span class="token string">"no"</span>:<span class="token number">2</span><span class="token punctuation">,</span><span class="token string">"area"</span>:<span class="token number">80</span><span class="token punctuation">,</span><span class="token string">"rooms"</span>:<span class="token number">3</span>}<span class="token punctuation">,</span>  <span class="token operator">+</span>
  {<span class="token string">"level"</span>:<span class="token number">1</span><span class="token punctuation">,</span><span class="token string">"no"</span>:<span class="token number">3</span><span class="token punctuation">,</span><span class="token string">"area"</span>:<span class="token boolean">null</span><span class="token punctuation">,</span><span class="token string">"rooms"</span>:<span class="token number">2</span>}<span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- nested JSON_ARRAYAGG</span>
﻿<span class="token keyword">SELECT</span>
  JSON_ARRAYAGG<span class="token punctuation">(</span>floor<span class="token punctuation">)</span> <span class="token keyword">AS</span> floors
<span class="token keyword">FROM</span> <span class="token punctuation">(</span>
  <span class="token keyword">SELECT</span>
    JSON_OBJECT<span class="token punctuation">(</span>
      <span class="token string">'level'</span>: level<span class="token punctuation">,</span>
      <span class="token string">'apt'</span>: JSON_ARRAYAGG<span class="token punctuation">(</span>
        JSON_OBJECT<span class="token punctuation">(</span>
          <span class="token string">'no'</span>: <span class="token keyword">no</span><span class="token punctuation">,</span>
          <span class="token string">'area'</span>: area<span class="token punctuation">,</span>
          <span class="token string">'rooms'</span>: rooms
        <span class="token punctuation">)</span>
      <span class="token punctuation">)</span>
    <span class="token punctuation">)</span> <span class="token keyword">AS</span> floor
  <span class="token keyword">FROM</span> house_apt
  <span class="token keyword">GROUP</span> <span class="token keyword">BY</span> level
<span class="token punctuation">)</span> floor<span class="token punctuation">;</span>
                                                                                                                       floors
<span class="token comment">-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------</span>
 <span class="token punctuation">[</span>{<span class="token string">"level"</span> : <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"apt"</span> : <span class="token punctuation">[</span>{<span class="token string">"no"</span> : <span class="token number">4</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">100</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span> : <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">2</span>}<span class="token punctuation">]</span>}<span class="token punctuation">,</span> {<span class="token string">"level"</span> : <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"apt"</span> : <span class="token punctuation">[</span>{<span class="token string">"no"</span> : <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">40</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">1</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span> : <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span> : <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"area"</span> : <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span> : <span class="token number">2</span>}<span class="token punctuation">]</span>}<span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>

</code></pre>
<h3 id="json_exists---determines-whether-a-json-value-satisfies-a-search-criterion">JSON_EXISTS - determines whether a JSON value satisfies a search criterion</h3>
<p>JSON_EXISTS is a predicate, that can be used to  test whether an SQL/JSON path expression finds one or more  SQL/JSON items.</p>
<p>Syntax:</p>
<pre><code>JSON_EXISTS (
	json_api_common_syntax
	[ { TRUE | FALSE | UNKNOWN | ERROR } ON ERROR ]
)

Default is FALSE ON ERROR.
</code></pre>
<p>Examples:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token comment">-- analogous to jsonb '{"a": 1}' ? 'a'</span>
<span class="token keyword">SELECT</span> JSON_EXISTS<span class="token punctuation">(</span>jsonb <span class="token string">'{"a": 1}'</span><span class="token punctuation">,</span> <span class="token string">'strict $.a'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_exists
<span class="token comment">-------------</span>
 t
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span> JSON_EXISTS<span class="token punctuation">(</span>jsonb <span class="token string">'{"a": [1,2,3]}'</span><span class="token punctuation">,</span> <span class="token string">'strict $.a[*] ? (@ &gt; 2)'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_exists
<span class="token comment">-------------</span>
 t
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token comment">--  Using non-constant JSOP path</span>
<span class="token keyword">SELECT</span> JSON_EXISTS<span class="token punctuation">(</span>jsonb <span class="token string">'{"a": 123}'</span><span class="token punctuation">,</span> <span class="token string">'$'</span> <span class="token operator">||</span> <span class="token string">'.'</span> <span class="token operator">||</span> <span class="token string">'a'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_exists
<span class="token comment">-------------</span>
 t
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
<span class="token comment">-- Error behavior (FALSE by default)</span>
<span class="token keyword">SELECT</span> JSON_EXISTS<span class="token punctuation">(</span>jsonb <span class="token string">'{"a": [1,2,3]}'</span><span class="token punctuation">,</span> <span class="token string">'strict $.a[5]'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
ERROR: Invalid SQL<span class="token operator">/</span>JSON subscript
<span class="token keyword">SELECT</span> JSON_EXISTS<span class="token punctuation">(</span>jsonb <span class="token string">'{"a": [1,2,3]}'</span><span class="token punctuation">,</span> <span class="token string">'lax $.a[5]'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_exists
<span class="token comment">-------------</span>
 <span class="token number">f</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="json_value---extracts-a-scalar-json-value--and-returns-it-as-a-native-sql-type">JSON_VALUE - extracts a scalar JSON value  and returns it as a native SQL type</h3>
<p>Syntax:</p>
<pre><code>JSON_VALUE (
  json_api_common_syntax
  [ RETURNING data_type ]
  [ { ERROR | NULL | DEFAULT expression } ON EMPTY ]
  [ { ERROR | NULL | DEFAULT expression } ON ERROR ]
)
</code></pre>
<p>The optional <code>RETURNING</code> clause performs a typecast. Without a <code>RETURNING</code> clause, <code>JSON_VALUE</code> returns a string.</p>
<p>Examples:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token comment">-- Only singleton scalars are allowed here for returning:</span>
<span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'[1]'</span><span class="token punctuation">,</span> <span class="token string">'strict $'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
ERROR: SQL<span class="token operator">/</span>JSON scalar required
<span class="token keyword">SELECT</span> JSON_VALUE<span class="token punctuation">(</span>jsonb <span class="token string">'[1,2]'</span><span class="token punctuation">,</span> <span class="token string">'strict $[*]'</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span><span class="token punctuation">;</span>
ERROR: more than one SQL<span class="token operator">/</span>JSON item
</code></pre>
<h3 id="json_query---extracts-a-part--of-json-document-and-returns-it-as-a-json-string">JSON_QUERY - extracts a part  of JSON document and returns it as a JSON string</h3>
<p>JSON_QUERY uses SQL/JSON path expression to extract a part[s] of JSON document. Since it should return a JSON string,  multiple parts should be wrapped   into an array  (<code>WRAPPER</code>) or the function should raise an exception.</p>
<p>Syntax:</p>
<pre><code>JSON_QUERY (
	json_api_common_syntax
	[ RETURNING data_type [ FORMAT json_representation ] ]
	[ { WITHOUT | WITH { CONDITIONAL | [UNCONDITIONAL] } }
	[ ARRAY ] WRAPPER ]
	[ { KEEP | OMIT } QUOTES [ ON SCALAR STRING ] ]
	[ { ERROR | NULL | EMPTY { ARRAY | OBJECT } } ON EMPTY ]
	[ { ERROR | NULL | EMPTY { ARRAY | OBJECT } } ON ERROR ]
)
</code></pre>
<p>Default behavior of <code>WRAPPER</code> clause is not to wrap the matched  parts of JSON document  ( <code>WITHOUT WRAPPER</code> ) and to wrap unconditionally if wrap (<code>WITH UNCONDITIONAL WRAPPER</code>).</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
    js<span class="token punctuation">,</span>
    JSON_QUERY<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'lax $[*]'</span><span class="token punctuation">)</span> <span class="token keyword">AS</span> <span class="token string">"without"</span><span class="token punctuation">,</span>
    JSON_QUERY<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'lax $[*]'</span> <span class="token keyword">WITH</span> WRAPPER<span class="token punctuation">)</span>  <span class="token keyword">AS</span> <span class="token string">"with uncond"</span><span class="token punctuation">,</span>
    JSON_QUERY<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'lax $[*]'</span> <span class="token keyword">WITH</span> CONDITIONAL WRAPPER<span class="token punctuation">)</span> <span class="token keyword">AS</span> <span class="token string">"with cond"</span>
<span class="token keyword">FROM</span>
    <span class="token punctuation">(</span><span class="token keyword">VALUES</span> <span class="token punctuation">(</span>jsonb <span class="token string">'[]'</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token punctuation">(</span><span class="token string">'[1]'</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token punctuation">(</span><span class="token string">'[[1,2,3]]'</span><span class="token punctuation">)</span><span class="token punctuation">,</span>  <span class="token punctuation">(</span><span class="token string">'[{"a": 1}]'</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token punctuation">(</span><span class="token string">'[1, null, "2"]'</span><span class="token punctuation">)</span><span class="token punctuation">)</span> foo<span class="token punctuation">(</span>js<span class="token punctuation">)</span><span class="token punctuation">;</span>
       js       <span class="token operator">|</span>  without  <span class="token operator">|</span>  <span class="token keyword">with</span> uncond   <span class="token operator">|</span>   <span class="token keyword">with</span> cond
<span class="token comment">----------------+-----------+----------------+----------------</span>
 <span class="token punctuation">[</span><span class="token punctuation">]</span>             <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>    <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>         <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
 <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">]</span>            <span class="token operator">|</span> <span class="token number">1</span>         <span class="token operator">|</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">]</span>            <span class="token operator">|</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">]</span>
 <span class="token punctuation">[</span><span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span><span class="token punctuation">]</span>    <span class="token operator">|</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span> <span class="token operator">|</span> <span class="token punctuation">[</span><span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span><span class="token punctuation">]</span>    <span class="token operator">|</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token number">3</span><span class="token punctuation">]</span>
 <span class="token punctuation">[</span>{<span class="token string">"a"</span>: <span class="token number">1</span>}<span class="token punctuation">]</span>     <span class="token operator">|</span> {<span class="token string">"a"</span>: <span class="token number">1</span>}  <span class="token operator">|</span> <span class="token punctuation">[</span>{<span class="token string">"a"</span>: <span class="token number">1</span>}<span class="token punctuation">]</span>     <span class="token operator">|</span> {<span class="token string">"a"</span>: <span class="token number">1</span>}
 <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"2"</span><span class="token punctuation">]</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>    <span class="token operator">|</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"2"</span><span class="token punctuation">]</span> <span class="token operator">|</span> <span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">,</span> <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"2"</span><span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>Quotes behavior (KEEP by default):</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span> JSON_QUERY<span class="token punctuation">(</span>jsonb <span class="token string">'"aaa"'</span><span class="token punctuation">,</span> <span class="token string">'strict $'</span> RETURNING <span class="token keyword">text</span> OMIT QUOTES<span class="token punctuation">)</span><span class="token punctuation">;</span>
 json_query
<span class="token comment">------------</span>
 <span class="token number">aaa</span>
<span class="token punctuation">(</span><span class="token number">1</span> <span class="token keyword">row</span><span class="token punctuation">)</span>
</code></pre>
<h3 id="json_table---query-a-json-text-and-present-the-results-as-a-relational-table">JSON_TABLE - query a JSON text and present the results as a relational table</h3>
<p>JSON_TABLE creates a relational view of JSON data.  It can be used only in FROM clause.  The co</p>
<p>JSON_TABLE  has three parameters:</p>
<ol>
<li>The JSON value on which to operate.</li>
<li>An SQL/JSON path expression to specify zero or more rows.</li>
<li>A COLUMNS clause to specify the shape of the output table. The COLUMNS can be nested.</li>
</ol>
<p>JSON_TABLE internally uses XML_TABLE infrastructure.</p>
<p>Syntax:</p>
<pre><code>JSON_TABLE (
json_api_common_syntax
COLUMNS ( json_table_column [, ...] )
[ PLAN ( json_table_plan ) |
  PLAN DEFAULT ( json_table_default_plan_choices ) ]
)

json_table_column ::=
    column_name FOR ORDINALITY

    | column_name data_type
		[ PATH json_path_specification ]
		/* behavior are the same as in JSON_VALUE */
		[ empty_behavior ON EMPTY ] 
		[ error_behavior ON ERROR ]

| column_name data_type FORMAT json_representation
		[ PATH json_path_specification ]
		/* behavior are the same as in JSON_QUERY */
	    [ wrapper_behavior WRAPPER ]
		[ quotes_behavior QUOTES [ ON SCALAR STRING ] ]
		[ empty_behavior ON EMPTY ]
		[ error_behavior ON ERROR ]
		
| NESTED PATH json_path_specification [ AS json_path_name ]
		COLUMNS ( json_table_column [, ...] )
json_table_default_plan_choices ::=
      { INNER | OUTER } [ , { CROSS | UNION } ]
	| { CROSS | UNION } [ , { INNER | OUTER } ]

json_table_plan ::=
	json_table_path_name
	| json_table_plan_parent/child
	| json_table_plan_sibling

json_table_plan_parent/child ::=
	  json_table_path_name OUTER json_table_plan_primary
	| json_table_path_name INNER json_table_plan_primary

json_table_plan_sibling ::=
	  json_table_plan_primary { UNION json_table_plan_primary }[...]
	| json_table_plan_primary { CROSS json_table_plan_primary }[...]

json_table_plan_primary ::=
		json_table_path_name | ( json_table_plan )
</code></pre>
<p>JSON_TABLE uses path expression (in <code>json_api_common_syntax</code>) to produce an SQL/JSON sequence, with one SQL/JSON item for each row of the output table.</p>
<p>The COLUMNS clause can define several kinds of columns: ordinality columns, regular columns, formatted columns and nested columns .</p>
<p>In the following example, the path expression <code>$.floor[*]</code> specifies  rows to output. Each row has column  <code>level</code> , which obtained by <strong>implicit</strong> path  <code>$.lvelel</code>, where <code>$</code> is  relative to parent path (absolute path is <code>$.floor[*].level</code>), column <code>num_apt</code>  and column <code>apts</code> explicitly defined by corresponding path expressions.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token comment">-- basic example: regular and formatted columns, paths</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>     <span class="token comment">-- PATH 'lax $.level' is used here by default</span>
    num_apt <span class="token keyword">int</span> PATH <span class="token string">'$.apt.size()'</span><span class="token punctuation">,</span> 
    apts jsonb FORMAT JSON PATH <span class="token string">'$.apt'</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span> num_apt <span class="token operator">|</span>                                                    apts                                                     
<span class="token comment">-------+---------+-------------------------------------------------------------------------------------------------------------</span>
     <span class="token number">1</span> <span class="token operator">|</span>       <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">[</span>{<span class="token string">"no"</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">40</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">1</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}<span class="token punctuation">]</span>
     <span class="token number">2</span> <span class="token operator">|</span>       <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">[</span>{<span class="token string">"no"</span>: <span class="token number">4</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">100</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}<span class="token punctuation">]</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>An ordinality column provides a sequential numbering of rows. Row numbering is 1-based.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*] ? (@.rooms &gt; 1)'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    id <span class="token keyword">FOR</span> ORDINALITY<span class="token punctuation">,</span>
    <span class="token keyword">no</span> <span class="token keyword">int</span><span class="token punctuation">,</span>
    rooms <span class="token keyword">int</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 id <span class="token operator">|</span> <span class="token keyword">no</span> <span class="token operator">|</span> rooms 
<span class="token comment">----+----+-------</span>
  <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">2</span> <span class="token operator">|</span>     <span class="token number">3</span>
  <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">3</span> <span class="token operator">|</span>     <span class="token number">2</span>
  <span class="token number">3</span> <span class="token operator">|</span>  <span class="token number">4</span> <span class="token operator">|</span>     <span class="token number">3</span>
  <span class="token number">4</span> <span class="token operator">|</span>  <span class="token number">5</span> <span class="token operator">|</span>     <span class="token number">2</span>
<span class="token punctuation">(</span><span class="token number">4</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>A regular column supports columns of scalar type. The column is produced using the semantics of JSON_VALUE, so it is possible to specify all  behavior clauses of JSON_VALUE, for example,  an optional ON EMPTY and ON ERROR clauses.</p>
<p>The column has an <strong>optional</strong> path expression, called the column pattern, which can be defaulted from the column name. The column pattern is used to search for the column within the current SQL/JSON item produced by the row pattern.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    <span class="token keyword">no</span> <span class="token keyword">int</span><span class="token punctuation">,</span>
    area <span class="token keyword">float</span> PATH <span class="token string">'$.area / 100'</span><span class="token punctuation">,</span>
    area_type <span class="token keyword">text</span> PATH <span class="token string">'$.area.type()'</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 <span class="token keyword">no</span> <span class="token operator">|</span> area <span class="token operator">|</span> area_type 
<span class="token comment">----+------+-----------</span>
  <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">0.4</span> <span class="token operator">|</span> number
  <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">0.8</span> <span class="token operator">|</span> number
  <span class="token number">3</span> <span class="token operator">|</span>      <span class="token operator">|</span> <span class="token boolean">null</span>
  <span class="token number">4</span> <span class="token operator">|</span>    <span class="token number">1</span> <span class="token operator">|</span> number
  <span class="token number">5</span> <span class="token operator">|</span>  <span class="token number">0.6</span> <span class="token operator">|</span> number
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- regular datetime columns</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.info.dates[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    str <span class="token keyword">text</span> PATH <span class="token string">'$'</span><span class="token punctuation">,</span>
    dt <span class="token keyword">date</span> PATH <span class="token string">'$.datetime("DD-MM-YYYY")'</span><span class="token punctuation">,</span>
    ts <span class="token keyword">timestamp</span> PATH <span class="token string">'$.datetime("DD-MM-YYYY HH24:MI:SS")'</span><span class="token punctuation">,</span>
    tstz timestamptz PATH <span class="token string">'$.datetime("DD-MM-YYYY HH24:MI:SS TZH:TZM", "+03")'</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
           str           <span class="token operator">|</span>     dt     <span class="token operator">|</span>         ts          <span class="token operator">|</span>          tstz          
<span class="token comment">-------------------------+------------+---------------------+------------------------</span>
 <span class="token number">01</span><span class="token operator">-</span><span class="token number">02</span><span class="token operator">-</span><span class="token number">2015</span>              <span class="token operator">|</span> <span class="token number">2015</span><span class="token operator">-</span><span class="token number">02</span><span class="token operator">-</span><span class="token number">01</span> <span class="token operator">|</span> <span class="token number">2015</span><span class="token operator">-</span><span class="token number">02</span><span class="token operator">-</span><span class="token number">01</span> <span class="token number">00</span>:<span class="token number">00</span>:<span class="token number">00</span> <span class="token operator">|</span> <span class="token number">2015</span><span class="token operator">-</span><span class="token number">02</span><span class="token operator">-</span><span class="token number">01</span> <span class="token number">00</span>:<span class="token number">00</span>:<span class="token number">00</span><span class="token operator">+</span><span class="token number">03</span>
 <span class="token number">04</span><span class="token operator">-</span><span class="token number">10</span><span class="token operator">-</span><span class="token number">1957</span> <span class="token number">19</span>:<span class="token number">28</span>:<span class="token number">34</span> <span class="token operator">+</span><span class="token number">00</span> <span class="token operator">|</span> <span class="token number">1957</span><span class="token operator">-</span><span class="token number">10</span><span class="token operator">-</span><span class="token number">04</span> <span class="token operator">|</span> <span class="token number">1957</span><span class="token operator">-</span><span class="token number">10</span><span class="token operator">-</span><span class="token number">04</span> <span class="token number">19</span>:<span class="token number">28</span>:<span class="token number">34</span> <span class="token operator">|</span> <span class="token number">1957</span><span class="token operator">-</span><span class="token number">10</span><span class="token operator">-</span><span class="token number">04</span> <span class="token number">22</span>:<span class="token number">28</span>:<span class="token number">34</span><span class="token operator">+</span><span class="token number">03</span>
 <span class="token number">12</span><span class="token operator">-</span><span class="token number">04</span><span class="token operator">-</span><span class="token number">1961</span> <span class="token number">09</span>:<span class="token number">07</span>:<span class="token number">00</span> <span class="token operator">+</span><span class="token number">03</span> <span class="token operator">|</span> <span class="token number">1961</span><span class="token operator">-</span><span class="token number">04</span><span class="token operator">-</span><span class="token number">12</span> <span class="token operator">|</span> <span class="token number">1961</span><span class="token operator">-</span><span class="token number">04</span><span class="token operator">-</span><span class="token number">12</span> <span class="token number">09</span>:<span class="token number">07</span>:<span class="token number">00</span> <span class="token operator">|</span> <span class="token number">1961</span><span class="token operator">-</span><span class="token number">04</span><span class="token operator">-</span><span class="token number">12</span> <span class="token number">09</span>:<span class="token number">07</span>:<span class="token number">00</span><span class="token operator">+</span><span class="token number">03</span>
<span class="token punctuation">(</span><span class="token number">3</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- regular columns: DEFAULT ON EMPTY behavior</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    area float4 PATH <span class="token string">'$.area ? (@ != null)'</span> <span class="token keyword">DEFAULT</span> <span class="token number">0</span> <span class="token keyword">ON</span> EMPTY    
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 area 
<span class="token comment">------</span>
   <span class="token number">40</span>
   <span class="token number">80</span>
    <span class="token number">0</span>
  <span class="token number">100</span>
   <span class="token number">60</span>
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- regular columns: ERROR ON EMPTY behavior</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    area float4 PATH <span class="token string">'$.area ? (@ != null)'</span> ERROR <span class="token keyword">ON</span> EMPTY ERROR <span class="token keyword">ON</span> ERROR
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
ERROR:  <span class="token keyword">no SQL</span><span class="token operator">/</span>JSON item

<span class="token comment">-- regular columns: DEFAULT ON ERROR behavior</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*].apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    area <span class="token keyword">text</span> PATH <span class="token string">'$.area * 100'</span> <span class="token keyword">DEFAULT</span> <span class="token string">'Unknown'</span> <span class="token keyword">ON</span> ERROR
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
  area   
<span class="token comment">---------</span>
 <span class="token number">4000</span>
 <span class="token number">8000</span>
 Unknown
 <span class="token number">10000</span>
 <span class="token number">6000</span>
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- JSON_TABLE ON ERROR behavior: EMPTY ON ERROR is by default</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'strict $.foo[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    bar <span class="token keyword">int</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 bar 
<span class="token comment">-----</span>
<span class="token punctuation">(</span><span class="token number">0</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- JSON_TABLE ON ERROR behavior: ERROR ON ERRROR</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'strict $.foo[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    bar <span class="token keyword">int</span>
  <span class="token punctuation">)</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
ERROR:  SQL<span class="token operator">/</span>JSON member <span class="token operator">not</span> found
  
<span class="token comment">-- JSON_TABLE ON ERROR behavior overrides default ON ERROR behavior of its columns</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    bar <span class="token keyword">int</span> PATH <span class="token string">'strict $.bar'</span> <span class="token comment">-- NULL ON ERROR is by default here</span>
  <span class="token punctuation">)</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
ERROR:  SQL<span class="token operator">/</span>JSON member <span class="token operator">not</span> found
</code></pre>
<p>Formatted columns are used for returning of composite SQL/JSON items, they internally transformed into JSON_QUERY,  so it is possible to specify all  behavior clauses of JSON_QUERY, for example,  an optional ON EMPTY and ON ERROR clauses.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    floor jsonb FORMAT JSON PATH <span class="token string">'$'</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
                                                              floor                                                               
<span class="token comment">----------------------------------------------------------------------------------------------------------------------------------</span>
 {<span class="token string">"apt"</span>: <span class="token punctuation">[</span>{<span class="token string">"no"</span>: <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">40</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">1</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">80</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token boolean">null</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}<span class="token punctuation">]</span><span class="token punctuation">,</span> <span class="token string">"level"</span>: <span class="token number">1</span>}
 {<span class="token string">"apt"</span>: <span class="token punctuation">[</span>{<span class="token string">"no"</span>: <span class="token number">4</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">100</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">3</span>}<span class="token punctuation">,</span> {<span class="token string">"no"</span>: <span class="token number">5</span><span class="token punctuation">,</span> <span class="token string">"area"</span>: <span class="token number">60</span><span class="token punctuation">,</span> <span class="token string">"rooms"</span>: <span class="token number">2</span>}<span class="token punctuation">]</span><span class="token punctuation">,</span> <span class="token string">"level"</span>: <span class="token number">2</span>}
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- returning of composite JSON items without FORMAT JSON =&gt; error (NULL ON ERROR by default)</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    floor jsonb PATH <span class="token string">'$'</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 floor
<span class="token comment">--------</span>
 <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
 <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- returning of composite JSON items without FORMAT JSON =&gt; error (ERROR ON ERROR)</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    floor jsonb PATH <span class="token string">'$'</span>
  <span class="token punctuation">)</span> ERROR <span class="token keyword">ON</span> ERROR<span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
ERROR:  SQL<span class="token operator">/</span>JSON scalar required
</code></pre>
<p>The nested COLUMNS clause begins with the keyword NESTED, followed by a path and an optional path name. The path provides a refined context for the nested columns. The primary use of the path name is if the user wishes to specify an explicit plan. After the prolog to specify the path and path name, there is a COLUMNS clause, which has the same capabilities already considered. The NESTED clause allows unnesting of (even deeply) nested JSON objects/arrays in one invocation rather than chaining several JSON_TABLE expressions in the SQL-statement.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token comment">-- nested columns (1-level)</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      <span class="token keyword">no</span> <span class="token keyword">int</span><span class="token punctuation">,</span>
      area <span class="token keyword">float</span><span class="token punctuation">,</span>
      rooms <span class="token keyword">int</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>

 level <span class="token operator">|</span> <span class="token keyword">no</span> <span class="token operator">|</span>  area  <span class="token operator">|</span> rooms 
<span class="token comment">-------+----+--------+-------</span>
     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">1</span> <span class="token operator">|</span>     <span class="token number">40</span> <span class="token operator">|</span>     <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">2</span> <span class="token operator">|</span>     <span class="token number">80</span> <span class="token operator">|</span>     <span class="token number">3</span>
     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>     <span class="token number">2</span>
     <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">4</span> <span class="token operator">|</span>    <span class="token number">100</span> <span class="token operator">|</span>     <span class="token number">3</span>
     <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">5</span> <span class="token operator">|</span>     <span class="token number">60</span> <span class="token operator">|</span>     <span class="token number">2</span>
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>Every path may be followed by a path name using an AS clause. Path names are identifiers and must be unique. Path names are used in the PLAN clause to express the desired output plan. PLAN clause could be INNER, OUTER, UNION and CROSS, which correspond to INNER JOIN, LEFT OUTER JOIN, FULL OUTER JOIN and CROSS JOIN respectively.  If there is an explicit PLAN clause, all path names must be explicit and appear in the PLAN clause exactly once.</p>
<p>INNER and OUTER  (default) are used for parent/child relationship and it is mandatory to specify the first operand (path name in AS clause) and it must be an ancestor of all path names in the second operand.</p>
<p>UNION expresses semantics of a FULL OUTER JOIN and is default with sibling relationship.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span>  <span class="token keyword">AS</span> lvl <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.area &gt; 1000)'</span> <span class="token keyword">AS</span> big <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      <span class="token keyword">no</span> <span class="token keyword">int</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token punctuation">(</span>lvl <span class="token keyword">OUTER</span> big<span class="token punctuation">)</span> <span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span>   <span class="token keyword">no</span>
<span class="token comment">-------+--------</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- Default plan clause is  OUTER for parent/child relationship</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.area &gt; 1000)'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token comment">-- returns no items</span>
      <span class="token keyword">no</span> <span class="token keyword">int</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span>   <span class="token keyword">no</span>
<span class="token comment">-------+--------</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
<span class="token punctuation">(</span><span class="token number">2</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>Example of nested columns (2-level).</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    city <span class="token keyword">text</span> PATH <span class="token string">'$.address.city'</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      level <span class="token keyword">int</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
        <span class="token keyword">no</span> <span class="token keyword">int</span><span class="token punctuation">,</span>
        area <span class="token keyword">float</span><span class="token punctuation">,</span>
        rooms <span class="token keyword">int</span>
      <span class="token punctuation">)</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
  city  <span class="token operator">|</span> level <span class="token operator">|</span> <span class="token keyword">no</span> <span class="token operator">|</span>  area  <span class="token operator">|</span> rooms
<span class="token comment">--------+-------+----+--------+-------</span>
 Moscow <span class="token operator">|</span>     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">1</span> <span class="token operator">|</span>     <span class="token number">40</span> <span class="token operator">|</span>     <span class="token number">1</span>
 Moscow <span class="token operator">|</span>     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">2</span> <span class="token operator">|</span>     <span class="token number">80</span> <span class="token operator">|</span>     <span class="token number">3</span>
 Moscow <span class="token operator">|</span>     <span class="token number">1</span> <span class="token operator">|</span>  <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>     <span class="token number">2</span>
 Moscow <span class="token operator">|</span>     <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">4</span> <span class="token operator">|</span>    <span class="token number">100</span> <span class="token operator">|</span>     <span class="token number">3</span>
 Moscow <span class="token operator">|</span>     <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">5</span> <span class="token operator">|</span>     <span class="token number">60</span> <span class="token operator">|</span>     <span class="token number">2</span>
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>Two nested paths on the same level are union-joined by default.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      no1 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span>
    <span class="token punctuation">)</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      no2 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span>  no1   <span class="token operator">|</span>  no2
<span class="token comment">-------+--------+--------</span>
     <span class="token number">1</span> <span class="token operator">|</span>      <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">1</span> <span class="token operator">|</span>      <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">1</span> <span class="token operator">|</span>      <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">3</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">5</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">10</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>PLAN DEFAULT (INNER) is equivalent to explicit PLAN clause.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.area &gt; 1000)'</span> <span class="token keyword">AS</span> apt <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token comment">-- returns 0 items</span>
      <span class="token keyword">no</span> <span class="token keyword">int</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">INNER</span><span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span> <span class="token keyword">no</span>
<span class="token comment">-------+----</span>
<span class="token punctuation">(</span><span class="token number">0</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.area &gt; 1000)'</span> <span class="token keyword">AS</span> apt <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token comment">-- returns 0 items</span>
      <span class="token keyword">no</span> <span class="token keyword">int</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token punctuation">(</span>floor <span class="token keyword">INNER</span> apt<span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span> <span class="token keyword">no</span>
<span class="token comment">-------+----</span>
<span class="token punctuation">(</span><span class="token number">0</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>PLAN DEFAULT (CROSS) is equivalent to explicit PLAN clause for sibling columns.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">AS</span> apt1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      no1 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span>
    <span class="token punctuation">)</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">AS</span> apt2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      no2 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">CROSS</span><span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span> no1 <span class="token operator">|</span> no2
<span class="token comment">-------+-----+-----</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">3</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span> <span class="token operator">|</span>   <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span> <span class="token operator">|</span>   <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span> <span class="token operator">|</span>   <span class="token number">3</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">4</span> <span class="token operator">|</span>   <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">4</span> <span class="token operator">|</span>   <span class="token number">5</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">5</span> <span class="token operator">|</span>   <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">5</span> <span class="token operator">|</span>   <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">13</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">AS</span> apt1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      no1 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span>
    <span class="token punctuation">)</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span> <span class="token keyword">AS</span> apt2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      no2 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span>
    <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token punctuation">(</span>floor <span class="token keyword">OUTER</span> <span class="token punctuation">(</span>apt1 <span class="token keyword">CROSS</span> apt2<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span> no1 <span class="token operator">|</span> no2
<span class="token comment">-------+-----+-----</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">3</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span> <span class="token operator">|</span>   <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span> <span class="token operator">|</span>   <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span>   <span class="token number">3</span> <span class="token operator">|</span>   <span class="token number">3</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">4</span> <span class="token operator">|</span>   <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">4</span> <span class="token operator">|</span>   <span class="token number">5</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">5</span> <span class="token operator">|</span>   <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">5</span> <span class="token operator">|</span>   <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">13</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<p>Combination of OUTER/INNER and UNION/CROSS joins:</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token comment">-- OUTER, UNION is the same as INNER, UNION</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.no &gt; 3)'</span> <span class="token keyword">AS</span> apt1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no1 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span>              <span class="token keyword">AS</span> apt2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no2 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">OUTER</span><span class="token punctuation">,</span> <span class="token keyword">UNION</span><span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span>  no1   <span class="token operator">|</span>  no2
<span class="token comment">-------+--------+--------</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">3</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">5</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">7</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- INNER, UNION</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.no &gt; 3)'</span> <span class="token keyword">AS</span> apt1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no1 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span>              <span class="token keyword">AS</span> apt2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no2 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">INNER</span><span class="token punctuation">,</span> <span class="token keyword">UNION</span><span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span>  no1   <span class="token operator">|</span>  no2
<span class="token comment">-------+--------+--------</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">1</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">2</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">3</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">5</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">7</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- OUTER, CROSS</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.no &gt; 3)'</span> <span class="token keyword">AS</span> apt1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no1 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span>              <span class="token keyword">AS</span> apt2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no2 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">OUTER</span><span class="token punctuation">,</span> <span class="token keyword">CROSS</span><span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span>  no1   <span class="token operator">|</span>  no2
<span class="token comment">-------+--------+--------</span>
     <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span>      <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span>      <span class="token number">5</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">5</span> <span class="token operator">|</span>      <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">5</span> <span class="token operator">|</span>      <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">5</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- INNER, CROSS</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  house<span class="token punctuation">,</span>
  JSON_TABLE<span class="token punctuation">(</span>js<span class="token punctuation">,</span> <span class="token string">'$.floor[*]'</span> <span class="token keyword">AS</span> floor <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
    level <span class="token keyword">int</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*] ? (@.no &gt; 3)'</span> <span class="token keyword">AS</span> apt1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no1 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
    NESTED PATH <span class="token string">'$.apt[*]'</span>              <span class="token keyword">AS</span> apt2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> no2 <span class="token keyword">int</span> PATH <span class="token string">'$.no'</span> <span class="token punctuation">)</span>
  <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">INNER</span><span class="token punctuation">,</span> <span class="token keyword">CROSS</span><span class="token punctuation">)</span><span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 level <span class="token operator">|</span> no1 <span class="token operator">|</span> no2
<span class="token comment">-------+-----+-----</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">4</span> <span class="token operator">|</span>   <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">4</span> <span class="token operator">|</span>   <span class="token number">5</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">5</span> <span class="token operator">|</span>   <span class="token number">4</span>
     <span class="token number">2</span> <span class="token operator">|</span>   <span class="token number">5</span> <span class="token operator">|</span>   <span class="token number">5</span>
<span class="token punctuation">(</span><span class="token number">4</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- OUTER, UNION</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  JSON_TABLE<span class="token punctuation">(</span><span class="token string">'[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]'</span><span class="token punctuation">,</span> <span class="token string">'$[*]'</span> <span class="token keyword">AS</span> root
    <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      n <span class="token keyword">int</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*] ? (@ &gt; 2)'</span> <span class="token keyword">AS</span> ap1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a1</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*]'</span>           <span class="token keyword">AS</span> ap2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a2</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span>
    <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">OUTER</span><span class="token punctuation">,</span> <span class="token keyword">UNION</span><span class="token punctuation">)</span>
  <span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 n <span class="token operator">|</span>   <span class="token number">a1</span>   <span class="token operator">|</span>   <span class="token number">a2</span>
<span class="token comment">---+--------+--------</span>
 <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">1</span>
 <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">2</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
 <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">3</span>
 <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">4</span>
 <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
<span class="token punctuation">(</span><span class="token number">7</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- INNER, UNION</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  JSON_TABLE<span class="token punctuation">(</span><span class="token string">'[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]'</span><span class="token punctuation">,</span> <span class="token string">'$[*]'</span> <span class="token keyword">AS</span> root
    <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      n <span class="token keyword">int</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*] ? (@ &gt; 2)'</span> <span class="token keyword">AS</span> ap1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a1</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*]'</span>           <span class="token keyword">AS</span> ap2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a2</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span>
    <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">INNER</span><span class="token punctuation">,</span> <span class="token keyword">UNION</span><span class="token punctuation">)</span>
  <span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 n <span class="token operator">|</span>   <span class="token number">a1</span>   <span class="token operator">|</span>   <span class="token number">a2</span>
<span class="token comment">---+--------+--------</span>
 <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">1</span>
 <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">2</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
 <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">3</span>
 <span class="token number">2</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span>      <span class="token number">4</span>
<span class="token punctuation">(</span><span class="token number">6</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">--OUTER, CROSS</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  JSON_TABLE<span class="token punctuation">(</span><span class="token string">'[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]'</span><span class="token punctuation">,</span> <span class="token string">'$[*]'</span> <span class="token keyword">AS</span> root
    <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      n <span class="token keyword">int</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*] ? (@ &gt; 2)'</span> <span class="token keyword">AS</span> ap1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a1</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*]'</span>           <span class="token keyword">AS</span> ap2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a2</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span>
    <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">OUTER</span><span class="token punctuation">,</span> <span class="token keyword">CROSS</span><span class="token punctuation">)</span>
  <span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 n <span class="token operator">|</span>   <span class="token number">a1</span>   <span class="token operator">|</span>   <span class="token number">a2</span>
<span class="token comment">---+--------+--------</span>
 <span class="token number">1</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">3</span> <span class="token operator">|</span>      <span class="token number">3</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">3</span> <span class="token operator">|</span>      <span class="token number">4</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span>      <span class="token number">3</span>
 <span class="token number">2</span> <span class="token operator">|</span>      <span class="token number">4</span> <span class="token operator">|</span>      <span class="token number">4</span>
 <span class="token number">3</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span> <span class="token operator">|</span> <span class="token punctuation">(</span><span class="token boolean">null</span><span class="token punctuation">)</span>
<span class="token punctuation">(</span><span class="token number">6</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>

<span class="token comment">-- INNER, CROSS</span>
<span class="token keyword">SELECT</span>
  jt<span class="token punctuation">.</span><span class="token operator">*</span>
<span class="token keyword">FROM</span>
  JSON_TABLE<span class="token punctuation">(</span><span class="token string">'[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]'</span><span class="token punctuation">,</span> <span class="token string">'$[*]'</span> <span class="token keyword">AS</span> root
    <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span>
      n <span class="token keyword">int</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*] ? (@ &gt; 2)'</span> <span class="token keyword">AS</span> ap1 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a1</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span><span class="token punctuation">,</span>
      NESTED PATH <span class="token string">'$.a[*]'</span>           <span class="token keyword">AS</span> ap2 <span class="token keyword">COLUMNS</span> <span class="token punctuation">(</span> <span class="token number">a2</span> <span class="token keyword">int</span> PATH <span class="token string">'$'</span> <span class="token punctuation">)</span>
    <span class="token punctuation">)</span> <span class="token keyword">PLAN</span> <span class="token keyword">DEFAULT</span> <span class="token punctuation">(</span><span class="token keyword">INNER</span><span class="token punctuation">,</span> <span class="token keyword">CROSS</span><span class="token punctuation">)</span>
  <span class="token punctuation">)</span> jt<span class="token punctuation">;</span>
 n <span class="token operator">|</span> <span class="token number">a1</span> <span class="token operator">|</span> <span class="token number">a2</span>
<span class="token comment">---+----+----</span>
 <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">3</span> <span class="token operator">|</span>  <span class="token number">3</span>
 <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">3</span> <span class="token operator">|</span>  <span class="token number">4</span>
 <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">4</span> <span class="token operator">|</span>  <span class="token number">3</span>
 <span class="token number">2</span> <span class="token operator">|</span>  <span class="token number">4</span> <span class="token operator">|</span>  <span class="token number">4</span>
<span class="token punctuation">(</span><span class="token number">4</span> <span class="token keyword">rows</span><span class="token punctuation">)</span>
</code></pre>
<h2 id="sqljson-conformance">SQL/JSON conformance</h2>
<ul>
<li><code>like_regex</code> supports posix regular expressions,  while the  standard requires xquery regexps.</li>
<li>Not supported (due to unresolved conflicts in SQL grammar):
<ul>
<li>expr FORMAT JSON IS [NOT] JSON</li>
<li>JSON_OBJECT(KEY key VALUE value, …)</li>
<li>JSON_ARRAY(SELECT … FORMAT JSON …)</li>
<li>JSON_ARRAY(SELECT … (ABSENT|NULL) ON NULL …)</li>
</ul>
</li>
<li>json_path_specification extended  to be an expression of jsonpath type. The standard requires  it <code>character_string_literal</code>.</li>
<li>Only error codes are returned for the failed arithmetic operations inside jsonpath, error messages are lost</li>
<li>Use boolean  expression on the path, PostgreSQL extension</li>
<li>Default timezone added to <code>datetime()</code> as second argument. That helped to keep jsonpath operators and functions to be immutable.</li>
<li><code>.**</code>  - recursive wildcard member accessor, PostgreSQL extension</li>
<li>json[b] op jsonpath - PostgreSQL extension</li>
<li>[path] - wrap SQL/JSON sequences into an array - PostgreSQL extension</li>
</ul>
<h2 id="links">Links</h2>
<ul>
<li>Github Postgres Professional repository<br>
<a href="https://github.com/postgrespro/sqljson">https://github.com/postgrespro/sqljson</a></li>
<li>WEB-interface to play with SQL/JSON<br>
<a href="http://sqlfiddle.postgrespro.ru/#!21/0/2580">http://sqlfiddle.postgrespro.ru/#!21/0/2580</a></li>
<li>Try SQL/JSON on your android phone<br>
<a href="https://obartunov.livejournal.com/199005.html">https://obartunov.livejournal.com/199005.html</a></li>
<li>Technical Report (SQL/JSON) - available for free<br>
<a href="http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip">http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip</a></li>
<li>Jsonb roadmap - talk at <a href="http://PGConf.eu">PGConf.eu</a>, 2018<br>
<a href="http://www.sai.msu.su/~megera/postgres/talks/sqljson-pgconf.eu-2017.pdf">http://www.sai.msu.su/~megera/postgres/talks/sqljson-pgconf.eu-2017.pdf</a></li>
<li>Play with SQL/JSON on Android phone<br>
<a href="https://obartunov.livejournal.com/199005.html">https://obartunov.livejournal.com/199005.html</a></li>
</ul>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>
<!--stackedit_data:&#10;eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19&#10;-->
<!--stackedit_data:&#10;eyJoaXN0b3J5IjpbNTgwMjQzOTRdfQ==&#10;-->

