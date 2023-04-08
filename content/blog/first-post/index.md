---
title: Merging conflicts between cql service and cql schema
date: "2023-04-08T14:12:03.284Z"
description: "What to do when changing nested types in Cassandra db"
---



# Learning from errors: Apache Cassandra 

## I broke the cql service with a small change

We all need to work on something unfamiliar as a developer, and devils are hidden in details.

All I did was adding a couple of new fields in a cql type defined in cql schema, but I did not know this broke the service until customers called in to reflect the situation.

## But why?

We can break down and understand what is a type first

### types
Cassandra supports many data types. The concept is similar to SQL but of course they are different data types

[Cassandra Data Type](https://cassandra.apache.org/doc/latest/cassandra/cql/types.html) This is a quick introduction about Cassandra data types.

### How cql service and schema work together
The following is from chatGPT:

 > In Cassandra, a schema is a collection of tables that belong to a keyspace. A keyspace is like a database in traditional SQL databases, and it is a logical grouping of tables. Each table in Cassandra is defined by a schema that specifies the table's columns and data types. When creating a table in Cassandra, you first specify the keyspace to which the table belongs. Then you define the table schema, which includes the column names and data types, as well as any primary key or secondary indexes.


### User-Defined Types (UDTs)

The data type I modified is a user-defined type (UDT). I followed the basic instruction and added new fields in the type it belongs to.

    > ALTER field_name TYPE new_cql_datatype
        | ADD (field_name cql_datatype[,...])

The above example is from [DataStax Documentation](https://docs.datastax.com/en/cql-oss/3.3/cql/cql_reference/cqlAlterType.html)

ChatGPT has this example:
    >ALTER TYPE person ADD email text;

### The error
After I deployed the schema to prod, the service started to have the error: UDT type <type name> cannot be converted to another UDT type <type name>

I was confused because the UDT type name was the same. After investigating, this UDT type was the higher level of the UDT type I modified.


### Potential Solutions

One possible solution is to deploy both service and schema at the same time, but it is not ideal especially in prod environment. This error can still be thrown because we can't promise service and schema would finish deployment at the same time

The other solution can gracefully avoid the issue. It is again organized by chatGPT because it is a better writer in English than I am.

1. Create a new version of the UDT with the updated fields, keeping the original UDT intact

It means doing something like this in schema (parts of the codes are from chatGPT too):

    ```
    CREATE TYPE person_v2 (<old fields and new fields>)

    CREATE TYPE users_v2 (person frozen<person_v2>);

    ``  

2. Create a new table that uses the new version of UDT. 

In my scenario, I need to create a new higher-level UDT that uses this UDT, and make sure I update **ALL** tables that use these UDTs

    like this:

    ```
    ALTER TABLE mytable ALTER mycolumn TYPE frozen<users_v2>;
    ``

3. Update the service code to use the **new version** of the UDT. Make sure the service can handle both the old and new versions of the UDT during the rollout period

Then on service, we just created new objects for these newly-added items. For example:
    
    ```
    @UDT(keyspace = "my_keyspace", name = "person_v2")
        public class PersonV2 {
        private String name;
        private int age;
        private String email;
        // getters and setters
        }

     /// and users_v2   
    ``


4. Migrate the data from the old table to the new table, making sure to convert the UDT values to the new version as needed. This can be done in a rolling fashion to minimize downtime

I am not sure if I actually need this step. The newly-added fields are nullable, so I just need to make sure the service can tolerate both old and new types

5. Once all the data has been migrated, update the service code to use the new table only

I probably should do so and monitor it

6. If everything goes well, after a sufficient period of time, we can drop the old UDT and table in schema and monitor both to make sure it works fine


Because we needed to fix the issue as soon as possible, we just deployed the updated service that had the new UDT to prod. I did not try the graceful solution, so it might not be 100% correct. However, it is a good lesson to remind us to be more careful when updating a database service and schema.