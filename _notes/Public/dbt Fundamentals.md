---
title : dbt Fundamentals
notetype : feed
date : 2022-06-22
time: 20:57
mermaid: false
mathjax: false
---

# dbt Fundamentals
[Link to the course](https://courses.getdbt.com/courses/fundamentals)

Below are my notes for the class dbt Fundamentals. dbt is a platform for data transformation using Jinja syntax and SQL.

---

## Models
Models are `.sql` files in the `models` folder
`dbt run` materializes a model as a table or view, a model needs to be run first to be referenced in a subsequent model.

### ref Macro
We can use `{{ ref('') }}` to reference an underlying table or view that was built before. This builds dependencies between models, thus constructing the lineage graph

### Naming Conventions
- Sources (*src*): the raw table in the data warehouse
- Staging (*stg*): models that built directly on sources, a raw table (source) has a one-to-one relationship with staging tables. They usually has minimal transformation (renaming, conversion) and are often materialized as views
- Intermediate (*int*): any models that comes before the fact and dimension tables. Should be built on staging tables.
- Fact (*fct*): any data that already occurred or is occurring, e.g. orders, sessions, transactions, stories…
- Dimension (*dim*): data that represents entities and changed less often, e.g. customers, products, buildings

## Sources
Sources are the raw data from the data warehouse, we could define its origin in the yml file to avoid changing the reference everywhere. We then reference the alias defined in the yml file using `{{ source('name', 'table') }}`. We could configure multiple tables from a single source (i.e. schema) at the same time.

### Test
Testing is used to validate the assumptions we have on the data going into the model and out of the model.

- Generic Tests: written in `.yml` file and return the number of tests that fails assertions, they run on a specific column
	1. Freshness
	   Define the `loaded_at_field` to be the field in the data warehouse that specifies data importation date, it’s `_etl_loaded_at` in snowflake. Then we use `warn_after: {count: , period: }` and `error_after: {count: , period: }` to specify the freshness for warning and error (stops subsequent models)
	2. Unique
	3. Not_null
	4. Accept_values: every value in the column is in the list
	5. Relationships: every value in the column exists in a column from another table
- Singular Tests: customized, single tests to run against the whole model, defined in a sql file using `ref('')` to build dependency, contains under the `test` folder

Use `dbt test test_type:generic/singular` to specify test type, use `dbt test model_name` to run tests pertaining to a single model

## Documentation
Documentation in dbt are in the `.yml` files with the optional reference to `.md` files. We use doc blocks in `.md` files to make the markdown syntax accessible from the YAML file, that is {`% docs block_name %`} and  {`% enddocs %`} to enclose a markdown block. We document using the `description` keyword in the YAML file and use `'{{ doc("block_name")}}'` to reference the markdown block.

## Deployment
We usually develop the dbt code in the development branch, once they are ready, we mergey the code into ta deployment branch. The deployment usually runs on a schedule. 