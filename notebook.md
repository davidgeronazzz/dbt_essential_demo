# Quickstart per dbt e Snowflake

## Introduzione  
In questa guida rapida imparerai come usare dbt con Snowflake. Ti mostreremo come:  
- Creare un nuovo worksheet in Snowflake.  
- Caricare dati di esempio nel tuo account Snowflake.  
- Collegare dbt a Snowflake.  
- Prendere una query di esempio e trasformarla in un **modello** in dbt (in dbt un modello è semplicemente una `SELECT`).  
- Aggiungere **sources** al tuo progetto dbt. I sources ti permettono di dare un nome e una descrizione ai dati grezzi già caricati in Snowflake.  
- Aggiungere **test** ai tuoi modelli.  
- Documentare i tuoi modelli.  
- Programmare un job per l’esecuzione.  
Snowflake offre anche un quickstart separato con un dataset pubblico differente. Per maggiori dettagli vedi la documentazione Snowflake.

### Prerequisiti  
- Devi avere un account dbt.  
- Devi avere un account trial di Snowflake. Durante la creazione del trial assicurati di scegliere l’**Enterprise edition** in modo da avere accesso come `ACCOUNTADMIN`.  
- Per una implementazione completa, considera le questioni organizzative quando scegli il provider cloud. Per scopi di questa configurazione, tutti i cloud provider e le region vanno bene — scegli quello che vuoi.

## Crea un nuovo worksheet in Snowflake  
1. Accedi al tuo account trial Snowflake.  
2. Nell’interfaccia Snowflake clicca su **+ Create** nell’angolo sinistro sotto il logo Snowflake. Dal menu a tendina scegli **SQL Worksheet**.

## Carica i dati  
I dati usati qui sono memorizzati come file CSV in un bucket pubblico su S3. I passaggi seguenti ti guidano su come preparare il tuo account Snowflake per questi dati e caricarli.

1. Crea un nuovo virtual warehouse, due nuovi database (uno per i dati grezzi “raw”, l’altro per lo sviluppo futuro con dbt), e due nuovi schemi (uno per i dati `jaffle_shop`, l’altro per `stripe`). Esegui questi comandi SQL nella Editor del worksheet e clicca “Run”:  
   ```sql
   create warehouse transforming;
   create database raw;
   create database analytics;
   create schema raw.jaffle_shop;
   create schema raw.stripe;
   ```
2. Nel database `raw` e negli schemi `jaffle_shop` e `stripe` crea tre tabelle e carica i dati rilevanti in esse:  
   - Pulisci l’Editor, poi esegui questo comando per creare la tabella `customers`:  
     ```sql
     create table raw.jaffle_shop.customers
     (
       id integer,
       first_name varchar,
       last_name varchar
     );
     ```
   - Poi carica i dati nella tabella `customers`:  
     ```sql
     copy into raw.jaffle_shop.customers (id, first_name, last_name)
     from 's3://dbt-tutorial-public/jaffle_shop_customers.csv'
     file_format = (
         type = 'CSV',
         field_delimiter = ',',
         skip_header = 1
     );
     ```
   - Analogamente, crea la tabella `orders` in `raw.jaffle_shop`:  
     ```sql
     create table raw.jaffle_shop.orders
     (
       id integer,
       user_id integer,
       order_date date,
       status varchar,
       _etl_loaded_at timestamp default current_timestamp
     );
     ```
   - Carica i dati in `orders`:  
     ```sql
     copy into raw.jaffle_shop.orders (id, user_id, order_date, status)
     from 's3://dbt-tutorial-public/jaffle_shop_orders.csv'
     file_format = (
         type = 'CSV',
         field_delimiter = ',',
         skip_header = 1
     );
     ```
   - Poi crea la tabella `payment` in `raw.stripe`:  
     ```sql
     create table raw.stripe.payment
     (
       id integer,
       orderid integer,
       paymentmethod varchar,
       status varchar,
       amount integer,
       created date,
       _batched_at timestamp default current_timestamp
     );
     ```
   - Carica i dati in `payment`:  
     ```sql
     copy into raw.stripe.payment (id, orderid, paymentmethod, status, amount, created)
     from 's3://dbt-tutorial-public/stripe_payments.csv'
     file_format = (
         type = 'CSV',
         field_delimiter = ',',
         skip_header = 1
     );
     ```
3. Verifica che i dati siano caricati eseguendo queste query di controllo:  
   ```sql
   select * from raw.jaffle_shop.customers;
   select * from raw.jaffle_shop.orders;
   select * from raw.stripe.payment;
   ```
