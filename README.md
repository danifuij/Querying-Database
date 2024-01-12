# SQL-Query
## Database Schema
![image](https://github.com/danifuij/SQL-Query/assets/124475097/01c8c417-0ee1-4c1f-a8eb-16e1a64fc826)

**Deals table**

The main table of operations of the CRM system.

The primary key is object_id.

**Contacts table**

Contains customer data.

The primary key is object_id.

**Engagements table**

Keep data on the actions of employees who have CRM management access (for example, a department manager
sales).

The primary key is object_id.

Columns:
- type – indicates the type of action, for example, values: call – call, meeting – pre-booked call, email –
sent letter, closed_at - date and time of event closing.
- created_at – indicates the date and time the event was created, the event began to take place.
- closed_at – indicates the closing date of the created event.
  
**Stages table**

Contains data about the stage of the operation and the funnel.
The primary key is object_id.
Columns:
- stage_label – the stage in which the client is (for example, just arrived, already called, bought the product).
- pipeline – a funnel to which the client belongs (for example, a Ukrainian project or a Polish project).
  
**Associations table**

A table of connections, with the help of which we can connect other tables. 
The primary key is missing.

Columns:
- from – denotes the object_id of the main table.
- to – denotes the object_id of the table with which the main one is linked.
- type – indicates which tables are connected. There are 2 types: deal_contact, deal_engagement.

**Additional Information**

In our CRM system, there is such a thing as an entity, which means any event that happened in the system, simple
in words - any row in the database tables.

Each of the tables except Associations has an is_deleted column that returns 1 if the entity has already been deleted
directly by the manager/owner of the entity from the CRM system and 0 if not.

## Task:

The customer set the task. It is necessary to build a SQL query, which will have a table with the following columns:
- pipeline
- product
- stage_label
- deals.created_year_and_month
- contacts.email
- country
- city
- full_name --first_name last_name
- calls_number (count of calls from engagements)
- mean_calls_duration_sec (median time of calls duration)
- days_to_meet (median time in days from created_at of Deals until created_at of Engagements)

  ## Solution:
  
  ```SQL WITH EngDuration AS (
    SELECT
        DATEDIFF(SECOND, closed_at, created_at) AS duration_sec
    FROM Engagements
    ORDER BY duration_sec
    OFFSET (SELECT COUNT(*) FROM Engagements) / 2 ROWS
    FETCH NEXT 1 + (SELECT COUNT(*) % 2) ROWS ONLY
  )
    SELECT
    s.pipeline,
    d.product,
    s.stage_label,
    DATE_FORMAT(d.created_at, '%Y-%m') AS created_y_and_m,
    c.email,
    c.country,
    c.city,
    CONCAT(c.first_name, ' ', c.last_name) AS full_name,
    COUNT(CASE WHEN e.type = 'call' THEN 1 END) AS calls_number,
    (
        SELECT
            (MAX(duration_sec) + MIN(duration_sec)) / 2
        FROM EngDuration
    ) AS mean_calls_duration_sec,
    CASE
        WHEN COUNT(*) % 2 = 0 THEN AVG(DATEDIFF(eng.created_at, d.created_at))
        ELSE (
            SELECT DATEDIFF(eng.created_at, d.created_at)
            FROM Engagements AS eng
            JOIN Associations AS assoc ON eng.object_id = assoc.from AND assoc.type = 'deal_engagement'
            WHERE assoc.to = d.object_id AND eng.type = 'meeting'
            ORDER BY DATEDIFF(eng.created_at, d.created_at)
            LIMIT 1 OFFSET COUNT(*) / 2
        )
    END AS days_to_meet
    FROM
    Deals AS d
    JOIN Associations AS a ON d.object_id = a.from AND a.type = 'deal_contact'
    JOIN Contacts AS c ON c.object_id = a.to
    JOIN Stages AS s ON d.stage_id = s.object_id
    LEFT JOIN Associations AS a2 ON d.object_id = a2.from AND a2.type = 'deal_engagement'
    LEFT JOIN Engagements AS e ON e.object_id = a2.to AND e.type = 'call'
    GROUP BY
    s.pipeline, d.product, s.stage_label, created_y_and_m, c.email, c.country, c.city, full_name
    ORDER BY
    created_y_and_m, full_name, s.pipeline, d.product, s.stage_label;
    ```
