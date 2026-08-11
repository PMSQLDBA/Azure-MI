Previous: 
Great. The correct answer to the challenge is **D — DNS/network configuration on the application side**.

Why? **SSMS works, MI is healthy, and DNS fails specifically from the application server.** 
There is no reason to start changing indexes, vCores, or queries when the application cannot even resolve the MI endpoint.

# Lesson 4 — Azure SQL Managed Instance Security & Authentication

This is another area where **SQL Server DBA knowledge transfers directly to Azure MI**.

Our connection path is:

```text
Application
    |
    v
DNS
    |
    v
Network
    |
    v
SQL Managed Instance
    |
    v
Authentication
    |
    v
Authorization
    |
    v
Database/Object
```

The important distinction is:

> **Authentication = Who are you?**

> **Authorization = What are you allowed to do?**

---

## 1. SQL Server security model

Suppose our application connects as:

```text
app_sales
```

The flow is:

```text
app_sales
    |
    ↓
Login
    |
    ↓
User
    |
    ↓
Database
    |
    ↓
Roles
    |
    ↓
Permissions
```

For example:

```text
Login: app_sales

        ↓

User: app_sales

        ↓

SalesDB

        ↓

db_datareader
db_datawriter
```

The application can then perform whatever operations its permissions allow.

---

# 2. Login vs User

This is a classic SQL Server interview question.

### Login

A **login** is associated with the SQL Server/Managed Instance level.

Think:

```text
SQL Managed Instance
       |
       +-- Login A
       +-- Login B
       +-- Login C
```

### User

A **user** exists inside a database.

```text
SalesDB
   |
   +-- User A
   +-- User B
```

So:

```text
Instance
   |
   +-- Login
          |
          ↓
       Database
          |
          +-- User
```

A login may be mapped to a database user.

---

# 3. Example

Create a login:

```sql
CREATE LOGIN app_sales
WITH PASSWORD = 'StrongPasswordHere';
```

Then inside SalesDB:

```sql
USE SalesDB;
GO

CREATE USER app_sales
FOR LOGIN app_sales;
```

Then assign permissions:

```sql
ALTER ROLE db_datareader
ADD MEMBER app_sales;
```

Now:

```text
app_sales
   |
   ↓
Login
   |
   ↓
SalesDB user
   |
   ↓
db_datareader
```

The application can read data.

---

# 4. Authentication vs authorization

Imagine the application connects.

### Authentication succeeds

```text
Username/password correct
        ↓
"You are app_sales."
```

Then authorization happens:

```text
app_sales
    ↓
Can you SELECT SalesDB.dbo.Orders?
```

If yes:

```text
SELECT succeeds
```

If no:

```text
Permission denied
```

This distinction is extremely useful for troubleshooting.

---

# 5. Real production incident

The application team reports:

> **"We are getting Login failed for user app_sales."**

You investigate.

First:

```text
DNS → SUCCESS
Network → SUCCESS
MI → HEALTHY
```

Now investigate authentication.

Possible causes:

```text
Wrong password
Login disabled
Wrong authentication method
Connection string incorrect
Login doesn't exist
Password expired/changed
```

Notice that we haven't started looking at table permissions yet.

Why?

Because:

> **The user hasn't successfully authenticated.**

---

# 6. Different incident: Permission denied

Now suppose the error is:

```text
SELECT permission denied on object 'Orders'
```

That's different.

The login succeeded:

```text
Authentication
     ↓
SUCCESS
```

But authorization failed:

```text
Authorization
     ↓
FAILED
```

Now investigate:

```text
User mapping
Role membership
Object permissions
Schema permissions
Database permissions
```

---

# 7. Microsoft Entra authentication

Modern Azure environments often use **Microsoft Entra ID** rather than relying exclusively on SQL authentication.

Conceptually:

```text
Application/User
       |
       v
Microsoft Entra ID
       |
       | Authentication
       v
SQL Managed Instance
       |
       v
Database permissions
```

This can integrate with enterprise identity management.

Instead of:

```text
username + SQL password
```

you can use:

```text
Azure identity
```

depending on the application and authentication method.

---

# 8. Why enterprises like Entra authentication

Imagine ABC Retail has:

```text
5,000 employees
```

They don't want database administrators manually managing passwords everywhere.

Instead:

```text
Employee
   ↓
Corporate identity
   ↓
Microsoft Entra ID
   ↓
SQL MI
```

Access can be managed through organizational identity and groups.

For example:

```text
Entra Group:
SQL-Reporting-Users

        ↓

SQL MI

        ↓

ReportingDB

        ↓

SELECT permissions
```

Then when an employee joins/leaves a team, access can be managed through identity/group membership rather than manually changing individual SQL permissions everywhere.

---

# 9. Least privilege

This is one of the most important security principles.

Don't give the application:

```text
sysadmin
```

just because it needs:

```text
SELECT
INSERT
UPDATE
```

Instead:

```text
Application
    ↓
Specific login/user
    ↓
Specific database
    ↓
Specific role
    ↓
Required permissions
```

For example:

