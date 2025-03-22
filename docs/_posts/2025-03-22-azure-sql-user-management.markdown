---
layout: post
title:  "Azure SQL Databases User Management scripts"
date:   2025-03-22 10:10:24 +0100
categories: azure sql
---

Every so often I have a need to add an Entra ID user directly to an Azure SQL database, which involves a hunt on google

This command will also give the user read only access to the database.

```SQL
CREATE USER [bob.thebuilder@bobsville.co.uk] FROM EXTERNAL PROVIDER;
EXEC sp_addrolemember 'db_datareader' , N'bob.thebuilder@bobsville.co.uk'
```


Additionally, sometimes I need to look at which roles are assigned to which users:

```SQL
SELECT DP1.name AS DatabaseRoleName,   
    isnull (DP2.name, 'No members') AS DatabaseUserName   
FROM sys.database_role_members AS DRM  
RIGHT OUTER JOIN sys.database_principals AS DP1  
    ON DRM.role_principal_id = DP1.principal_id  
LEFT OUTER JOIN sys.database_principals AS DP2  
    ON DRM.member_principal_id = DP2.principal_id  
WHERE DP1.type = 'R'
ORDER BY DP1.name;  
```