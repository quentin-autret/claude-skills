# Security: privileges, roles, row-level security, functions, connections

## Contents
- Privileges
- Roles
- Row-level security
- SECURITY DEFINER functions
- Passwords
- TLS and connection parameters
- Old patterns

## Privileges

- PUBLIC gets by default: CONNECT and TEMPORARY on databases, EXECUTE on functions and procedures, USAGE on types and languages. Nothing on tables, sequences or schemas. https://www.postgresql.org/docs/18/ddl-priv.html
- Granting on a table does not extend to the sequences it uses, including those of `serial` columns. Grant `USAGE` on such a sequence for `nextval`. https://www.postgresql.org/docs/18/sql-grant.html https://www.postgresql.org/docs/18/ddl-priv.html
- Revoking one column after a table-level grant does nothing: the table grant still covers it. Grant columns one by one instead: https://www.postgresql.org/docs/18/sql-grant.html
  ```sql
  GRANT SELECT (col1), UPDATE (col1) ON mytable TO miriam_rw;
  ```
- UPDATE and DELETE also need SELECT when they reference existing column values (for example in a WHERE). `SELECT ... FOR UPDATE` needs UPDATE on a column. https://www.postgresql.org/docs/18/ddl-priv.html
- Grant REFERENCES and TRIGGER with great care: a foreign key can call an arbitrary function with the table owner's privileges, and triggers run with the privileges of the users who modify the table. https://www.postgresql.org/docs/18/ddl-priv.html
- `ALL FUNCTIONS IN SCHEMA` does not cover procedures: use `ALL ROUTINES`. https://www.postgresql.org/docs/18/sql-grant.html https://www.postgresql.org/docs/18/ddl-priv.html
- Revoke in the same transaction that creates the object to avoid an exposure window. https://www.postgresql.org/docs/18/ddl-priv.html
- `MAINTAIN` on a table allows VACUUM, ANALYZE, REINDEX, CLUSTER, REFRESH MATERIALIZED VIEW and LOCK TABLE. The `pg_maintain` role grants it on every relation. https://www.postgresql.org/docs/18/ddl-priv.html https://www.postgresql.org/docs/18/predefined-roles.html
- Revoking USAGE on a schema does not hide object names, which stay visible in the catalogs. https://www.postgresql.org/docs/18/ddl-priv.html

## Roles

- Work as a non-superuser role. Manage privileges through group roles without LOGIN. https://www.postgresql.org/docs/18/role-attributes.html https://www.postgresql.org/docs/18/ddl-priv.html
- Roles are cluster-wide. LOGIN, SUPERUSER, CREATEDB and CREATEROLE are never inherited: `SET ROLE` is needed to use them. https://www.postgresql.org/docs/18/role-membership.html
- `GRANT island TO joe WITH INHERIT TRUE, SET FALSE` gives the privileges without allowing `SET ROLE island`. https://www.postgresql.org/docs/18/role-membership.html
- A non-superuser CREATEROLE role manages only the roles it has ADMIN OPTION on, and cannot grant SUPERUSER, REPLICATION or BYPASSRLS. https://www.postgresql.org/docs/18/role-attributes.html
- `pg_read_all_data` and `pg_write_all_data` do not bypass row-level security. `pg_read_server_files`, `pg_write_server_files` and `pg_execute_server_program` can lead to superuser access. https://www.postgresql.org/docs/18/predefined-roles.html

## Row-level security

- Policies do nothing until `ALTER TABLE t ENABLE ROW LEVEL SECURITY`. Enabled with no policy means deny all. https://www.postgresql.org/docs/18/ddl-rowsecurity.html
- The table owner bypasses policies unless `FORCE ROW LEVEL SECURITY`. Superusers and BYPASSRLS roles always bypass. TRUNCATE and REFERENCES ignore RLS. https://www.postgresql.org/docs/18/ddl-rowsecurity.html
  ```sql
  ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
  CREATE POLICY account_managers ON accounts TO managers
      USING (manager = current_user);
  ```
