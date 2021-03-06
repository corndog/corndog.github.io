# Another Take on Trees in SQL

Consider the problem of representing the structure of an html form in a database. You/your clients rough specs view the form as a heirarchical structure. There is the form itself, which probably has a title, then they want sets of related questions to be presented under their own headings. Possibly individual questions, from the users' perspective, consist of multiple form inputs, such as an address section made up of several text inputs.

Furthermore different pieces need to be reused across forms. Consider a course evaluation system, an online version of the bubble sheets colleges use for students to evaluate professors, may need to have a set of questions common across the college, then other groups of questions that individual departments want asked for their freshman classes, for example.
The two implementations I've worked on before essentially had a table for level of the heirarchy they settled on. Some issues associated with this were inflexiblity and naive ORM usage with an N + 1 query problem at each level of the heirarchy.

Eventually the question became, can we represent our heirarchy in a flexible way, allowing reuse of nodes in the tree, and fetch all the information needed to display the form in a single query?

## Recursive Common Table Expressions in Postgres

Consider the following table and data

ancestry:
id   | name | parent
1	 | fred | null
2    | jill | null
3    | jane | 2
4    | bob  | 1
5    | dave | 4

We can select the names of everyone in Dave's family tree with this query:

WITH RECURSIVE family(name, parent) AS (
 SELECT name, parent from ancestry an where id = 5
 UNION ALL
 SELECT a.name, a.parent 
 FROM ancestry a
 JOIN family f ON a.id = f.parent
)
SELECT name from family

This begins with the nonrecursive part:
SELECT an.name, an.parent from ancestry an where id = 5
The results of this query are added to a result set, and then placed in a working set. In this simple example this is just one row, with Dave in it. This working set is referred to in the recursive step of the query, family points to it's data. So when we join family (initially Dave) on ancestry we get Bob.
 In general, at each step any new rows returned are added to the result set and used as the working set in the next iteration. When no new rows are found the process is complete and the result set is available to the final select query.

 This introduces recursive CTEs, but for our purposes is limited in that each node only has one parent, so we can't reuse children/subtrees.

Consider another table and data, component_ids is an array.

components:
id | component_type | label 	| component_ids
1  | text			| name 		| null
2  | text 			| address 1 | null
3  | text	  		| address 2 | null
4  | text 			| city		| null
5  | text 			| zip		| null
6  | address 		| address	| { 2, 3, 4, 5}
7  | form 			| details   | { 1, 6}

Row 6 describes an address component, made up of 4 pieces, address 1 and 2, city and zip. Row 7 describes a form component made up of a name and an address. The following query selects all the rows necessary to construct a given node/subtree, in this case for the address component.

WITH RECURSIVE all_components (id, component_type, label, component_ids, root_id) as (
 select id, component_type, label, component_ids, id from components
 UNION ALL
 select q1.id, q1.component_type, q1.label, q1.component_ids, ac.root_id
 from components q1
 JOIN all_components ac ON q1.id = ANY (ac.component_ids)
 )
select id, component_type, label, component_ids, root_id from all_components where root_id = 6

The most interesting thing here is that we can join on the contents of an array. This allows to build our tree top down, each node having an arbitrary number of children, and nodes can be children of any number of parents. The other thing to note is we include the id of the root element in each row so it can be used in the where clause to select all the rows we want.

Returning to our original problem of representing form inputs, we would probably want a value column as well to represent say an option in a select list. Anyway, since both our conceptual structure of the form and html itself is a tree we can explore a lot of ideas in this fairly simple framework.

