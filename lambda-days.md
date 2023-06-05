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
### Functional Pipelines

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

1. Why I'm here?
2. How aggregation functions works
3. How Timescale implemented the Function Pipelines
4. Examples and composition

# Why I'm here?

* FP is really cool
* SQL gets complicated really fast

# Volatility example


```sql
SELECT city_name, SUM(abs_delta) AS volatility
FROM (
  SELECT city_name,
    ABS(
      temp_c - LAG(temp_c) OVER (
        PARTITION BY city_name ORDER BY time)
    ) AS abs_delta
  FROM weather_metrics
) AS calc_delta
GROUP BY city_name
```

# Volatility with pipelines

```sql
SELECT city_name,
  ( timevector(time,temp_c::numeric)
    ->sort()
    ->delta()
    ->sum()) as volatility
FROM weather_metrics
GROUP BY 1 ;
```

# Timevector

```sql
SELECT timevector(now(), 1);
```

# Mapping

```sql
SELECT timevector(now(), 1)->map('$value+1');
```

# Unnesting

```sql
SELECT timevector(now(), 1)
  ->map('$value+1')
  ->unnest();
```

# Expand `(...).*`

```sql
SELECT (
  timevector(now(), 1)
  ->map('$value+1')
  ->unnest()
).*;
```

# Inline mapping

```sql
SELECT (
  timevector(now(), 10.0)
  ->map('$value/3.0')
  ->unnest()
).*;
```

# Pre-built functions

```sql
SELECT (
  timevector(now(), 10.0)
  ->map('$value/3.0')
  ->map('round($value)')
  ->unnest()
).*;
```

# Intervals

```sql
SELECT (
  timevector(now(), 1)
  ->map($$($time + '1 year'i, $value * 2)$$)
  -> unnest()
).*;
```

# Filtering

```sql
SELECT (
   timevector(now(), i)
     ->map('$value+1')
     ->filter('$value > 2')
     ->unnest()
).*
FROM generate_series(1,3) i;
```

# Customize

```sql
CREATE FUNCTION tripple(double precision)
RETURNS double precision AS $$
  SELECT $1 * 3;
$$ LANGUAGE SQL;
```

Usage:

```sql
SELECT (
  timevector(now(), 1)
  ->map('$value+1')
  ->map_data('tripple')
  ->unnest()
).*;
```

# Data types

```sql
SELECT pg_typeof(timevector(now(), 1));
```

# Functions matching PG Type

```sql
SELECT n.nspname as "Schema",
  p.proname as "Name",
  pg_catalog.pg_get_function_result(p.oid) as "Result data type",
  pg_catalog.pg_get_function_arguments(p.oid) as "Argument data types",
 CASE p.prokind
  WHEN 'a' THEN 'agg'
  WHEN 'w' THEN 'window'
  WHEN 'p' THEN 'proc'
  ELSE 'func'
 END as "Type"
FROM pg_catalog.pg_proc p
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = p.pronamespace
WHERE pg_catalog.pg_function_is_visible(p.oid)
      AND n.nspname <> 'pg_catalog'
      AND n.nspname <> 'information_schema'
      AND pg_catalog.pg_get_function_arguments(p.oid) ~ 'timevector_tstz_f64'
ORDER BY 1, 2, 4;
```

# How aggregates works?

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