```text
app_sales
    ↓
SalesDB
    ↓
SalesAppRole
    ↓
SELECT / INSERT / UPDATE
```

Not:

```text
app_sales
    ↓
sysadmin
```

---

# 10. Why `db_owner` can be dangerous

Suppose someone says:

> "The application is getting permission errors. Just put it in db_owner."

That may fix the immediate problem, but it's poor security practice.

`db_owner` gives extremely broad control over the database.

Instead, determine:

```text
What does the application actually need?
```

Maybe:

```text
SELECT → Orders
INSERT → Orders
UPDATE → Orders
EXECUTE → Stored Procedures
```

Then grant only those permissions.

---

# 11. Stored procedures and application security

A common enterprise design is:

```text
Application
    |
    ↓
EXEC dbo.CreateOrder
    |
    ↓
Stored Procedure
    |
    ↓
Orders table
```

Instead of giving the application direct permissions on every table, permissions can be designed around controlled stored-procedure access where appropriate.

For example:

```sql
GRANT EXECUTE
ON OBJECT::dbo.CreateOrder
TO app_sales;
```

This can provide a more controlled permission model.

---

# 12. Database roles

Think of roles as permission containers.

For example:

```text
SalesAppRole
    |
    +-- SELECT Orders
    +-- INSERT Orders
    +-- UPDATE Orders
    +-- EXEC CreateOrder
```

Then:

```text
app_sales
     |
     ↓
SalesAppRole
```

This is easier to manage than manually assigning dozens of permissions to every user.

---

# 13. Production security architecture

Our ABC Retail architecture can now look like:

```text
                  Application
                       |
                       v
                Private Network
                       |
                       v
               SQL Managed Instance
                       |
              +--------+--------+
              |                 |
              v                 v
        Authentication     Authorization
              |                 |
        Entra / SQL        Roles / Permissions
                                |
                                v
                           SalesDB
                                |
                       +--------+--------+
                       |                 |
                       v                 v
                    Orders          Customers
```

---

# 14. Important DBA troubleshooting sequence

When an application can't access SQL MI, use this sequence:

```text
1. DNS
   ↓
2. Network
   ↓
3. Endpoint / port
   ↓
4. Authentication
   ↓
5. Login
   ↓
6. Database user
   ↓
7. Role membership
   ↓
8. Object permission
   ↓
9. Query/application behavior
```

Don't skip straight to step 8.

---

# 15. Production scenario

At 2 PM, developers deploy a new version.

Immediately:

```text
Application
     ↓
SQL MI
     ↓
Permission denied
```

Before deployment:

```text
app_sales
    ↓
SalesDB
    ↓
SELECT Orders
```

After deployment:

```text
Application
    ↓
New stored procedure
    ↓
EXECUTE permission missing
```

The database isn't down.

The network isn't broken.

Authentication works.

The problem is:

> **Authorization.**

You investigate:

```sql
SELECT *
FROM sys.database_principals;
```

and:

```sql
SELECT *
FROM sys.database_permissions;
```

Then check role membership and the stored procedure permissions.

---

# 16. Another real-world problem — orphaned user

Imagine a login is recreated:

```text
Old Login
   ↓
Old SID

New Login
   ↓
Different SID
```

The database user may no longer map correctly to the login.

Conceptually:

```text
Login SID
    X
Database User SID
```

You can end up with an **orphaned user**.

This is particularly important during migrations because you need to preserve and validate security principals correctly.

---

# 17. Security checklist for migration

Before production cutover, verify:

```text
☐ Logins migrated
☐ Users migrated
☐ SID mappings correct
☐ Entra identities configured
☐ Roles configured
☐ Permissions validated
☐ Application identity tested
☐ Least privilege reviewed
☐ Secrets/credentials handled securely
☐ Auditing/security monitoring configured
```

---

# 18. Interview questions

### Question 1

**What is the difference between authentication and authorization?**

Answer:

> Authentication verifies identity. Authorization determines what that authenticated identity is allowed to access or execute.

---

### Question 2

**Login vs user?**

Answer:

> A login is associated with the SQL Server/Managed Instance level, while a user is associated with a specific database.

---

### Question 3

**Application gets "Login failed." What do you check?**

Think:

```text
Network
 ↓
Authentication
 ↓
Login
 ↓
Credentials
 ↓
Authentication method
```

---

### Question 4

**Application gets "SELECT permission denied."**

Think:

```text
Authentication succeeded
        ↓
Authorization failed
        ↓
User / Role / Permission
```

---

# Your Lesson 4 challenge 🧑‍💻

ABC Retail reports this error:

```text
Login failed for user 'app_sales'
```

You verify:

```text
DNS:              SUCCESS
Network:          SUCCESS
MI:               HEALTHY
Database:         ONLINE
Login:            EXISTS
Application:      Using correct username
```

But the application still cannot log in.

### What would you investigate next?

Pick the **most likely areas**:

**A.** SQL query execution plan
**B.** Password/credential or authentication configuration
**C.** Index fragmentation
**D.** Storage IOPS
**E.** Whether the login is disabled or authentication method is incorrect

You can choose more than one.
