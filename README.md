# dbt-audit-helper

Useful macros when performing data audits

# Contents
* [compare_relations](#compare_relations-source)
* [compare_queries](#compare_queries-source)
* [compare_column_values](#compare_column_values-source)
* [compare_relation_columns](#compare_relation_columns-source)
* [compare_all_columns](#compare_all_columns-source)
* [compare_column_values_verbose](#compare_column_values_verbose-source)

# Installation instructions
New to dbt packages? Read more about them [here](https://docs.getdbt.com/docs/building-a-dbt-project/package-management/).
1. Include this package in your `packages.yml` file — check [here](https://hub.getdbt.com/dbt-labs/audit_helper/latest/) for the latest version number.
2. Run `dbt deps` to install the package.

# Macros
## compare_relations ([source](macros/compare_relations.sql))
This macro generates SQL that can be used to do a row-by-row validation of two
relations. It is largely based on the [equality](https://github.com/dbt-labs/dbt-utils#equality-source)
test in dbt-utils. By default, the generated query returns a summary of audit
results, like so:

| in_a  | in_b  | count | percent_of_total |
|-------|-------|------:|-----------------:|
| True  | True  | 6870  | 99.74            |
| True  | False | 9     | 0.13             |
| False | True  | 9     | 0.13             |

The generated SQL also contains commented-out SQL that you can use to check
the rows that do not match perfectly:

| order_id | order_date | status    | in_a  | in_b  |
|----------|------------|-----------|-------|-------|
| 1        | 2018-01-01 | completed | True  | False |
| 1        | 2018-01-01 | returned  | False | True  |
| 2        | 2018-01-02 | completed | True  | False |
| 2        | 2018-01-02 | returned  | False | True  |


This query is particularly useful when you want to check that a refactored model,
or a model that you are moving over from a legacy system, match up.

Usage:
The query is best used in dbt Develop so you can interactively check results
```sql
{# in dbt Develop #}

{% set old_etl_relation=adapter.get_relation(
      database=target.database,
      schema="old_etl_schema",
      identifier="fct_orders"
) -%}

{% set dbt_relation=ref('fct_orders') %}

{{ audit_helper.compare_relations(
    a_relation=old_etl_relation,
    b_relation=dbt_relation,
    exclude_columns=["loaded_at"],
    primary_key="order_id"
) }}


```
Arguments:
* `a_relation` and `b_relation`: The [relations](https://docs.getdbt.com/reference#relation)
  you want to compare.
* `exclude_columns` (optional): Any columns you wish to exclude from the
  validation.
* `primary_key` (optional): The primary key of the model. Used to sort unmatched
  results for row-by-row validation.

## compare_queries ([source](macros/compare_queries.sql))
Super similar to `compare_relations`, except it takes two select statements. This macro is useful when:
* You need to filter out records from one of the relations.
* You need to rename or recast some columns to get them to match up.
* You only want to compare a small number of columns, so it's easier write the columns you want to compare, rather than the columns you want to exclude.

```sql
{# in dbt Develop #}

{% set old_fct_orders_query %}
  select
    id as order_id,
    amount,
    customer_id
  from old_etl_schema.fct_orders
{% endset %}

{% set new_fct_orders_query %}
  select
    order_id,
    amount,
    customer_id
  from {{ ref('fct_orders') }}
{% endset %}

{{ audit_helper.compare_queries(
    a_query=old_fct_orders_query,
    b_query=new_fct_orders_query,
    primary_key="order_id"
) }}


```

## compare_column_values ([source](macros/compare_column_values.sql))
This macro will return a query, that, when executed, compares a column across
two queries, and summarizes how many records match perfectly (note: a primary
key is required to match values across the two queries).

| match_status                | count  | percent_of_total |
|-----------------------------|-------:|-----------------:|
| ✅: perfect match            | 37,721 | 79.03            |
| ✅: both are null            | 5,789  | 12.13            |
| 🤷: missing from b          | 25     | 0.05             |
| 🤷: value is null in a only | 59     | 0.12             |
| 🤷: value is null in b only | 73     | 0.15             |
| 🙅: ‍values do not match    | 4,064  | 8.51             |

This macro is useful when:
* You've used the `compare_queries` macro (above) and found that a significant
number of your records don't match.
* So now you want to find which column is causing most of these discrepancies.

### Usage:
```
{# in dbt Develop #}

{% set old_etl_relation_query %}
    select * from public.dim_product
    where is_latest
{% endset %}

{% set new_etl_relation_query %}
    select * from {{ ref('dim_product') }}
{% endset %}

{% set audit_query = audit_helper.compare_column_values(
    a_query=old_etl_relation_query,
    b_query=new_etl_relation_query,
    primary_key="product_id",
    column_to_compare="status"
) %}

{% set audit_results = run_query(audit_query) %}

{% if execute %}
{% do audit_results.print_table() %}
{% endif %}
```

**Usage notes:**
* `primary_key` must be a unique key in both tables, otherwise the join won't
work as expected.


### Advanced usage - dbt Cloud:
The ``.print_table()`` function is not compatible with dbt Cloud so an adjustment needs to be made in order to print the results. Replace the following section of code:
```
        {% set audit_results = run_query(audit_query) %}
        {% do audit_results.print_table() %}
        {{ log("", info=True) }}
```
with:
```
        {% set audit_results = run_query(audit_query) %}

        {% do log(audit_results.column_names, info=True) %}
        {% for row in audit_results.rows %}
            {% do log(row.values(), info=True) %}
        {% endfor %}
```


## compare_relation_columns ([source](macros/compare_relation_columns.sql))
This macro will return a query, that, when executed, compares the ordinal_position
and data_types of columns in two [Relations](https://docs.getdbt.com/docs/api-variable#section-relation).

| column_name | a_ordinal_position | b_ordinal_position | a_data_type       | b_data_type       |
|-------------|--------------------|--------------------|-------------------|-------------------|
| order_id    | 1                  | 1                  | integer           | integer           |
| customer_id | 2                  | 2                  | integer           | integer           |
| order_date  | 3                  | 3                  | timestamp         | date              |
| status      | 4                  | 5                  | character varying | character varying |
| amount      | 5                  | 4                  | bigint            | bigint            |


This is especially useful in two situations:
1. Comparing a new version of a relation with an old one, to make sure that the
structure is the same
2. Helping figure out why a `union` of two relations won't work (often because
the data types are different)

For example, in the above result set, we can see that `status` and `amount` have
switched order. Further, `order_date` is a timestamp in our "a" relation, whereas
it is a date in our "b" relation.

```sql
{#- in dbt Develop -#}

{% set old_etl_relation=adapter.get_relation(
      database=target.database,
      schema="old_etl_schema",
      identifier="fct_orders"
) -%}

{% set dbt_relation=ref('fct_orders') %}

{{ audit_helper.compare_relation_columns(
    a_relation=old_etl_relation,
    b_relation=dbt_relation
) }}

```

## compare_all_columns ([source](macros/compare_all_columns.sql))
This macro is designed to be added to a dbt test suite to monitor changes
to values when code is changed, as part of a PR or during development. It sets 
up a test that will fail if any column values do not match, with options for more 
or less stringent requirements. If there is a test failure, results can be inspected 
in the warehouse, possibly by using the primary key in the test output to join to
relevant tables in your dev or prod schema to investigate the error.

### Usage:

To create a test for the `stg_customers` model, create a custom test 
in the `tests` subdirectory of your dbt project that looks like this:

```
{{ 
  audit_helper.compare_all_columns(
    model_name='stg_customers', 
    primary_key='id', 
    prod_schema='analytics', 
    exclude_columns=['updated_at']
  ) 
}}
where not perfect_match
/* This `where` statement is an example of a filter you can apply to determine what
constitutes a test failure
*/
```

* `model_name`: The model you're testing.
* `primary_key`: The primary key of that model.
* `prod_schema`: The schema that dbt writes to in production. This is what you'll test your code changes
against, such as during a PR run of the dbt test suite.
* `exclude_columns`: A list of columns that you will not expect to match.

If you want to create test results that include columns for attributes from the model itself 
for easier inspection, that can be written into the test:

```
{{ 
  audit_helper.compare_all_columns(
    model_name='stg_customers', 
    primary_key='id', 
    prod_schema='analytics', 
    exclude_columns=['updated_at']
  ) 
}}
left join {{ ref('stg_customers') }} using(id)
```

This structure also allows for the test to group or filter by any attribute in the model or in 
the macro's output as part of the test, for example:

```
with base_test_cte as (
  {{
    audit_helper.compare_all_columns(
      model_name='stg_customers', 
      primary_key='id', 
      prod_schema='analytics', 
      exclude_columns=['updated_at']
    ) 
  }}
  left join {{ ref('stg_customers') }} using(id)
  where not perfect_match
)
select
  status, --let's say there's a "status" column in the model you're testing
  count(distinct case when conflicting_values then id end) as conflicting_values
from base_test_cte
group by 1
```

You can write a `compare_all_columns` test on individual table; and the test will be run 
as part of a full test suite run.

```
dbt test --select stg_customers
```

If you want to [store results in the warehouse for further analysis](https://docs.getdbt.com/docs/building-a-dbt-project/tests#storing-test-failures), add the `--store-failures`
flag.

```
dbt test --select stg_customers --store-failures
```

## compare_column_values_verbose ([source](macros/compare_column_values_verbose.sql))
This macro will return a query that, when executed, returns the same information as 
`compare_column_values`, but not summarized. `compare_column_values_verbose` enables `compare_all_columns` to give the user more flexibility around what will result in a test failure.


# To-do:
* Macro to check if two schemas contain the same relations
