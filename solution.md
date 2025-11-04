# Solution Code - User Retention Snapshot
<details>
<summary>Step 1 Solution  - Explore the Tables and Relationships</summary>
    
```sql
-- Preview some data
SELECT * FROM users LIMIT 5;
SELECT * FROM sessions LIMIT 5;
    
-- ** Additional ideas **
-- View total number of records for each table
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM sessions;
    
-- Check to be sure there are no sessions without a user
SELECT COUNT(*) AS orphan_sessions
FROM sessions AS s
LEFT JOIN users AS u 
ON s.user_id = u.user_id
WHERE u.user_id IS NULL;
    
-- Check to be sure no users had a session before signing up
SELECT COUNT(*) AS sessions_before_signup
FROM sessions AS s
JOIN users AS u 
ON s.user_id = u.user_id
WHERE date(s.session_date) < date(u.signup_date);
```
</details>
<details>
<summary>Step 2 Solution - Identify Each User’s First Session</summary>
    
```sql
SELECT
  user_id,
  min(date(session_date)) AS first_session_date
FROM sessions
GROUP BY user_id;
    
-- Note: Since the session_date column is stored as TEXT, you
-- need to use the date() function first to convert it.
```
</details>
<details>    
<summary>Step 3 Solution - Join First Sessions to the Session Log</summary>
    
```sql
-- First connect the sessions table to the subquery. Just SELECT 
-- everything for now.
SELECT *
FROM sessions AS s
JOIN (
  SELECT
    user_id,
    min(date(session_date)) AS first_session_date
  FROM sessions
  GROUP BY user_id
) AS fs
ON fs.user_id = s.user_id;
    
-- Add the WHERE clause to get everything after the first session.
SELECT *
FROM sessions AS s
JOIN (
  SELECT
    user_id,
    min(date(session_date)) AS first_session_date
  FROM sessions
  GROUP BY user_id
) AS fs
ON fs.user_id = s.user_id
WHERE date(s.session_date) > date(fs.first_session_date);
    
-- Now JOIN the users table as well. SELECT only specific columns
-- to clean up the results, and maybe add a nice ORDER BY.
SELECT
  s.user_id,
  s.session_date,
  u.region,
  s.device_type
FROM sessions AS s
JOIN (
  SELECT 
    user_id, 
    min(date(session_date)) AS first_session_date
  FROM sessions
  GROUP BY user_id
) AS fs
  ON fs.user_id = s.user_id
JOIN users AS u
  ON u.user_id = s.user_id
WHERE date(s.session_date) > date(fs.first_session_date)
ORDER BY s.user_id, s.session_date;
```
</details>
<details>    
<summary>Step 4 Solution - Group and Summarize Activity</summary>
    
```sql
-- Amend your previous query. You only need to SELECT region and device_type,
-- and then add a "session count".
-- In this case you're GROUPing BY both region and device_type, and then 
-- ORDERing BY the session count:
SELECT
  u.region,
  s.device_type,
  COUNT(*) AS session_count
FROM sessions s
JOIN (
  SELECT 
    user_id, 
    min(date(session_date)) AS first_session_date
  FROM sessions
  GROUP BY user_id
) AS fs
  ON fs.user_id = s.user_id
JOIN users u
  ON u.user_id = s.user_id
WHERE date(s.session_date) > date(fs.first_session_date)
GROUP BY u.region, s.device_type
ORDER BY session_count DESC;
    
-- Below is an example of using all the sessions (not just the "return visits").
-- It shows how to get the year & week from the date, and then use that
-- for GROUPing and ORDERing.
SELECT
  strftime('%Y-%W', s.session_date) AS year_week,
  COUNT(*) AS sessions_in_week
FROM sessions AS s
GROUP BY year_week
ORDER BY year_week;
-- Note that if you joined to the users table as before and SELECTed the 
-- region and device type data, those details would be lost when the records  
-- are grouped by year_week.
```
</details>
<details>    
<summary>Step 5 Solution - Compare New vs. Returning Sessions</summary>
    
```sql
-- First get the CASE working
SELECT
  CASE
    WHEN date(s.session_date) =
          (SELECT min(date(x.session_date))
          FROM sessions AS x
          WHERE x.user_id = s.user_id)
    THEN 'new'
    ELSE 'returning'
  END AS session_type,
  COUNT(*) AS session_count
FROM sessions AS s
GROUP BY session_type;
    
-- Now JOIN to the users table (to get the regions). Then 
-- SELECT and GROUP BY region.
SELECT
  u.region,
  CASE
    WHEN date(s.session_date) =
          (SELECT min(date(x.session_date))
          FROM sessions AS x
          WHERE x.user_id = s.user_id)
    THEN 'new'
    ELSE 'returning'
  END AS session_type,
  COUNT(*) AS session_count
FROM sessions AS s
JOIN users u ON u.user_id = s.user_id
GROUP BY u.region, session_type;
    
-- You could also add in the device type.
SELECT
  u.region,
  s.device_type,
  CASE
    WHEN date(s.session_date) =
          (SELECT min(date(x.session_date))
          FROM sessions AS x
          WHERE x.user_id = s.user_id)
    THEN 'new'
    ELSE 'returning'
  END AS session_type,
  COUNT(*) AS session_count
FROM sessions AS s
JOIN users u ON u.user_id = s.user_id
GROUP BY u.region, s.device_type, session_type;
```
</details>