# Today's notebook
## Accounts & Setup
- [ ] Create a dbt account. (link)
- [ ] Create a Snowflake trial account (link).
- [ ] Load data into snowflake
- *New worksheet*:

    ```sql
    -- initial setup
    use role sysadmin;
    create database dbt_db;
    create database raw;
    create schema raw.jaffle_shop;
    create schema raw.stripe;
    create warehouse transforming;

    -- customer table --
    create table raw.jaffle_shop.customers 
    ( id integer,
    first_name varchar,
    last_name varchar
    );
    -- load customer data
    copy into raw.jaffle_shop.customers (id, first_name, last_name)
    from 's3://dbt-tutorial-public/jaffle_shop_customers.csv'
    file_format = (
        type = 'CSV'
        field_delimiter = ','
        skip_header = 1
        );

    -- orders table --
    create table raw.jaffle_shop.orders
    ( id integer,
    user_id integer,
    order_date date,
    status varchar,
    _etl_loaded_at timestamp default current_timestamp
    );
    -- load orders data --
    copy into raw.jaffle_shop.orders (id, user_id, order_date, status)
    from 's3://dbt-tutorial-public/jaffle_shop_orders.csv'
    file_format = (
        type = 'CSV'
        field_delimiter = ','
        skip_header = 1
        );

    -- payments table -- 
    create table raw.stripe.payment 
    ( id integer,
    orderid integer,
    paymentmethod varchar,
    status varchar,
    amount integer,
    created date,
    _batched_at timestamp default current_timestamp
    );
    -- load payments data -- 
    copy into raw.stripe.payment (id, orderid, paymentmethod, status, amount, created)
    from 's3://dbt-tutorial-public/stripe_payments.csv'
    file_format = (
        type = 'CSV'
        field_delimiter = ','
        skip_header = 1
        );
    
    -- checkout the tables
    select * from raw.jaffle_shop.customers;
    select * from raw.jaffle_shop.orders;
    select * from raw.stripe.payment;  
    ```
--- 

- [ ] Connect Snowflake & dbt.
- Using **Partner Connect** (Recomended):
    
    <img src="files/image.png" alt="image" width="280" border="1"/>  <img src="files/image-2.png" alt="image-2" width="350" border="1" /> <img src="files/image-1.png" alt="image-1" width="350" border="1"/>
    

- **Manual connection**:
    ```sql
    use role accountadmin;
    create role dbt_transformer;
    set name = (select current_user());
    grant role dbt_transformer to user identifier($name);

    grant all on database raw to role dbt_transformer;
    grant all on database dbt_db to role dbt_transformer;

    grant all on schema raw.jaffle_shop to role dbt_transformer;
    grant all on schema raw.stripe to role dbt_transformer;

    grant all on all tables in database raw to role dbt_transformer;
    grant all on future tables in database raw to role dbt_transformer;
    ```

    Go to dbt *account settings*, create a **new project**, which require a *development environment* and so a *connection*.
    1. create a new project.
    2. setup a new connection: chose **Snowflake**\
        Enter: ACCOUNT - ROLE (leave black) - DATABASE `dbt_db` - WAREHOUSE `transforming`.
    3. setup the development environment:\
        Chose **Username and Password**;\
        Enter: USERNAME - PASSWORD - ROLE `dbt_transformer` - SCHEMA `dbt_<first-initial><last-name>`
    4. test connection and save.
    5. Setup a **managed** repository, and give it a name.

        <img src="files/image-3.png" alt="image-3" width="800" border="1"/>
---

## Interface
<img src="files/image-4.png" alt="image-3" width="1000" border="1"/>

## Models
**What are Models?**

Models are *select statement* (sql files): 
- modular steps of logics between our raw layer and the trnasformed layer.
- Each model maps to a table/view in the data platform.

Look at the `models/example` folder:
- 2 models (sql file)
- 1 schema (yml file)

-> We can `compile` the code and `preview` the data.

In order to refer to other models we use the **`ref`** macro: `select * from {{ref('my_model')}}` 

---
Let's create a new model.\
Create a new file inside: `models/customers.sql`
```sql
with customers as (
    select
        id as customer_id,
        first_name,
        last_name
    from raw.jaffle_shop.customers

),

orders as (
    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status
    from raw.jaffle_shop.orders

),

customer_orders as (
    select
        customer_id,
        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders
    from orders
    group by 1

),

final as (
    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders
    from customers
    left join customer_orders using (customer_id)
)

select * from final;
```
#### How to create a model in the data warehouse?
The `dbt run` command.
- create a spcific model: `dbt run --select customers`
- create all models inside a folder: `dbt run --select example`
- create all models: `dbt run`

- [ ] Create the *customers* model.
- [ ] Take a look in Snowflake.
  
### Materialization (view or table?)
dbt, by default, always creates models as views. But we can declare the materialization of a model in different ways:\
at a project level, or at model level. The model level overwrites the project level.\
Look at the examples models:
- config blocks in the models
- config in the `example/schema.yml` file
- project config in the `dbt_project.yml` file

- [ ] Materiliaze the *customers* model as a table.
  
### Lineage
Now, lets create our staging models (1-1 with our source tables):
- [ ] Create `stg_jaffle_shop__customers.sql`
    ```sql
    with source as (
        select
            id as customer_id,
            first_name,
            last_name
        from raw.jaffle_shop.customers
    )

    select * from final
    ```
- [ ] create `stg_jaffle_shop__orders.sql`
    ```sql
    with source as (
        select
            id as order_id,
            user_id as customer_id,
            order_date,
            status
        from raw.jaffle_shop.orders
    )

    select * from final
    ```
- [ ] Use the `ref` macro to refer these in the *customer* model

### Sources
In dbt we configure our sources using `.yml` files.\
[YML files](https://docs.getdbt.com/best-practices/how-we-style/5-how-we-style-our-yaml) are human readable configuration files used to structure and document the models/objects inside the project.

- [ ] Add a customer and order source
- [ ] add a payments source.

### Project structure
```
jaffle_shop
├── README.md
├── analyses
├── seeds
│   └── employees.csv
├── dbt_project.yml
├── macros
│   └── cents_to_dollars.sql
├── models
│   ├── intermediate *(by area)*
│   │   └── finance
│   ├── marts *(by area)*
│   │   ├── finance
│   │   ├── marketing
│   │   └── etc.
│   ├── staging *(by source)*
│   │   ├── jaffle_shop
│   │   └── etc.
│   └── utilities
├── packages.yml
├── snapshots
└── tests
```
- [ ] Create the `staging`, `marts` and `intermediate` folders.
- [ ] Refactor the *customer model* into `marts/marketing/dim_customers.sql`
- [ ] Configure to materialize the *marts models* as tables and the *staging models* as views.
- [ ] Add a staging model for payments source.
- [ ] Create a `fct_orders.sql` into `marts/finance` folder.


### Source Freshness

