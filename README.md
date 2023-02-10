#dbt Style Guide

## Model Naming

dbt models should always be classified in 3 categories: base, staging, marts

- Base only selects from source and does not contain any logic. These base models only have column reordering/renaming and type casting.
- Staging models takes the base models, and clean and prepare them for further analysis.
  - It can contain joins and calculations to create new columns.
  - It's necessary to ensure an unique and not null primary key
  - Separate complex logic into intermediate tables.
  - Each row, active row if we think about the history mode of fivetran, should represent an atomic unit of the entity this table is trying to represent
- Marts are grouped by business unit: ops, finance, product. Models that are shared across an entire business are grouped in a core directory.

## dbt best practices

- Only `base__` models should select from `source`s.
- All the other models should select from models only!

## Folder structure

```
├── dbt_project.yml
└── models
    ├── marts
    |   └── core
    |       ├── intermediate
    |       |   ├── intermediate.yml
    |       |   ├── some_stripe_data__joined.sql
    |       |   ├── membership_data__validated.sql
    |       └── core.yml
    |       └── core.docs
    |       └── dim_clients.sql
    |       └── fct_subscriptions.sql
    └── staging
        └── stripe
            ├── base
            |   ├── base__stripe_payments.sql
            ├── src_stripe.yml
            ├── stg_stripe.yml
            ├── stg_stripe__payments.sql
```

- Base tables are prefixed with `base__`, such as: `base__<source>_<object>`
- All objects should be plural, such as: `stg_production__subscriptions`
- Intermediate tables should end with a past tense verb indicating the action performed on the object, such as: `stripe_payment__validated`
- Marts are categorized between fact (immutable, verbs) and dimensions (mutable, nouns) with a prefix that indicates either, such as: `fct_subscriptions` or `dim_customers`

## Model configuration

- Marts should always be configured as tables

## Testing

- Every subdirectory should contain a `.yml` file, in which each model in the subdirectory is tested. For staging folders, the naming structure should be `src_sourcename.yml`. For other folders, the structure should be `foldername.yml` (example `core.yml`).
- At a minimum, unique and not_null tests should be applied to the primary key of each model.

## Naming and field conventions

- Schema, table and column names should be in `lower_snake_case`. We should not have any uppercase in the models!
- Use names based on the _business_ terminology, rather than the source terminology.
- Each model should have a primary key.
- The primary key of a model should be named `<object>_id`, e.g. `account_id` – this makes it easier to know what `id` is being referenced in downstream joined models.
- For base/staging models, fields should be ordered in categories, where identifiers (ids) are first and timestamps are at the end.
- Timestamp columns should be named `<event>_at`, e.g. `created_at`.
- Booleans should be prefixed with `is_` or `has_`, e.g. `has_resell` or `is_fraud`.
- Avoid reserved words as column names.

## CTEs

- All `{{ ref('...') }}` statements should be placed in CTEs at the top of the file.
- Create as many CTEs as necessary. Each one, whenever it's possible, should do a logical unit of work.
- CTEs that are duplicated across models should be pulled out into their own models
- create a `final` or similar CTE that you select from as your last line of code. This makes it easier to debug code within a model (without having to comment out code!)
- CTEs should be formatted like this:

``` sql
with

events as (

    ...

),

-- CTE comments go here
filtered_events as (

    ...

)

select * from filtered_events
```

## SQL style guide

- Use trailing commas
- Indents should be four spaces (except for predicates, which should line up with the `where` keyword)
- Lines of SQL should be no longer than 80 characters
- Field names and function names should all be lowercase
- The `as` keyword should be used when aliasing a field or table
- Fields should be stated before aggregates / window functions
- Aggregations should be executed as early as possible before joining to another table.
- Ordering and grouping by a number (eg. group by 1, 2) is preferred over listing the column names. This is necessary as if you want to group by a calculated column, the logic would also exist in the `group by` statement. Note that if you are grouping by more than a few columns, it may be worth revisiting your model design.
- Prefer `union all` to `union` [*](http://docs.aws.amazon.com/redshift/latest/dg/c_example_unionall_query.html)
- If joining two or more tables, _always_ prefix your column names with the table alias. If only selecting from one table, prefixes are not needed.
- Be explicit about your join (i.e. write `inner join` instead of `join`). `left joins` are normally the most useful, `right joins` often indicate that you should change which table you select `from` and which one you `join` to.

- _DO NOT OPTIMIZE FOR A SMALLER NUMBER OF LINES OF CODE. NEWLINES ARE CHEAP, BRAIN TIME IS EXPENSIVE_

### Example SQL

```sql
with

my_data as (

    select * from {{ ref('my_data') }}

)

some_cte_agg as (

    select
        id,
        sum(field_4) as total_field_4,
        max(field_5) as max_field_5

    from my_data
    group by 1

),

final as (

    select [distinct]
        my_data.field_1,
        my_data.field_2,
        my_data.field_3,

        -- use line breaks to visually separate calculations into blocks
        case
            when my_data.cancellation_date is null
                and my_data.expiration_date is not null
                then expiration_date
            when my_data.cancellation_date is null
                then my_data.start_date + 7
            else my_data.cancellation_date
        end as cancellation_date,

        some_cte_agg.total_field_4,
        some_cte_agg.max_field_5

    from my_data
    left join some_cte_agg
        on my_data.id = some_cte_agg.id
    where my_data.field_1 = 'abc'
        and (
            my_data.field_2 = 'def' or
            my_data.field_2 = 'ghi'
        )
    having count(*) > 1

)

select * from final

```

## YAML style guide

- Indents should be two spaces
- List items should be indented
- Use a new line to separate list items that are dictionaries where appropriate
- Lines of YAML should be no longer than 80 characters.

## TIP

_If you're struggling to understand how to put together the YAML file, please copy and paste. If you still unsure on how to do it, ask for help in our #help_dbt slack group_

### Example YAML

```yaml
version: 2

models:
  - name: events
    columns:
      - name: event_id
        description: This is a unique identifier for the event
        tests:
          - unique
          - not_null

      - name: event_time
        description: "When the event occurred in UTC (eg. 2018-01-01 12:00:00)"
        tests:
          - not_null

      - name: user_id
        description: The ID of the user who recorded the event
        tests:
          - not_null
          - relationships:
              to: ref('users')
              field: id
```

## Jinja style guide

- When using Jinja delimiters, use spaces on the inside of your delimiter, like `{{ this }}` instead of `{{this}}`
- Use newlines to visually indicate logical blocks of Jinja

## Reference Docs

- [CTE](https://discourse.getdbt.com/t/why-the-fishtown-sql-style-guide-uses-so-many-ctes/1091)
- [dbt style guide](https://discourse.getdbt.com/t/how-we-structure-our-dbt-projects/355)