- `USING` filters existing rows silently. `WITH CHECK` rejects new rows with an error. For ALL and UPDATE without WITH CHECK, USING serves both. https://www.postgresql.org/docs/18/sql-createpolicy.html
- Permissive policies combine with OR, restrictive ones with AND. Restrictive policies alone allow nothing. https://www.postgresql.org/docs/18/sql-createpolicy.html
- UPDATE and DELETE that read columns (in a WHERE or RETURNING clause) must pass the SELECT policies too. RETURNING rows failing SELECT policies raise an error. https://www.postgresql.org/docs/18/sql-createpolicy.html
- Policy expressions run with the rights of the querying user: referenced tables and functions must be accessible to that user, or the query fails with permission denied. https://www.postgresql.org/docs/18/sql-createpolicy.html
- Unique and foreign key checks bypass RLS and can reveal hidden values. Generated values (surrogate keys) are preferable to keys with external meaning. https://www.postgresql.org/docs/18/ddl-rowsecurity.html https://www.postgresql.org/docs/18/sql-createpolicy.html
- Policies reading other tables can leak under concurrent updates. Policies on the row's own columns are the simplest and fastest. https://www.postgresql.org/docs/18/ddl-rowsecurity.html
- Test with `SET ROLE`. A USING rejection shows `UPDATE 0`, a WITH CHECK failure an error. `row_security = off` makes filtered queries fail instead (useful for backups). https://www.postgresql.org/docs/18/ddl-rowsecurity.html

## SECURITY DEFINER functions

- Pin `search_path` to trusted schemas with `pg_temp` last, revoke EXECUTE from PUBLIC and grant it, all in one transaction: https://www.postgresql.org/docs/18/sql-createfunction.html
  ```sql
  BEGIN;
  CREATE FUNCTION check_password(uname TEXT, pass TEXT) RETURNS BOOLEAN
      AS $$ ... $$ LANGUAGE plpgsql
      SECURITY DEFINER
      SET search_path = admin, pg_temp;
  REVOKE ALL ON FUNCTION check_password(uname TEXT, pass TEXT) FROM PUBLIC;
  GRANT EXECUTE ON FUNCTION check_password(uname TEXT, pass TEXT) TO admins;
  COMMIT;
  ```
- Functions, triggers and row-level security policies let users insert code that other users might execute unintentionally ("Trojan horse"). Control who can define objects and remove schemas that let untrusted users create objects from `search_path`. https://www.postgresql.org/docs/18/perm-functions.html
- `LEAKPROOF` needs superuser. A function whose errors reveal argument values is not leakproof. https://www.postgresql.org/docs/18/sql-createfunction.html

## Passwords

- Use `scram-sha-256` (default `password_encryption`). MD5 passwords are deprecated: PostgreSQL 18 warns when one is set. https://www.postgresql.org/docs/18/auth-password.html https://www.postgresql.org/docs/18/runtime-config-connection.html
- Migrating from md5: check client support, set `password_encryption = 'scram-sha-256'`, have every user reset the password, then switch `pg_hba.conf` to `scram-sha-256`. https://www.postgresql.org/docs/18/auth-password.html

## TLS and connection parameters

- Default `sslmode=prefer` offers no man-in-the-middle protection and is not recommended in secure deployments. Use `sslmode=verify-full` with a root certificate, or `sslrootcert=system` (which makes verify-full the default sslmode). https://www.postgresql.org/docs/18/libpq-ssl.html https://www.postgresql.org/docs/18/libpq-connect.html
- By default libpq accepts any authentication method and the server may skip authentication. `require_auth=scram-sha-256` makes the connection fail otherwise. `channel_binding=require` forces SCRAM channel binding. https://www.postgresql.org/docs/18/libpq-connect.html
- With untrusted users and no secure schema pattern, start each session with an empty `search_path` (`options=-csearch_path=`). https://www.postgresql.org/docs/18/libpq-connect.html
- Several hosts are tried in order, but an authentication failure stops the list. `target_session_attrs=read-write` accepts only a session that accepts read-write transactions by default (`primary` requires a server not in hot standby). https://www.postgresql.org/docs/18/libpq-connect.html
  ```
  postgresql://host1:123,host2:456/somedb?target_session_attrs=any&application_name=myapp
  ```
- PostgreSQL 18 adds OAuth (`require_auth=oauth`, `oauth_issuer`, `oauth_client_id`). The server loads token validators from `oauth_validator_libraries`, and none ship with PostgreSQL. https://www.postgresql.org/docs/18/release-18.html https://www.postgresql.org/docs/18/libpq-connect.html https://www.postgresql.org/docs/18/runtime-config-connection.html

## Old patterns

- md5 or `password` method in `pg_hba.conf` -> `scram-sha-256`. https://www.postgresql.org/docs/18/auth-password.html
- `requiressl=1` (equivalent to `sslmode=require`) or `ssl=true` (translated to `sslmode=require`) -> `sslmode=verify-full` for server verification. https://www.postgresql.org/docs/18/libpq-connect.html
- `LANGUAGE 'plpgsql'` (quoted, deprecated) -> `LANGUAGE plpgsql`. https://www.postgresql.org/docs/18/sql-createfunction.html
- `ssl_ecdh_curve` -> `ssl_groups`. https://www.postgresql.org/docs/18/runtime-config-connection.html
