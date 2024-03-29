# Library Database Queries Overview

## Patron Activity Report
Generates a report on patron activity, including average fines, number of negative and positive comments on returns, frequency of overdue and early returns, and each patron's favorite genre.

### Multiple Joined Subqueries
The query incorporates several subqueries to fetch different pieces of data:
- Subquery for average fines.
- Subqueries for counting negative and positive comments based on specific keywords in comments.
- Subquery for overdue returns.
- Subquery for early returns.
- Subquery to determine each patron's favorite genre based on the frequency of loans per genre.

### Aggregate Functions and Conditional Logic
Uses aggregate functions and conditional logic to calculate averages and counts, and to filter comments.

### Row Number Window Function
Utilizes the `row_number()` window function to rank the favorite genres for each patron.

### Grouping and Ordering Results
Groups the results by patron and favorite genre, and orders by favorite genre and patron ID.

### SQL Query

```sql
select 		l.patron_id "ID", concat(p.fname, ' ', p.lname) "Name"
		, round(avg(fine), 2) "Fines (AVG)"
		, count(comments_neg.id) "# Negative Comments"
		, count(comments_pos.id) "# Positive Comments"
		, count(overdue.id) "# Overdue Returns"
		, count(early_returns.id) "# Early Returns"
		, fav.genre "Favorite Genre"
from 		loans l
		left join 	patrons p on l.patron_id = p.id
		left join 	( select 	fine, id from 	returns ) fines on fines.id = l.return_id
		left join 	( select 	 id, comment
				from 	returns 
				where 	comment like '%late%'
					or comment like '%damage%'
					or comment like 'lost'
					or comment like '%missing%' ) comments_neg  on comments_neg.id = l.return_id
		left join 	( select 	id, comment
				from 	returns 
				where 	comment not like '%late%'
					and comment not like '%damage%'
					and comment not like '%lost%'
					and comment not like '%missing%' ) comments_pos on comments_pos.id = l.return_id		
		left join 	( select 	id, overdue from 	returns where 	overdue = true) overdue on overdue.id = l.return_id	
		left join 	( select  id from returns where 	comment like '%early%' ) early_returns on early_returns.id = l.return_id	
		left join 	( select 	patron_id, genre		
				from		(select 	*, row_number() over(partition by patron_id, genre order by count desc) "copy"
						from 		(select 	count(*) "count", patron_id, genre
								from 		loans l
										join books b on b.id = l.book_id
										join genres g on g.id = b.genre_id 	
								group by 	patron_id, genre) x) y
where 		copy = 1 )fav on fav.patron_id = l.patron_id				
group by  	l.patron_id, concat(p.fname, ' ', p.lname), fav.genre 
order by 	"Favorite Genre", "ID";
```

## Time Series Analysis of Library's Loan Data
Analyzes the number of loans per month and the growth percentage from the previous month.

### With Clause
The `WITH` clause creates a temporary result set (`x`) that is calculated as follows:
- Extracts the `year` and `month` from `c.date` and labels them as `"Year"` and `"Month #"` respectively.
- Retrieves the full month name and labels it as `"Month"`.
- Determines the previous month number and labels it as `"Previous Month #"`.
- Counts the number of loans per month and labels the result as `"Loans"`.

### Main Select Clause
The main query calculates:
- The number of loans for each month labeled as `"# of Loans"`.
- The growth percentage from the previous month, calculated and labeled as `"Growth (Prev. Month)"`.

### SQL Query

```sql
-- time series analysis: # of loans per month & growth % from previous month
with x as (
	select 		date_part('year', c.date) "Year", 
			date_part('month', c.date) "Month #", 
			to_char(c.date, 'Month') "Month",
			date_part('month', (c.date - interval '1 month')) "Previous Month #",
			count(*) as "Loans"
	from 		loans
			join returns r on r.id = loans.return_id
			join checkouts c on c.id = loans.checkout_id
	group by 	date_part('year', c.date), date_part('month', c.date), 
			date_part('month', (c.date - interval '1 month')), 
			to_char(c.date, 'Month')
	order by	"Year", "Month #" desc
)

select 	  "Year"
	, "Month"
	, "Loans" as "# of Loans"
	, case
		 when x2."Loans" - (select x."Loans" from x where x."Month #" = x2."Previous Month #") is null 
			 then '0.00 %'
		 else   
			concat(
				cast (((
					round(((
						cast(x2."Loans" - (select x."Loans" from x where x."Month #" = x2."Previous Month #") as numeric)) / x2."Loans"), 2) * 100)) as text), ' %')
		end as "Growth (Prev. Month)"
from		x as x2 
; 
```

## General Overview of Library's Book Collection
Lists titles, ISBNs, genres, and conditions of books, sorted by genre and title.

### Select Clause
The `SELECT` clause specifies the columns to be retrieved:
- `title` is renamed as `"Title"`.
- `isbn` is renamed as `"ISBN"`.
- `genre` (from the `genres` table) is renamed as `"Genre"`.
- `condition` (from the `conditions` table) is renamed as `"Condition"`.

