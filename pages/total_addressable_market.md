---
Title: NNAF Market Analysis
---

# NNAF: Total Addressable Market

Summary of the Total Addressable Market (TAM) and Serviceable Obtainable Market (SOM) across NNAF subscription markets from Q1 2023 through Q4 2025. Country data is enriched with geographic classifications from country metadata to support aggregation at the region, sub-region, and market level.

**Definitions**
* *TAM*: Total addressable market
* *SOM*: Serviceable obtainable market

## TAM & SOM Over Time

```sql tam_global_trend
select
  t.quarter_end,
  sum(t.TAM::double)::double as global_total_addressable_market,
  sum(t.SOM::double)::double as global_serviceable_obtainable_market,
  sum(t.total_subscriptions::double)::double as global_subs,
  count(distinct t.market) as market_count
from TAM_nnaf_quarterly_totals t
group by t.quarter_end
order by t.quarter_end
```

<LineChart
  data={tam_global_trend}
  x=quarter_end
  y={["global_total_addressable_market", "global_serviceable_obtainable_market"]}
  title="Global TAM vs SOM Over Time"
  yFmt='#,##0.0,,"M"'
  labels=true
/>

```sql tam_quarterly_trend
select
  t.quarter_end,
  t.region,
  sum(t.TAM::double)::double as total_tam,
  sum(t.SOM::double)::double as total_som
from TAM_nnaf_quarterly_totals t
group by t.quarter_end, t.region
order by t.quarter_end
```
<ButtonGroup name=tam_som_chart_toggle>
  <ButtonGroupItem valueLabel="Regional TOM" value="tom" default />
  <ButtonGroupItem valueLabel="Regional SOM" valye="som" />
</ButtonGroup>

