Scenario: 
Permissions needed for encrypt and decrypt symmetric key on column level encryption to a SQL Login or DB user?

**SQL Server Always Encrypted with Column-Level Encryption (CLE)** using **symmetric keys**, the required permissions depend on the encryption technology you're using. 

Here are the common scenarios.

### Scenario 1: SQL Server Column-Level Encryption (`EncryptByKey` / `DecryptByKey`)

For a user to encrypt and decrypt data using a symmetric key, they need:

1. **Permission on the symmetric key**

```sql
GRANT CONTROL ON SYMMETRIC KEY::YourSymmetricKey TO YourUser;
```

or, at minimum:

```sql
GRANT VIEW DEFINITION ON SYMMETRIC KEY::YourSymmetricKey TO YourUser;
```

2. **Permission on the certificate or asymmetric key** that protects the symmetric key (if the user must open it)

For a certificate:

```sql
GRANT CONTROL ON CERTIFICATE::YourCertificate TO YourUser;
```

3. **Permissions on the table/columns**

```sql
GRANT SELECT, INSERT, UPDATE ON dbo.YourTable TO YourUser;
```

4. The user must explicitly open the symmetric key before using it:

```sql
OPEN SYMMETRIC KEY YourSymmetricKey
DECRYPTION BY CERTIFICATE YourCertificate;

-- Encrypt
UPDATE dbo.YourTable
SET EncryptedColumn = EncryptByKey(Key_GUID('YourSymmetricKey'), 'Sensitive Data');

-- Decrypt
SELECT CONVERT(varchar(100),
       DecryptByKey(EncryptedColumn))
FROM dbo.YourTable;

CLOSE SYMMETRIC KEY YourSymmetricKey;
```

---

### Scenario 2: Always Encrypted

If you're using **Always Encrypted**, SQL Server never decrypts the data. The client driver performs encryption and decryption.

The user needs:

* Normal SQL permissions (`SELECT`, `INSERT`, `UPDATE`).
* Access to the **Column Master Key** in the external key store (Windows Certificate Store, Azure Key Vault, HSM, etc.).
* No SQL permission on the column encryption key for decryption itself.

---

### Scenario 3: Database Master Key → Certificate → Symmetric Key hierarchy

If using the traditional SQL Server encryption hierarchy:

```
Database Master Key
      ↓
Certificate
      ↓
Symmetric Key
      ↓
Encrypted Column
```

Typical grants are:

```sql
GRANT CONTROL ON CERTIFICATE::YourCertificate TO YourUser;
GRANT CONTROL ON SYMMETRIC KEY::YourSymmetricKey TO YourUser;

GRANT SELECT, INSERT, UPDATE ON dbo.YourTable TO YourUser;
```

---

## How to check what permissions are needed

If `OPEN SYMMETRIC KEY` fails, it's usually because the user lacks permission on the certificate or symmetric key.

You can inspect existing permissions:

```sql
SELECT *
FROM sys.database_permissions
WHERE major_id IN (
    SELECT symmetric_key_id FROM sys.symmetric_keys
    UNION
    SELECT certificate_id FROM sys.certificates
);
```

## Least-privilege recommendation

Avoid granting `CONTROL` unless necessary. A common approach is:

* `SELECT`, `INSERT`, `UPDATE` on the table.
* `VIEW DEFINITION` on the symmetric key.
* `VIEW DEFINITION` on the certificate.
* Encapsulate `OPEN SYMMETRIC KEY`, `EncryptByKey`, and `DecryptByKey` inside stored procedures and grant users only `EXECUTE` on those procedures. 
This limits direct access to encryption objects and is generally a more secure design.