### From Clause
The main table in the query is `books` (aliased as `b`).

### Joins
- **Left Join with Genres**: Links the `genres` table (aliased as `g`) on `g.id = b.genre_id`.
- **Left Join with Conditions**: Links the `conditions` table (aliased as `c`) on `c.id = b.condition_id`.
- **Left Join with Statuses**: Links the `statuses` table (aliased as `s`) on `s.id = b.status_id`.

### Order By Clause
Results are sorted first by `"Genre"` and then by `"Title"`.

```sql
select 		--*,
			title "Title", isbn "ISBN", genre "Genre", condition "Condition"
from		books b
			left join genres g on g.id = b.genre_id 
			left join conditions c on c.id = b.condition_id
			left join statuses s on s.id = b.status_id
order by	Genre, Title;
```

## List of Available Books and Author Information
Displays details of available books and their authors, ordered by book title.

### Select Clause
- Selects `title` from `books` table and renames it as `"Book Title"`.
- Concatenates `fname`, `mi`, and `lname` from `authors` table and renames the result as `"Full Name"`.

### From Clause
- The main table is `book_authors` (aliased as `ba`).

### Joins
- **Left Join with Authors**: Connects `authors` table (aliased as `a`) on `ba.author_id = a.id`.
- **Left Join with Books**: Connects `books` table (aliased as `b`) on `ba.book_id = b.id`.
- **Left Join with Statuses**: Connects `statuses` table (aliased as `s`) on `b.status_id = s.id`.

### Where Clause
- Filters the results to include only books with `status` marked as `'Available'`.

### Group By Clause
- Groups results by `"Full Name"` and `"Book Title"`.

### Order By Clause
- Orders results by `"Book Title"`.

### Aggregation
- Uses the `string_agg` function to aggregate `"Full Name"` values per `"Book Title"` into a single field named `Author`.

```sql
with x as (
    select      --*,
                title as "Book Title",
                concat(fname,' ', mi, '. ', lname) as "Full Name"
    from        book_authors  ba
                left join authors a on ba.author_id = a.id
                left join books b on ba.book_id = b.id 
                left join statuses s on b.status_id = s.id
    where       status in ('Available')
    group by    "Full Name", "Book Title"
    order by    "Book Title"
)

select          x."Book Title",
                string_agg("Full Name", ', ') as Author          
from            x 
group by        x."Book Title"
order by        x."Book Title"
```

## List of Books Never Loaned Out
Displays books from the library's collection that have never been loaned out, sorted by title and author.

### Select Clause
- Selects `title` from `books` table, renamed as `"Book Title"`.
- Selects `fname`, `lname`, `mi` from `authors` table, renamed as `"Author TItle"`.

### From Clause
- Main table is `book_authors` (aliased as `ba`).

### Joins
- **Left Join with Authors**: Joins `authors` table (aliased as `a`) on `ba.author_id = a.id`.
- **Left Join with Books**: Joins `books` table (aliased as `b`) on `ba.book_id = b.id`.
- **Left Join with Statuses**: Joins `statuses` table (aliased as `s`) on `b.status_id = s.id`.

### Where Clause
- Excludes books that have been loaned out (`b.id not in (select book_id from loans)`).

### Order By Clause
- Results ordered by `Book Title`.

```sql
with x as (
	select 		--*,
				title as "Book Title",
				concat(fname,' ', mi, '. ', lname) as "Full Name"
	from 		book_authors  ba
				left join authors a on ba.author_id = a.id
				left join books b on ba.book_id = b.id 
				left join statuses s on b.status_id = s.id
	where 		b.id not in (select book_id from loans)
	order by 	"Book Title"
)

select		x."Book Title",
			string_agg("Full Name", ', ') as Author			
from 		x 
group by 	x."Book Title"
order by 	x."Book Title"
```

## Average Condition of All Books in Collection
Presents an overview of the conditions of all books in the collection, showing the count and percentage of each condition.

### Select Clause
- Selects `condition` from `conditions` table, renamed as `"Condition"`.
- Calculates the count of each condition with `count(*)`, renamed as `"Count"`.
- Calculates the percentage of books in each condition with `concat((count(*) * 100/(select count(*) from books))`, renamed as `"Percent"`.

### From Clause
- Main table is `books` (aliased as `b`).

### Joins
- **Left Join with Conditions**: Joins `conditions` table (aliased as `c`) on `c.id = b.condition_id`.

### Group By Clause
- Groups results by `condition`.

### Order By Clause
- Orders results by `"Count"` in descending order, then by `condition`.

```sql
select 		condition "Condition",
			count(*) "Count",
			concat((count(*) * 100/(select count(*) from books)) , '% of Books') "Percent"
from		books b
			left join conditions c on c.id = b.condition_id
group by 	condition
order by	"Count" desc, condition;
```

## Number of Days a Book Has Been Checked Out
Calculates the number of days each book has been checked out from the library.

### With Clause (Common Table Expression - CTE)
- **x**: A CTE that selects all fields from `books`, `loans`, `returns`, and `checkouts`. It calculates "Return Date" and "Checkout Date" using CASE statements. If the return date is not null, it uses the return date; otherwise, it uses the current date. Similarly, for checkout date.