{#if inputs.tam_som_chart_toggle === 'tom'}
<AreaChart
  data={tam_quarterly_trend}
  x=quarter_end
  y=total_tam yFmt='#,##0.0,,"M"'
  series=region
  title="TAM by Region Over Time"
  subtitle="Stacked by NNAF region (APAC, EMEA, LATAM, UCAN)"
  type=stacked
/>
{:else}
<AreaChart
  data={tam_quarterly_trend}
  x=quarter_end
  y=total_som yFmt='#,##0.0,,"M"'
  series=region
  title="SOM by Region Over Time"
  subtitle="Stacked by region (APAC, EMEA, LATAM, UCAN)"
  type=stacked
/>
{/if}

## Latest Quarter Snapshot

```sql tam_snapshot
select
  sum(TAM) as global_tam,
  sum(SOM) as global_som,
  sum(total_subscriptions) as global_subs,
  count(distinct market) as market_count
from TAM_nnaf_quarterly_totals
where quarter_end = (select max(quarter_end) from TAM_nnaf_quarterly_totals)
```

<BigValue data={tam_snapshot} value=global_tam title="Global TAM" fmt='#,##0.00,,"M"' />
<BigValue data={tam_snapshot} value=global_som title="Global SOM" fmt='#,##0.00,,"M"' />
<BigValue data={tam_snapshot} value=global_subs title="Total Subscriptions" fmt='#,##0.00,,"M"' />
<BigValue data={tam_snapshot} value=market_count title="Markets" />

<!-- ### TAM by Region -->

<!-- ### Regional Summary -->

```sql tam_region_summary
select
  t.region as nnaf_region,
  count(distinct t.market) as markets,
  sum(t.total_subscriptions) as total_subscriptions,
  sum(t.subscriber_penetration) as subscriber_penetration,
  sum(t.TAM) as total_tam,
  sum(t.SOM) as total_som,
  case when sum(t.TAM) > 0 then sum(t.SOM) / sum(t.TAM) else null end as som_capture_rate
from TAM_nnaf_quarterly_totals t
where t.quarter_end = (select max(quarter_end) from TAM_nnaf_quarterly_totals)
group by t.region
order by total_tam desc
```

<DataTable data={tam_region_summary} title="TAM & SOM by Region" rows=all>
  <Column id=nnaf_region title="Region"/>
  <Column id=markets title="Markets"/>
  <Column id=total_subscriptions title="Total Subscriptions" fmt='#,##0.0,,"M"'/>
  <Column id=subscriber_penetration title="Subscriber Penetration" fmt='#,##0.0,,"M"'/>
  <Column id=total_tam title="TAM" contentType=bar fmt='#,##0.0,,"M"'/>
  <Column id=total_som title="SOM" contentType=bar fmt='#,##0.0,,"M"'/>
  <Column id=som_capture_rate title="SOM/TAM" fmt='pct0'/>
</DataTable>

```sql tam_regional
select
  cm.region as geo_region,
  sum(t.TAM) as total_addressable_market,
  sum(t.SOM) as serviceable_obtainable_market,
  count(distinct t.market) as markets
from TAM_nnaf_quarterly_totals t
left join country_metadata cm on t.market = cm.iso
where t.quarter_end = (select max(quarter_end) from TAM_nnaf_quarterly_totals)
  and t.TAM > 0
group by cm.region
order by total_addressable_market desc
```

<BarChart
  data={tam_regional}
  x=geo_region
  y={["total_addressable_market", "serviceable_obtainable_market"]}
  yFmt='#,##0.0,,"M"'
  sort=true
  labels=true
  labelPosition=outside
  type=grouped
/>

### Top 15 Markets

```sql tam_top_markets
select
  t.market,
  coalesce(cm.display_name, t.market) as country_name,
  t.TAM,
  t.SOM
from TAM_nnaf_quarterly_totals t
left join country_metadata cm on t.market = cm.iso
where t.quarter_end = (select max(quarter_end) from TAM_nnaf_quarterly_totals)
  and t.TAM > 0
order by t.TAM desc
limit 15
```

<ButtonGroup name=tam_som_chart_toggle_2>
  <ButtonGroupItem valueLabel="Total Addressable Market" value="tom" default />
  <ButtonGroupItem valueLabel="Serviceable Obtainable Market" valye="som" />
</ButtonGroup>

{#if inputs.tam_som_chart_toggle_2 === 'tom'}
<BarChart
  data={tam_top_markets}
  x=country_name
  y=TAM
  title="Top 15 Markets by TAM"
  yFmt='#,##0.00,,"M"'
  sort=false
  swapXY=true
  labels=true
  labelPosition=outside
  fillColor="#14ADA4"
/>
{:else}
<BarChart
  data={tam_top_markets}
  x=country_name
  y=SOM
  title="Top 15 Markets by SOM"
  yFmt='#,##0.00,,"M"'
  sort=false
  swapXY=true
  labels=true
  labelPosition=outside
  fillColor="#14ADA4"
/>
{/if}

<!-- ## All Market Data

```sql tam_all_markets
select
  t.quarter_end,
  t.market,
  coalesce(cm.display_name, t.market) as country_name,
  cm.region as geo_region,
  cm.sub_region,
  t.region as nnaf_region,
  t.total_subscriptions,
  t.subscriber_penetration,
  t.TAM,
  t.SOM,
  case when t.TAM > 0 then t.SOM / t.TAM else null end as som_capture_rate
from TAM_nnaf_quarterly_totals t
left join country_metadata cm on t.market = cm.iso
order by t.quarter_end desc, t.TAM desc
```

<DataTable data={tam_all_markets} title="All Markets: TAM & SOM (All Quarters) - Downloadable" rows=20 search=true downloadable=true>
  <Column id=quarter_end title="Quarter" fmt='MM-yyyy'/>
  <Column id=nnaf_region title="NNAF Region"/>
  <Column id=country_name title="Market"/>
  <Column id=market title="ISO"/>
  <Column id=total_subscriptions title="Total Subscriptions" fmt='#,##0'/>
  <Column id=subscriber_penetration title="Subscriber Penetration" fmt='#,##0'/>
  <Column id=TAM title="TAM" fmt='#,##0'/>
  <Column id=SOM title="SOM" fmt='#,##0'/>
  <Column id=som_capture_rate title="SOM/TAM" fmt='pct0'/>
  <Column id=geo_region title="Region"/>
  <Column id=sub_region title="Sub-Region"/>
</DataTable> -->
