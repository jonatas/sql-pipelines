# Functional Pipelines in SQL

```sql
SELECT
  'Lambda Days' as conference_name,
  2023 as edition,
  'Functional Pipelines in SQL' as talk_title,
  'Jônatas Davi Paganini' as author,
  'jonatas@timescale.com' as author_mail
```

# Welcome

### Timescaledb Toolkit
#### &
### Functinal Pipelines

#### **Jônatas Davi Paganini**
#### @jonatasdp

# @jonatasdp

    * Postgresql since 2004.

* Backend developer
* Ruby/Shell/Postgresql/Vim

    * Developer Advocate at Timescale

#### twitter: @jonatasdp
#### github: @jonatas

# Agenda

1. W** I'm doing here?
2. How aggregation functions works
3. How Timescale implemented the Function Pipelines
4. Examples and composition

# What?

> Why functional programming?

* Backend developer
* Ruby/Shell/Postgresql/Vim
* Postgresql since 2004.

# How Aggregation works

[How PostgreSQL Aggregation Works and How It Inspired Our Hyperfunctions’ Design](https://www.timescale.com/blog/how-postgresql-aggregation-works-and-how-it-inspired-our-hyperfunctions-design-2/).


# Custom Aggregate


```sql
CREATE AGGREGATE aggregate_function_name (input_type)
(
    sfunc = state_function,
    stype = state_type,
    finalfunc = final_function
);
```

# Delta state

```sql
CREATE FUNCTION delta_statefunc(numeric[], numeric)
    RETURNS numeric[] AS $$
    SELECT array_append($1, $2) ORDER BY 1
$$ LANGUAGE sql;
```


# Delta final

```sql
CREATE FUNCTION delta_finalfunc(numeric[])
    RETURNS numeric AS $$
    DECLARE
        prev_val numeric;
        curr_val numeric;
        result numeric := 0;
    BEGIN
        FOR i IN 1..array_length($1, 1) LOOP
            curr_val := $1[i];
            IF i > 1 THEN
                result := result + (curr_val - prev_val);
            END IF;
            prev_val := curr_val;
        END LOOP;
        RETURN result;
    END;
$$ LANGUAGE plpgsql;
```

# Delta aggregate


```sql
CREATE AGGREGATE delta(numeric) (
    sfunc = delta_statefunc,
    stype = numeric[],
    finalfunc = delta_finalfunc
);
```

# Example

```sql
CREATE TABLE sales (
   id SERIAL PRIMARY KEY,
   product_name VARCHAR(50),
   price DECIMAL(10,2)
);
```

# Insert

```sql
INSERT INTO sales (product_name, price) VALUES
   ('Product A', 10.50),
   ('Product B', 20.75),
   ('Product C', 15.25),
   ('Product D', 12.50),
   ('Product E', 30.00);
```

# Delta

```sql
SELECT
    product_name,
    price,
    delta(price) OVER (ORDER BY product_name) AS delta
FROM sales
ORDER BY id;
```

# Moving Average State function

```sql
CREATE FUNCTION movavg_statefunc(numeric[], numeric, integer)
    RETURNS numeric[] AS $$
    SELECT array_append($1, $2) ORDER BY array_length($1, 1) DESC
$$ LANGUAGE sql;
```

# Mov Avg Final
```sql
CREATE FUNCTION movavg_finalfunc(numeric[])
    RETURNS numeric AS $$
    DECLARE
        sum numeric := 0;
        count integer := 0;
    BEGIN
        FOR i IN 1..array_length($1, 1) LOOP
            IF i <= $1[2] THEN
                sum := sum + $1[i];
                count := count + 1;
            END IF;
        END LOOP;

        IF count = 0 THEN
            RETURN NULL;
        ELSE
            RETURN sum / count;
        END IF;
    END;
$$ LANGUAGE plpgsql;

```

# Mov Avg Aggregate

```sql
CREATE AGGREGATE movavg(numeric, integer) (
    sfunc = movavg_statefunc,
    stype = numeric[],
    finalfunc = movavg_finalfunc
);
```

# Mov Avg query

```sql
SELECT
    id,
    value,
    movavg(value, 3) OVER (ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM
    (SELECT generate_series(1, 10) AS id, generate_series(1, 10)::numeric AS value) AS data;
```

# Toolkit

# Unnest
```sql
select unnest(timevector) from series limit 1;
```

# Unnest ().*

```sql
select (timevector->unnest()).* from series limit 3;
```

# Delta

```sql
select (timevector->sort()->delta()->unnest()).* from series limit 3;
```

# Sum

```sql
select (timevector->sort()->delta())->sum() from series limit 3;
```

# Lttb

```sql
select (timevector->sort()->lttb(4)->unnest()).* from series ;
```

# Plotting


all data:

```sql
with pair as (
  select (timevector->unnest()).* from series)
  select time as x, value as y from pair ;
```

# downsampling

```sql
with pair as (
  select (
    timevector
    ->toolkit_experimental.lttb(200)
    ->unnest()
  ).* from series
) select time as x, value as y from pair ;
```


# Extra Resources

- https://github.com/timescale/timescaledb
- https://github.com/timescale/timescaledb-toolkit
- https://timescale.com/community
- https://docs.timescale.com/
- https://blog.timescale.com/

[How PostgreSQL Aggregation Works and How It Inspired Our Hyperfunctions’ Design](https://www.timescale.com/blog/how-postgresql-aggregation-works-and-how-it-inspired-our-hyperfunctions-design-2/).

# Thanks

- [@jonatasdp](https://twitter.com/jonatasdp) on {Twitter,Linkedin}
- Github: [@jonatas](https://github.com/jonatas)

#### Jônatas Davi Paganini