### Select Clause
- Selects `title` from `books` and renames it as `"Book Title"`.
- Calculates the difference between "Return Date" and "Checkout Date" to find out the total days loaned, renamed as `"Days Loaned"`.

### From Clause
- Uses the CTE `x`.

### Order By Clause
- Orders results by `"Days Loaned"` in descending order.

```sql
with x as (
	select		*,
				case when r.date is not null 
					then r.date 
					else current_date 
				end as "Return Date",
				case when c.date is not null 
					then c.date
					else current_date
				end as "Checkout Date"
	from		books b
				left join loans l  on b.id = l.book_id 
				left join returns r on r.id = l.return_id
				left join checkouts c on c.id = l.checkout_id
) 

select 		title "Book Title",
			("Return Date" - "Checkout Date") as "Days Loaned" 
from		x
order by	"Days Loaned" desc;
```

## Overdue Returns
Identifies overdue returns in the library system, including the patron's details, book information, checkout and return dates, and fines.

### With Clause (Common Table Expression - CTE)
- **x**: A CTE that selects concatenated patron names, book title, checkout and due dates, return date, days overdue, and fine amount. It joins `loans`, `checkouts`, `returns`, `books`, and `patrons`.

### Select Clause
- Selects all fields from the CTE `x`.

### From Clause
- Uses the CTE `x`.

### Where Clause
- Filters the results to include only records where `"Days Overdue"` is greater than 0.

### Order By Clause
- Orders results by `"Days Overdue"` in descending order.

```sql
with x as ( 
	select 	concat(p.fname, ' ', p.lname) as "Patron",
			b.title "Book",
			c.date "Checkout",
			l.due "Due",
			r.date "Returned",
			(r.date - l.due) "Days Overdue",
			concat('$',r.fine) "Fine"
	from 	loans l
			join checkouts c on c.id = l.checkout_id
			join returns r on r.id = l.return_id
			join books b on b.id = l.book_id
			join patrons p on p.id = l. patron_id
)

select 		* 
from 		x 
where 		"Days Overdue" > 0
order by	"Days Overdue" desc;
```

## Average Cost of Fines by Book Genre
Calculates the average fine amount for overdue books, grouped by book genre.

### With Clause (Common Table Expression - CTE)
- **x**: A CTE that selects all fields, calculates "Days Overdue" for each loan by comparing return and due dates. It joins `loans`, `returns`, `books`, and `genres`.

### Select Clause
- Selects `genre` from `genres` table and renames it as `"Genre"`.
- Calculates average fine rounded to two decimal places and concatenates with `$` symbol. Renamed as `"Fine (AVG)"`.

### From Clause
- Uses the CTE `x`.

### Where Clause
- Filters the results to include only records where `"Days Overdue"` is greater than 0.

### Group By Clause
- Groups results by `genre`.

### Order By Clause
- Orders results by `genre`.

```sql
with x as ( 
	select 	*,
			(r.date - l.due) "Days Overdue"
	from 	loans l
			join returns r on r.id = l.return_id
			join books b on b.id = l.book_id
			join genres g on g.id = b.genre_id
)

select 		genre "Genre",
			concat('$', round(avg (fine), 2)) "Fine (AVG)"
from 		x 
where 		"Days Overdue" > 0
group by	"Genre"
order by	"Genre";
```

## Staff Record Creation Count in Database System
Displays the count of various types of records created by each staff member.

### With Clause (Common Table Expression - CTE)
- **x**: A CTE that selects the `created_staff_id` and assigns a label to each record type from various tables (`books`, `checkouts`, `loans`, `patrons`, `returns`, `staff`) using the `union` operator.

### Select Clause
- Concatenates `fname` and `lname` from the `staff` table to form the `"Name"` of the staff member.
- Counts the occurrences of each `"Record Type"` associated with a staff member.

### From Clause
- Joins the CTE `x` with the `staff` table.

### Group By Clause
- Groups results by the concatenated `"Name"`.

### Order By Clause
- Orders results by `"Name"`.

```sql
with x as (
	select		created_staff_id "ID", case when 1=1 then 'Books' end as "Record Type"
	from		books
	union
	select		created_staff_id "ID", case when 1=1 then 'Checkouts' end  "Record Type"
	from		checkouts
	union
	select		created_staff_id "ID", case when 1=1 then 'Loans' end  "Record Type"
	from		loans
	union
	select		created_staff_id "ID", case when 1=1 then 'Patrons' end  "Record Type"
	from		patrons
	union
	select		created_staff_id "ID", case when 1=1 then 'Returns' end  "Record Type"
	from		returns
	union
	select		created_staff_id "ID", case when 1=1 then 'Staff' end  "Record Type"
	from		staff 			
)

select 		concat(s.fname, ' ', s.lname) "Name", count("Record Type") "Count"
from		x
			join staff s on s.id = x."ID" 
group by 	"Name"
order by 	"Name";
```
