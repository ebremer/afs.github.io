---
layout: doc
title: Substitution of Variables in SPARQL
---
<div class="docinfo">
     <p>Andy Seaborne</p>
     <p>October 2019</p>
</div>

The SPARQL 1.1 algebra operation
"[substitute](https://www.w3.org/TR/sparql11-query/#defn_substitute)" evaluates
a graph pattern where there is a specific are variables given by a solution
mapping. The operation is used to in the evaluation of <tt>EXISTS</tt> and
<tt>NOT EXISTS</tt> operations.


This document list problems that have been identified with the
<tt>substitute</tt> operation and proposes an improved substitution evaluation
process that addresses these problems based on the concept of 
[Correlated Subquery](https://en.wikipedia.org/wiki/Correlated_subquery) found in SQL.

* [Summary](#summary)
* [Identified Issues](#issues)
* [An Improved "substitute" Operation](#substitute-ng)
* [Addressing Issues](#addressing-issues)
* [Notes](#notes)

## Summary {#summary}

The fundamental problem is that a variable can not be simply replaced by a value
(an RDF Term) in all places in a graph pattern. There are some places where
SPARQL function forms and "AS" assignment require a variable. The proposal here
is to take the variable binding from the input solution, retaining the
variable in the graph pattern, and disallowing cases that can reset the variable
binding as is already the case elsewhere in SPARQL.

## Identified Issues {#issues}

This section describes the issues identified on the 
SPARQL Exists Community group mailing list 
[public-sparql-exists/2016Jul/0014](https://lists.w3.org/Archives/Public/public-sparql-exists/2016Jul/0014.html).

* [Issue-1](#issue-1): Some uses of EXISTS are not defined during evaluation.
* [Issue-2](#issue-2): Substitution happens where definitions are only for variables.
* [Issue-3](#issue-3): Blank nodes substituted into BGPs act as variables.
* [Issue-4](#issue-4): Substitution can flip MINUS to its disjoint-domain case.
* [Issue-5](#issue-5): Substitution affects disconnected variables.

### Issue 1: Some uses of EXISTS are not defined during evaluation {#issue-1}

The evaluation process in the specificiation is defined for graph patterns but
there are situations where the evaluation is of an alegbra form not listed.

For example:

    FILTER EXISTS { SELECT ?y { ?y :q :c . } }

and

    FILTER EXISTS { VALUES ?y { 123 } }

The argument to
<tt>[exists](https://www.w3.org/TR/sparql11-query/#defn_evalExists)</tt>
is not explicitly listed as a "Graph Pattern" in the table of SPARQL algebra symbols in
[section 18.2](https://www.w3.org/TR/sparql11-query/#sparqlQuery)
when the argument to <tt>EXISTS</tt> is a 
[GroupGraphPattern](https://www.w3.org/TR/sparql11-query/#rGroupGraphPattern)
containing just a 
[subquery](https://www.w3.org/TR/sparql11-query/#subqueries)
or just 
[InlineData](https://www.w3.org/TR/sparql11-query/#inline-data).

### Issue 2: Substitution happens where definitions are only for variables {#issue-2}

There are places in the SPARQL syntax and algebra where
variables are allowed but not RDF terms (constant values).<p>

Example:

    FILTER EXISTS { BIND ( :e AS ?z ) 
                    { SELECT ?x { :b :p :c } }
                  }

Both positions "AS ?z" and "SELECT ?x" must be variables.

In the algebra, this affects

* <tt>[extend]("https://www.w3.org/TR/sparql11-query/#defn_extend")</tt>
  (related to the use of <tt>AS</tt> in SPARQL syntax)
* [in line data](https://www.w3.org/TR/sparql11-query/#inline-data) 
  (related to the use of <tt>VALUES</tt>)
* <tt>[BOUND](https://www.w3.org/TR/sparql11-query/#func-bound)</tt>

### Issue 3: Blank nodes substituted into BGPs act as variables {#issue-3}

In the 
[evaluation of basic graph patterns](https://www.w3.org/TR/sparql11-query/#BasicGraphPattern)
(BGPs) blank nodes 
[are replaced](https://www.w3.org/TR/sparql11-query/#BGPsparql)
by RDF terms from the graph being matched and variables are replaced by a solution
mapping from query variables to RDF terms so that the  basic graph pattern is
now a subgraph of the graph being matched.
        
Simply substituting a variable with a blank node in the <tt>EXISTS</tt>
evaluation process does not cause the basic graph pattern to be 
to be restricted to subgraphs containing that blank node as an RDF term
because it is mapped by an 
[RDF instance mapping](https://www.w3.org/TR/2004/REC-rdf-mt-20040210/#definst)
before checking that the BGP after mapping is a subgraph of the
graph being queried.
        
Note that elsewhere in the evaluation of the SPARQL algebra, a solution
mapping with a binding from variable to blank node, does treat blank nodes
as RDF terms. They are not mapped by an RDF instance mapping.
        
Example:

    SELECT ?x WHERE {
        ?x :p :d .
        FILTER EXISTS { ?x :q :b . }
    }

against the graph <tt>{ _:c :p :d , :e :q :b }</tt>
the substitution for <tt>EXISTS</tt> produces
<tt>BGP(_:c :q :b)</tt> which then
matches against <tt>:e :q :b</tt> because the <tt>_:c</tt> can be mapped to <tt>:e</tt> by
the RDF instance mapping that is part of pattern instance mappings in 
[18.3.1](https://www.w3.org/TR/sparql11-query/#BGPsparql).

### Issue 4: Substitution can flip MINUS to its disjoint-domain case {#issue-4}

In

    SELECT ?x WHERE {
        ?x :p :c .
        FILTER EXISTS { ?x :p :c . MINUS { ?x :p :c . } }
    }

on the graph <tt>{ :d :p :c }</tt>
the substitution from 18.6 ends up producing

    Minus( BGP( :d :p :c ), BGP( :d :p :c ) )

which produces a non-empty result because the two solution mappings for the
Minus have disjoint domains and 18.5 dictates that then the result is not
empty.

### Issue 5: Substitution affects disconnected variables {#issue-5}

In

    SELECT ?x WHERE {
      BIND ( :d AS ?x )
      FILTER EXISTS { BIND ( :e AS ?z )
                      SELECT ?y WHERE { ?x :p :c } }
    }

the substitution from 18.6 ends up producing

      Join ( Extend( BGP(), ?z :e ),
             ToMultiSet( 
                 Project( ToList( BGP( :d :p :c ) ), { ?y } )
           ))

The `?x` inside the `SELECT ?y` is not projected out so it is a "different" `?x`
than the outer one - changing it to another other unused name in the same query
would not normally affect the query results.
    
## An Improved "substitute" Operation {#substitute-ng}

Evalauting
<tt>[substitute](https://www.w3.org/TR/sparql11-query/#defn_substitute)</tt> is
performed for a given solution mapping. For example, the `EXISTS` operation
evaluates to `true` if a graph pattern has one or more matches given the variable bindings
of a solution mapping.  We call this solution mapping the <dfn>current row</dfn>
in this description.

This section proposes an alternative mechanism.  Rather than replace each
variable by the value it is bound to in the current row, this alternative
mechanism makes the whole of the current row available at any point in the
evaluation of an <tt>EXISTS</tt> expression. It uses the current row to
restrict the binding of variables at the points where variable bindings are
created during evaluation of <tt>EXISTS</tt> to be those from the current row.
It makes illegal syntactic constructs that could lead to an attempt to rebind a
variable from the current row through using the <tt>AS</tt> syntax.

Section "[Addressing Issues](#addressing-issues)" describes how this alternative
definition of <tt>substitute</tt> addresses each of the issues identified above.
      
There are 3 parts to the proposal:

* Place the current row of mapping variables to value (the RDF terms) so that
  the variables always have their values from the current row.
  This is the replacement for syntactic substitution in the original definition.
* Renaming inner scope variables so that variables that are only used within a
  sub-query are not affected by the current row. This reflects the fact that in
  SPARQL such variables are not present in solutions mappings outside their
  sub-query.
* Disallow syntactic forms that set variables potentially already present in the current
  row. SPARQL solutions mappings can only have one binding for a variable and the
  current row provides that binding.

### Renaming

Within sub-queries, variables with the same name can be used but do not
appear in the overall results of the query if they do not occur in the
projection in the sub-query. Such inner variables are not 
<a href="https://www.w3.org/TR/sparql11-query/#variableScope">in-scope</a>
when they are not in the output of the projection part of the inner SELECT expression.
        
    SELECT * {
      ?s :value ?v .
      FILTER EXISTS {
         { SELECT (count(*) AS ?C) {
               ?s :property ?w .
           } 
         }
         FILTER ( ?C < ?v )
      }
    }

Here, the <tt>?s</tt> is not mentioned in the projection in 
<tt>SELECT (count(*) AS ?C)</tt>. Replacing <tt>?s</tt> by, for example,
<tt>?V1234</tt> in the sub-query does not change the overall results.
        
    SELECT * {
      ?s :value ?v .
      FILTER EXISTS {
         { SELECT (count(*) AS ?C) {
                 ?V1234 :property ?w .
           }
         }
       FILTER ( ?C < ?v )
      }
    }

Such variable usages can be replaced with a variable of a different name, if
that name is not used anywhere else in the query, and the same results are
obtained in the sub-query. A sub-query always has a projection as its top-most
algebra operator.

To preserve this, any such variables are renamed so they do not coincide with
variables from the current row being filtered by <tt>EXISTS</tt>.
        
The SPARQL algebra "project" operator has two components, an algebra expression
and a set of variables for the projection.

<div class="defn">
<b>Definition: <a id="defn_projmap" name="defn_projmap">Projection Expression Variable Remapping</a></b>
<p>
For a projection algebra operation P with set of variables PV, define
a partial mapping F from
<a href="https://www.w3.org/TR/sparql11-query/#sparqlQueryVariables">V</a>,
the set of all variables, to V where:
</p>
<p class="defn-expr">
F(v) = v if v in PV<br/>
F(v) = v1 where v is a variable mentioned in the project expression
       and v1 is a fresh variable<br/>
F(v) = v otherwise.
</p>
Define the <dfn>Projection Expression Variable Remapping</dfn> <tt>PrjMap(P,PV)</tt> to
be the algebra expression P (and the subtree over which the projection is
defined) with F applied to every variable of the algebra expression P over
which P is evaluated.
</div>
 
This process is applied throughout the graph pattern of <tt>EXISTS</tt>:

<div class="defn">
<b>Definition: <a id="defn_varrename" name="defn_varrename">Variable Remapping</a></b>
<p>
For any algebra expression X define the
<dfn>Variable Remapping</dfn> PrjMap(X):
</p>
<p class="defn-expr">
PrjMap(X) = replace all project operations <tt>project(P PV)</tt> with <tt>project(PrjMap(P,PV) PV)</tt> for each projection in X.
</p>
This replacement is applied bottom-up when there are multiple project
operations in the graph pattern of <tt>EXISTS</tt>.
</div>

Applying the renaming steps inside a sub-query does not change the solution
mappings resulting from evaluating the sub-query. Remapping is only applied to
variables not visible outside the sub-query. Renaming a variable in a SPARQL
algebra expression causes the variable name used in bindings from evaluating the
algebra expression to change. Since these are only variables that are not
visible outside the sub-query, because they do not occur in the projection, the
result of the sub-query is unchanged. SPARQL algebra expressions can not access
the name of a variable nor introduce a variable except by remapping. Remapping
is only applied to variables not visible outside the sub-query.

### Limitations on Assignment

SPARQL syntactic forms that attempt to bind a variable through the use of
<tt>AS</tt> that might already be in a solution mapping are forbidden in
SPARQL: this is covered in the syntactic restrictions of
<a href="https://www.w3.org/TR/sparql11-query/#sparqlGrammar">19.8 Grammar</a>, 
notes 12 and 13.

This proposal adds the restriction that any variables in a current
row, the set of variables 
<a href="https://www.w3.org/TR/sparql11-query/#variableScope">in-scope</a>
of the expression containing EXISTS, can not be assigned with the <tt>extend</tt>
algebra function linked to the <tt>AS</tt> syntax.

In addition, any use of <tt>VALUES</tt> in the EXISTS expression must not
use a variable in the current row.

### Restriction of Bindings
        
The proposal is to retain the variables from the current row, not
substitute them for RDF terms, before evaluation, and also to restrict 
the binding of the solution to the RDF term of the current row. This occurs
after renaming.

Binding for variables occur in several places in SPARQL:
        
* <a href="https://www.w3.org/TR/sparql11-query/#BGPsparql">Basic Graph Pattern Matching</a>
* <a href="https://www.w3.org/TR/sparql11-query/#PropertyPathPatterns">Property Path Patterns</a>
* The <a href="https://www.w3.org/TR/sparql11-query/#defn_evalGraph">evaluation of algebra form
   <tt>Graph(var,P)</tt></a> involving a variable (from the syntax <tt>GRAPH ?variable {...}</tt>)

Note that other places where solution mappings add variables are in
<tt>extend</tt> function (connected to the <tt>AS</tt> syntax)
and <tt>a multiset</tt> from <tt>VALUES</tt> syntax.
[Limitations on Assignment]("#limitations-on-assignment")
forbid this being of variables of the current row.

Restricting the RDF Terms for a variable binding is done using
inline data that is joined with the evalaution of the basic graph pattern,
property path or graph match.
     
<div class="defn">
<b>Definition: <a id="defn_valuesinsertion" name="defn_valuesinsertion">Values Insertion</a></b>
<p>
For solution mapping μ, define Table(μ) to be the multiset formed from μ.
</p>
<p class="defn-expr">
  Table(μ) = { μ }<br/>
  Card[μ] = 1
</p>
<p>
  Define the <dfn>Values Insertion</dfn> function <tt>Replace(X, μ)</tt> to
  replace each occurence Y of a 
  <a href="https://www.w3.org/TR/sparql11-query/#sparqlTranslateBasicGraphPatterns">Basic Graph Pattern</a>,
  <a href="https://www.w3.org/TR/sparql11-query/#sparqlTranslatePathExpressions">Property Path Expression</a>,
  <a href="https://www.w3.org/TR/sparql11-query/#sparqlTranslateGraphPatterns">Graph(Var, pattern)</a>
  in X with join(Y, Table(μ)).
</p>
</div>

### Evaluation of EXISTS

The evaluation of <tt>EXISTS</tt> is defined as:
        
<div class="defn">
<b>Definition: <a id="defn_valuesinsertion" name="defn_valuesinsertion">Evaluation of Exists</a></b>
<p>
Let μ be the current solution mapping for a filter and X a graph pattern,
define the <dfn>Evaluation of Exists</dfn> <tt>exists(X)</tt>
</p>
<p class="defn-expr">
  exists(X) = true if eval(D(G), Replace(PrjMap(X), μ) is a non-empty solution sequence.
  <br/>
  exists(X) = false otherwise
</p>
</div>

## Addressing Issues {#addressing-issues}

This section addresses each issue identified, given the proposal above.
      
### Issue 1: Some uses of EXISTS are not defined during evaluation
        
This can be handled by handling solution sequences as graph patterns where
needed by adding 
<a href="https://www.w3.org/TR/sparql11-query/#defn_algToMultiSet">toMultiSet</a>
as is done for <a href="https://www.w3.org/TR/sparql11-query/#rSubSelect">SubSelect</a>
in <a href="https://www.w3.org/TR/sparql11-query/#sparqlTranslateGraphPatterns">18.2.2.6 Translate Graph Patterns</a>
with a a correction to the text at the end of
<a href="https://www.w3.org/TR/sparql11-query/#sparqlQuery">Section 18.2</a>
introductory paragraph.

<pre class="box">
query-errata-N:

"Section 18.2 Translation to the SPARQL Algebra" intro (end):

ToMultiSet can be used where a graph pattern is mentioned below because the
outcome of evaluating a graph pattern is a multiset.

Multisets of solution mappings are elements of the SPARQL algebra.  Multisets
of solution mappings count as graph patterns.
</pre>

### Issue 2: Substitution happens where definitions are only for variables

Rather then replace a variable by its value in the current row, the new
mechanism makes the binding of variable to value available. The variable
remains in the graph pattern of <tt>EXISTS</tt> and the evaluation.

### Issue 3: Blank nodes substituted into BGPs act as variables
 
By making the current row, which can include blank nodes, available, and not
modifying the BGP by substitution, no blank nodes are introduced into the
evalaution of the BGP. Instead, the possible solutions is restricted by the
current row.
         
### Issue 4: Substitution can flip MINUS to its disjoint-domain case
 
Issue 4 is addressed because variables are not removed from the domain of
<tt>MINUS</tt>. This propsoal does not preserve all uses of <tt>MINUS</tt>
expressions; the problem identified in issue 4 is considered to be a bug in the
SPARQL 1.1 specification.
         
### Issue 5: Substitution affects disconnected variables

Issue 5 is addressed by noting that variables inside sub-queries which are not
projected can be renamed without affecting the sub-query results. Whether to
preserve that invariant or allow the variables to be set by the current row is a
choice point - this design preserves the independence of disconnected variables.

## Notes {#notes}

The proposal described in this document does not cover use of variables from
the current row in a <tt>HAVING</tt> clause.
