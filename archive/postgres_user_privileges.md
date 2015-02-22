
HOWTO: Set up read and read/write users on PostgreSQL database
==============================================================

As we were writing scripts to access our database for various purposes,
we wanted to set up several users so that we didn't inadvertently alter
important data.  Our solution was to set up, in addition to the already
existing admin account, one user with read privileges, and one user with
read/write privileges (but not the ability to delete tables).  This is
the process for doing that thing:

Step 1: Create your users:
--------------------------

Log into your postgres database as admin, and:

```
    [mydb]=> create user [readuser] with password "[read's password]"
    [mydb]=> create user [readwriteuser] with password "[readwrite's password]"
```

Step 2: Give your users approrpiate access to the current tables:
-----------------------------------------------------------------

For read:

```
    [mydb]=> grant connect on database [mydb] to [readuser];
    [mydb]=> grant usage on schema public to [readuser];
    [mydb]=> grant select on all tables in schema public to [readuser];
```

Note that the default schema for your tables is schema "public".  If you have created different schemas, you'll need to grant access to those.

For read/write:

```
    [mydb]=> grant connect on database [mydb] to [readwriteuser];
    [mydb]=> grant usage on schema public to [readwriteuser];
    [mydb]=> grant all privileges on database [mydb] to [readwriteuser];
```

This grants read/write privileges, but not the ability to delete tables.  Only the admin can do that.

Step 3: Give your users appropriate access to future tables:
------------------------------------------------------------

This way, when somebody creates a new table, our read user will be able to read from it.

For read:

```
    [mydb]=> alter default privileges in schema public grant select on tables to [readuser];
```

For read/write:

```
    [mydb]=> alter default privileges in schema public grant all on tables to [readwriteuser];
```

