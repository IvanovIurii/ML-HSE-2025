# NOTES

## Balanced Dataset

```sql
with base as (
    select
        rc.rfq_id,
        rs.supplier_id,
        rs.match_type

    from rfq_core rc
             join recommended_supplier rs on rc.rfq_id = rs.rfq_id

    where rc.created_at > '2025-09-15'
      and rc.status in ('PROCESSED', 'ACCEPTED')
      and rs.match_type in ('match', 'related', 'weak_match', 'no_match')
      limit 10000
), ranked as (
    select *,
           row_number() over (
               partition by match_type
               order by rfq_id, random()
               ) as rn
    from base
)
select match_type, count(*) as cnt
from base
group by match_type;
```

```sql
with base as (
    select
        rc.rfq_id,
        rs.supplier_id,
        rs.match_type

    from rfq_core rc
             join recommended_supplier rs on rc.rfq_id = rs.rfq_id

    where rc.created_at > '2025-09-15'
      and rc.status in ('PROCESSED', 'ACCEPTED')
      and rs.match_type in ('match', 'related', 'weak_match', 'no_match')
      limit 10000
), ranked as (
    select *,
           row_number() over (
               partition by match_type
               order by rfq_id, random()
               ) as rn
    from base
), sample as (
         select *
         from ranked
         where rn <= 413
)
select match_type, count(*) as cnt
from sample
group by match_type
order by match_type;
```

| match_type | count |
|------------|-------|
| match      | 413   |
| no_match   | 413   |
| related    | 413   |
| weak_match | 413   |

And the full SQL what was used to get balanced data:

```sql
with base as (
    select distinct
        rc.rfq_id,
        rs.supplier_id,
        rs.match_type
    from rfq_core rc
             join recommended_supplier rs on rc.rfq_id = rs.rfq_id
    where rc.created_at > '2025-09-15'
      and rc.status in ('PROCESSED', 'ACCEPTED')
      and rs.match_type in ('match', 'related', 'weak_match', 'no_match')
),
     ranked as (
         select *,
                row_number() over (
                    partition by match_type
                    order by rfq_id, random()
                    ) as rn
         from base
     ),
     sample as (
         select rfq_id, supplier_id, match_type
         from ranked
         where rn <= 413
     )
select
    rc.rfq_id,
    rc.title           as rfq_title,
    rc.description     as rfq_description,
    rc.quantity        as quantity,
    rc.delivery_location,
    rc.supplier_types  as rfq_supplier_types,
    rs.name            as supplier_name,
    rs.country         as supplier_country,
    rs.distribution_area,
    rs.description_en  as supplier_description,
    rs.supplier_types,
    rs.products,
    rs.product_categories,
    rs.keywords,
    s.match_type
from sample s
         join rfq_core rc
              on rc.rfq_id = s.rfq_id
         join recommended_supplier rs
              on rs.rfq_id = s.rfq_id
                  and rs.supplier_id = s.supplier_id
                  and rs.match_type = s.match_type;
```
