# Create Custom Config Parameters in Postgres

In some case, you need to a customized parameters in config file which is **postgresql.conf**.

There are three steps to create a customized parameter.

## 1. Add Config Parameter Pattern 

**config_int** is an array containing int type parameters.

Here we add a parameter **query_order** which is the order of query to be run.

PGC_USERSET parameters could be modified without restarting the system.

```c
src/backend/utils/misc/guc.c

 static struct config_int ConfigureNamesInt[] =
{
    {
        {"query_order",PGC_USERSET, RESOURCES_ASYNCHRONOUS,
            gettext_noop("The order of query to be run."),
            NULL,
            1
        },
        &query_order,
        0, 0, 10000,
        NULL, NULL, NULL
    }
 }
```

## 2. Global Extern and Initialization

```c
src/backend/utils/init/globals.c
    
int         query_order = 0;
```

```c
src/include/miscadmin.h
extern PGDLLIMPORT int query_order;
```

## 3. Set Value in Config file

```bash
postgresql.conf

query_order = 1
```

