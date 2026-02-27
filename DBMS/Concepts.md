![[DB concepts.png]]


### USING (column_name)
[Start USING() for JOINs instead of ON](https://x.com/RealBenjizo/status/2027088163156472218?s=20)
```sql
SELECT                              | SELECT 
       e.Employee_ID,               |       e.Employee_ID, 
       e.Employee_Name,             |       e.Employee_Name,
       e.Department,                |       e.Department,
       s. Net_Sales                 |       s. Net_Sale 
FROM Employees e                    | FROM Employees e 
JOIN sales s                        | JOIN sales s 
ON e.Employee_ID = s.Employee_ID    | USING (Employee_ID); 
```