# Moving Average Final function

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
$$ LANGUAGE sql;
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
SELECT id, value,
  movavg(value, 3)
    OVER (
      ORDER BY id
      ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg
FROM (
  SELECT
    generate_series(1, 10) AS id,
    generate_series(1, 10)::numeric AS value
) AS data;
```

# Transition Avg

<video autoplay loop muted playsinline>
<source src="https://s3.amazonaws.com/blog.timescale.com/gifs/how-postgres-works/how_postgres_works_2.mp4" type="video/mp4">
</video>

# Final Func Avg

<video autoplay loop muted playsinline>
<source id="player" src="https://s3.amazonaws.com/blog.timescale.com/gifs/how-postgres-works/how_postgres_works_3.mp4" type="video/mp4">
</video>

# Learn More

> How PostgreSQL Aggregation Works and
> How It Inspired Our Hyperfunctions’ Design

[Link to the blog post](https://www.timescale.com/blog/how-postgresql-aggregation-works-and-how-it-inspired-our-hyperfunctions-design-2/).


                █████████████████████████████████████████████████
                █████████████████████████████████████████████████
                ████ ▄▄▄▄▄ █▀ █ ▄█▄▀ █ ▄ ▄▄▄█ ▀▀▄█▄▄ █ ▄▄▄▄▄ ████
                ████ █   █ █▀▀▄███▄▄▄█▀▄▄▄ ▄▄▀▀▄▄▀█▄▀█ █   █ ████
                ████ █▄▄▄█ █▀ ▀▄▄▀ ▄▄▀█▀▄ ▀▄█ ▀▀ █▀▄ █ █▄▄▄█ ████
                ████▄▄▄▄▄▄▄█▄█▄▀ █ █ █▄▀ ▀▄█▄█ ▀ █▄▀ █▄▄▄▄▄▄▄████
                ████ ▄▄ ▄█▄ ▄▀▀▄█ ▄██ █ ▀█▄▀█▄▄▄▀▄▀ ▀▄▀ ▀ █▄▀████
                ████▄ █▀▀▄▄ ▀▀▀▀▀ ▄▄▀ ▀█▄ ▀▀█▄▀▄█▄ ▀▄▄▀ ▀▀██▀████
                ██████ ██▄▄▀ ▄▄█▀▀▄▀▄█▀█▄▀▄ ▄▄ ▄   ▀▀█▀ ▀ ▄█ ████
                ████  █▀▀▄▄█   █▀▄  ▄▄ █▄▀ █▄▄ ▄▀█▀█ █▀▀█▄▄▀█████
                ████▀▄ █▀▀▄█▄  ▀▄▄ ▄▀▄▀▀▀▀ ▀▄▄█▄▀ ▀   ▀ ▀█▄▀▀████
                ████▄ █▀▀▄▄ █  ▀▄█▀▄ █▄█ ▀ █ █▄▄ ███  █ ▀▀█▀█████
                ██████▀▀▄█▄ ▄ ▀▄▄█▀▀▀█▄▄▀▀▄▀▄ ▄█▀▄▀▀█▀▀▀▀  ▄ ████
                ████▄█▄█▄▀▄▀▀▄█ ▄▄██▀██ ▀▀███▄ ▄████▀▀ ▄▄ █▀█████
                ████▀▀▄▄▄ ▄▀▀ ▄ ▄ ▄█▀ ▀ ██▄ █ ▄▄█  ▀▀█▀ ▀█▄  ████
                ████▄  ▄▀▀▄██▄██▀ ▄ ▄ ▀█▀▀▄███▀  █ █ ▄▄ █▄▄██████
                ████ █▄ █▄▄ █ ▀█▀▀▄▀▄█▀▄█▀█   ▄█  █ ▀█▀▀▀▄ ▀ ████
                ████ █▀█▀ ▄ ▀█▄█▀▄▀ ▄▄█▀   ▀▀██  ▄▀█ ▀█ ▄ █▀█████
                ████▄█▄▄▄▄▄█ ▄█▀▄▄ ▄▀▄▀ ▀█ ▀  ▄▄▀ ▀█ ▄▄▄ ▄  ▀████
                ████ ▄▄▄▄▄ █▄█▀███▀ ▀█▄█▄  ▀█▄ ▄ █▄  █▄█ █▄ █████
                ████ █   █ █  ▀▄▄█▀ ▀█▄▀█▀▀▀█ ▄█▀▄█▀▄▄▄▄▄▀▄█▀████
                ████ █▄▄▄█ █ ▀▀ ▄▄▀█▀█▄▀█▀▀▀▄▄  █▄▄█  ▄▀▄██▀█████
                ████▄▄▄▄▄▄▄█▄██▄▄▄██▄▄██▄█▄▄█▄▄▄█▄█▄█▄▄▄██▄██████
                █████████████████████████████████████████████████
                █████████████████████████████████████████████████


# Toolkit

We're going to explore the `toolkit_experimental` functions 😎

```sql
ALTER ROLE current_user
in DATABASE playground
set search_path =
  toolkit_experimental,
  public;
```

# Help us!

We want to make it less experimental, so, if you have any feedback, please, join
the toolkit and help us to promote experimental features with your feedback.

* Toolkit is written in Rust 🦀
* https://docs.timescale.com/api/latest/hyperfunctions/

> https://github.com/timescale/timescaledb-toolkit


# Series

```sql
CREATE VIEW series AS
SELECT timevector(time, value)
FROM toolkit_experimental
  .generate_periodic_normal_series(
    '2020-01-01 UTC'::timestamptz,
    rng_seed => 11111);
```

# Unnest

```sql
select unnest(timevector) from series limit 1;
```

# Unnest ().*

```sql
SELECT (timevector->unnest()).*
FROM series
LIMIT 3;
```


# Sum

```sql
select (timevector->sort()->delta())->sum() from series limit 3;
```

# Lttb

```sql
select (timevector->sort()->lttb(4)->unnest()).* from series ;
```

# Format

```sql
SELECT timevector(now(), 2);
```

Not all components are compatible with pipeline functions:

```sql
select to_text(
  timevector(now(), 2), 
  '{
     x: {{ TIMES | json_encode() | safe  }},
     y: {{ VALUES | json_encode() | safe }}
    }::json');
```

# Combine series

```sql
select to_text(
  timevector(now(), i),
  '{
     x: {{ TIMES | json_encode() | safe  }},
     y: {{ VALUES | json_encode() | safe }}
   }::json') from generate_series(1,3) i;
```

# Plotting

All data:

```sql
with pair as (
  select (timevector->unnest()).* from series)
  select time as x, value as y from pair ;
```

# Downsampling

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
