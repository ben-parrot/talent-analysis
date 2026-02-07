---
Title: NNAF Talent Analysis
---

# NNAF: Talent Analysis

## Executive Summary

```sql talent_board
with
talent_home as (
  select
    t.name,
    t.home_country,
    home_demand.average_demand as home_country_demand_multiplier,
    ww.average_demand as worldwide_demand
  from (select distinct name, home_country from talent_demand) t
  left join talent_demand home_demand
    on home_demand.name = t.name and home_demand.country = t.home_country
  left join talent_demand ww on ww.name = t.name and ww.country = 'WW'
),
ranked as (
  select
    t.name,
    t.country,
    row_number() over (partition by t.name order by t.average_demand desc) as rn
  from talent_demand t
  where t.country != 'WW'
),
top5 as (
  select
    r.name,
    r.rn,
    r.country
  from ranked r
  where r.rn <= 3
)
select
  th.name,
  th.home_country as home_country,
  th.home_country_demand_multiplier,
  th.worldwide_demand,
  string_agg(t.country, ', ' order by t.rn) as top_countries
from talent_home th
left join top5 t on th.name = t.name
group by th.name, th.home_country, th.home_country_demand_multiplier, th.worldwide_demand
order by th.worldwide_demand desc
```

<DataTable data={talent_board} title="Talent Analysis Cohort" rows=all sort="worldwide_demand desc">
  <Column id=name title="Talent"/>
  <Column id=home_country title="Home Country"/>
  <Column id=home_country_demand_multiplier contentType=bar title="Home Country Demand" fmt='#,##0.0'/>
  <Column id=worldwide_demand contentType=bar title="Worldwide Demand" fmt='#,##0.0'/>
  <Column id=top_countries title="Top 3 Markets"/>
</DataTable>

This dashboard uses Parrot demand data to compare the NNAF talent cohort across markets. **Demand** is shown as a multiplier (relative to average); **travelability** measures how well a talent’s demand in a given market or set of markets compares to their home market. All views use the same fixed talent set and exclude the worldwide (WW) aggregate.

***Note:* Demand Data is taken between 1 January 2025 to 28 January 2025.**

**Views:**
- **Global** — Home demand vs. travelability scatter; top 20 markets by share of demand; heatmap of each talent’s share within those markets.
- **Regional** — Average demand or share of global demand by region; select a region to see a talent leaderboard and sub-region heatmap.
- **By country** — Select a market for a leaderboard (demand multiplier and market standing) in that country.
- **By talent** — Select an individual for demand by market, travelability, top markets, and regional breakdown.


```sql demand_vs_travelability
with max_values as (
  select 
    max(average_home_demand) as max_home_demand,
    max(average_travelability) as max_travelability
  from demand_vs_travelability
)
select 
  d.name, 
  d.home_country,
  cm.display_name as home_country_name,
  d.name || ' (' || coalesce(cm.display_name, d.home_country) || ')' as label,
  (d.average_home_demand / mv.max_home_demand) as home_demand_index, 
  (d.average_travelability / mv.max_travelability) as travelability_index
from demand_vs_travelability d
cross join max_values mv
left join country_metadata cm on d.home_country = cm.iso
```

```sql demand_vs_travelability_averages
select 
  avg(home_demand_index) as average_home_demand_index, 
  avg(travelability_index) as average_travelability_index
from ${demand_vs_travelability}
```

## Global Analysis

<ScatterPlot
    data={demand_vs_travelability}
    title="Home Demand vs. Travelability"
    subtitle="Values are indexed to max demand (100% = most in-demand at home or abroad)"
    x=travelability_index xBaseline="true"
    y=home_demand_index yBaseline="true"
    series=label
    chartAreaHeight=300
    xFmt=pct0 xMin=-0.01 xMax=1.1
    yFmt=pct0 yMin=0 yMax=1.1
>
    <ReferenceLine
        data={demand_vs_travelability_averages}
        y=average_home_demand_index
        label="Average Home Demand Index"
        color=#00A7F1
        labelBackground="true"
    />
    <ReferenceLine
        data={demand_vs_travelability_averages}
        x=average_travelability_index
        label="Average Travelability Index"
        color=#00A7F1
        labelBackground="true"
    />
</ScatterPlot>
<Details title="Interpreting this chart">
Values are indexed against the highest demand multiplier. For example, Omar Sy is most popular in his own market, but travels 33% as well as Idris Elba, who performs best outside of the UK, his home market.

### Analysis notes:

Idris Elba, Viola Davis, Ayo Edibiri, and Omar Sy are top performers within this cohort; compared to the average, they are more in demand at home and abroad, on average.
</Details>


```sql largest_markets
with global_total as (
  select sum(average_demand) as total
  from talent_demand
  where country != 'WW'
),
top_markets as (
  select country, sum(average_demand) as total_demand
  from talent_demand
  where country != 'WW'
  group by country
  order by sum(average_demand) desc
  limit 20
)
select 
  country, 
  total_demand,
  total_demand / (select total from global_total) as share_of_global_demand
from top_markets
order by total_demand desc
```

<BarChart
  data={largest_markets}
  x=country
  y=share_of_global_demand
  title="Top 20 Markets: Share of Global Talent Demand"
  subtitle="Percentage of total, global demand across the talent selection"
  yFmt=pct0
  sort=false
  labels=true
  labelPosition=outside
  fillColor="#14ADA4"
/>

```sql heatmap_data
with top_markets as (
  select country
  from talent_demand
  where country != 'WW'
  group by country
  order by sum(average_demand) desc
  limit 20
),
market_totals as (
  select 
    country,
    sum(average_demand) as total_market_demand
  from talent_demand
  where country != 'WW'
  group by country
)
select 
  t.name,
  t.country,
  coalesce(mt.total_market_demand, 0) as total_market_demand,
  t.average_demand as demand_multiplier,
  t.average_demand / coalesce(mt.total_market_demand, 1) as demand_share
from talent_demand t
inner join top_markets tm on t.country = tm.country
left join market_totals mt on t.country = mt.country
where t.country != 'WW'
order by t.name, t.country
```

<Heatmap 
  data={heatmap_data}
  title="Top 20 Markets: Talent Demand Share"
  subtitle="Each cell shows a talent's share of total demand within that market (left is greatest demand)."
  x=country
  y=name
  value=demand_share
  xSort=total_market_demand
  xSortOrder=desc
  colorPalette={['#ffffff', '#00A7F1', '#14ADA4', '#05B462']}
  valueFmt=pct0
  legend=true filter=true
/>
<Note>Note: Click and drag the slider's edges to filter</Note>
<Details title="Interpreting this chart">
This heatmap shows each talent's share of total demand within that country. For example, in the US, Ayo Edebiri's share of demand is 18%, while Damson Idris's demand share is 9%.

The chart corresponds to the 20 markets with the most demand for this cohort, in descending order from left to right (see the bar chart above).

### Analysis notes:

Idris Elba, Viola Davis, Ayo Edebiri, and Damson Idris perform particularly well across all of the top 20 markets. Omar Sy stands out extremely well in France (FR, his home market), Belgium (BE), Switzerland (CH), and Russia (RU).
</Details>

## Regional Analysis

* **Average demand** is the mean demand multiplier across the chosen talent in each region
* **Share of global demand** is each region's total, combined demand as a percentage of global of the chosen talent.

```sql regional_demand
select 
  cm.region,
  t.name as talent_name,
  avg(t.average_demand) as average_demand_multiplier,
  max(t.average_demand) as peak_demand,
  count(distinct t.country) as markets
from talent_demand t
inner join country_metadata cm on t.country = cm.iso
where t.country != 'WW'
  and cm.region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
group by cm.region, t.name
```

```sql region_summary
with regional as (
  select 
    cm.region,
    avg(t.average_demand) as average_demand_multiplier,
    sum(t.average_demand) as total_demand
  from talent_demand t
  inner join country_metadata cm on t.country = cm.iso
  where t.country != 'WW'
    and cm.region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
  group by cm.region
),
global_total as (
  select sum(total_demand) as total from regional
)
select 
  r.region,
  r.average_demand_multiplier,
  r.total_demand,
  r.total_demand / (select total from global_total) as pct_global_demand
from regional r
```

```sql region_first_avg
with regional as (
  select 
    cm.region,
    avg(t.average_demand) as average_demand_multiplier,
    sum(t.average_demand) as total_demand
  from talent_demand t
  inner join country_metadata cm on t.country = cm.iso
  where t.country != 'WW'
    and cm.region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
  group by cm.region
),
global_total as (
  select sum(total_demand) as total from regional
)
select 
  r.region,
  r.average_demand_multiplier,
  round(r.average_demand_multiplier, 2) as multiplier_display
from regional r
order by r.average_demand_multiplier desc
limit 1
```

```sql region_first_pct
with regional as (
  select 
    cm.region,
    avg(t.average_demand) as average_demand_multiplier,
    sum(t.average_demand) as total_demand
  from talent_demand t
  inner join country_metadata cm on t.country = cm.iso
  where t.country != 'WW'
    and cm.region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
  group by cm.region
),
global_total as (
  select sum(total_demand) as total from regional
)
select 
  r.region,
  r.total_demand / (select total from global_total) as pct_global_demand,
  round(100.0 * r.total_demand / (select total from global_total), 0) as pct_display
from regional r
order by r.total_demand / (select total from global_total) desc
limit 1
```

<ButtonGroup name=region_chart_toggle>
  <ButtonGroupItem valueLabel="Average demand (multiplier)" value="avg" default />
  <ButtonGroupItem valueLabel="Share of global demand" value="total" />
</ButtonGroup>

{#if inputs.region_chart_toggle === 'avg'}
<BarChart
  data={region_summary}
  x=region
  y=average_demand_multiplier
  title="Average Demand by Region"
  subtitle="Mean demand multiplier (impact) across talent and markets in each region"
  yFmt='#,##0.00'
  labels=true labelPosition=outside
  sort=true
  fillColor=#00A7F1
/>
<Details title="Interpreting this chart">
In {region_first_avg[0].region}, the average demand of the talent pool is {region_first_avg[0].multiplier_display}x greater than average.
</Details>

{:else}
<BarChart
  data={region_summary}
  x=region
  y=pct_global_demand
  title="Share of Global Demand by Region"
  subtitle="Each region's total demand as a percentage of global (reach)"
  yFmt=pct0
  labels=true labelPosition=outside
  sort=true
  fillColor=#14ADA4
/>
<Details title="Interpreting this chart">
How to interpret this chart: {region_first_pct[0].pct_display}% of the talent pool's demand is concentrated in {region_first_pct[0].region}.
</Details>
{/if}

```sql top_talent_by_region
select 
  cm.region,
  t.name,
  coalesce(cm_home.display_name, t.home_country) as home_country_name,
  avg(t.average_demand) as avg_regional_demand,
  count(distinct t.country) as markets_in_region,
  max(t.market_standing) as best_standing
from talent_demand t
inner join country_metadata cm on t.country = cm.iso
left join country_metadata cm_home on t.home_country = cm_home.iso
where t.country != 'WW'
  and cm.region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
group by cm.region, t.name, home_country_name
```

```sql regions_list
select distinct region 
from country_metadata 
where region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
order by region
```

**Select a region to explore:** <Dropdown
    data={regions_list}
    name=selected_region
    value=region
    defaultValue="Europe"
/>

```sql filtered_regional_talent
select 
  cm.region,
  t.name,
  coalesce(cm_home.display_name, t.home_country) as home_country,
  avg(t.average_demand) as avg_regional_demand,
  count(distinct t.country) as markets_in_region
from talent_demand t
inner join country_metadata cm on t.country = cm.iso
left join country_metadata cm_home on t.home_country = cm_home.iso
where t.country != 'WW'
  and cm.region = '${inputs.selected_region.value}'
group by cm.region, t.name, coalesce(cm_home.display_name, t.home_country)
order by avg_regional_demand desc
```

<DataTable data={filtered_regional_talent} title="Talent Leaderboard: {inputs.selected_region.value}" rows=all sort="avg_regional_demand desc">
  <Column id=name title="Talent"/>
  <Column id=home_country title="Home Country"/>
  <Column id=avg_regional_demand contentType=bar title="Average Demand Multiplier in {inputs.selected_region.value}" fmt='#,##0.00'/>
</DataTable>

```sql regional_heatmap
select 
  cm.region,
  cm.sub_region,
  t.name,
  avg(t.average_demand) as avg_demand
from talent_demand t
inner join country_metadata cm on t.country = cm.iso
where t.country != 'WW'
  and cm.region = '${inputs.selected_region.value}'
  and cm.region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
group by cm.region, cm.sub_region, t.name
```

<Heatmap 
  data={regional_heatmap}
  title="Demand by Sub-Region: {inputs.selected_region.value}"
  subtitle="Average demand multiplier per talent across sub-regions"
  x=sub_region
  y=name
  value=avg_demand
  colorPalette={['#ffffff', '#00A7F1', '#14ADA4', '#05B462']}
  valueFmt='#,##0.0'
  legend=true filter=true
/>
<Details title="Interpreting this chart">
Each cell shows the average demand multiplier for a talent in that sub-region of {inputs.selected_region.value}. Lighter shades indicate higher demand. Use this to see which sub-regions drive the most demand for each talent within the selected region.
</Details>

## Talent Leaderboard per Country

```sql countries
select distinct country from talent_demand
```

```sql countries_enriched
select distinct 
  td.country as value,
  coalesce(cm.display_name, td.country) || ' (' || td.country || ')' as label
from talent_demand td
left join country_metadata cm on td.country = cm.iso
where td.country != 'WW'
order by label
```

**Select a country:** <Dropdown
  data={countries_enriched}
  name=selected_country
  value=value
  label=label
  defaultValue="GB"
/>

This table compares talent demand in **{inputs.selected_country.label}**.

```sql leaderboard
select distinct 
  t.name, 
  coalesce(cm.display_name, t.home_country) as home_country, 
  t.country, 
  t.average_demand, 
  t.market_standing,
  case t.market_standing
    when 'Exceptional' then '<span style="color:#05B462; font-weight:bold;">Exceptional</span>'
    when 'Outstanding' then '<span style="color:#14ADA4; font-weight:bold;">Outstanding</span>'
    when 'Good' then '<span style="color:#00A7F1; font-weight:bold;">Good</span>'
    when 'Average' then '<span style="color:#9844FC; font-weight:bold;">Average</span>'
    when 'Below Average' then '<span style="color:#FF5069; font-weight:bold;">Below Average</span>'
    else t.market_standing
  end as market_standing_colored
from talent_demand t
left join country_metadata cm on t.home_country = cm.iso
where t.country = '${inputs.selected_country.value}'
group by all
```

<DataTable data={leaderboard} sort="average_demand desc" rows="all">
  <Column id=name/>
  <Column id=home_country title="Home Country"/>
  <Column id=market_standing_colored title="Market Standing" contentType=html/>
  <Column id=average_demand contentType=bar title="Demand Multiplier" fmt='#,##0.00'/>
</DataTable>
<Details title="Interpreting this table">
**Market standing** is a Parrot classification of how a talent performs in that market relative to other talent there: Exceptional, Outstanding, Good, Average, Below Average.
</Details>

### Talent Showcase: {inputs.selected_person.label}

This section examines the performance of individuals across different markets. 


```sql talent_imgs
select * from talent.talent_imgs
```


**Select talent:** <Dropdown
    data={talent_imgs}
    name=selected_person
    value=short_id
    label=name
    defaultValue={4059503986}
    fontSize=18
/>

```sql selected_person_image
select img_url, name 
from ${talent_imgs}
where short_id = '${inputs.selected_person.value}'
```

```sql demand_of_talent
select t.short_id, t.name, 
  coalesce(cm.display_name, t.country) as country, 
  t.country as country_iso,
  t.average_demand, t.market_standing, t.market_travelability_index, first(f.flag) as flag,
  case t.market_standing
    when 'Exceptional' then '<span style="color:#05B462; font-weight:bold;">Exceptional</span>'
    when 'Outstanding' then '<span style="color:#14ADA4; font-weight:bold;">Outstanding</span>'
    when 'Good' then '<span style="color:#00A7F1; font-weight:bold;">Good</span>'
    when 'Average' then '<span style="color:#9844FC; font-weight:bold;">Average</span>'
    when 'Below Average' then '<span style="color:#FF5069; font-weight:bold;">Below Average</span>'
    else t.market_standing
  end as market_standing_colored
from talent_demand t
left join flags f on t.country = f.country
left join country_metadata cm on t.country = cm.iso
where t.short_id = '${inputs.selected_person.value}' and t.country != 'WW'
group by t.short_id, t.name, coalesce(cm.display_name, t.country), t.country, t.average_demand, t.market_standing, t.market_travelability_index
order by t.average_demand DESC
```

```sql travelability
select t.short_id, t.name, t.home_country, 
  coalesce(cm.display_name, t.country) as country,
  t.country as country_iso,
  t.market_travelability_index, first(f.flag) as flag
from talent_demand t
left join flags f on t.country = f.country
left join country_metadata cm on t.country = cm.iso
where t.short_id = '${inputs.selected_person.value}' and t.country != 'WW' and t.market_travelability_index > 0
group by t.short_id, t.name, t.country, coalesce(cm.display_name, t.country), t.market_travelability_index, t.home_country
order by t.market_travelability_index desc
limit 15
```

<div style="display: grid; grid-template-columns: 2fr 3fr; gap: 1rem; align-items: center;">
    <div>
        {#if selected_person_image[0]}
            <img 
                src={selected_person_image[0].img_url}
                alt={selected_person_image[0].name}
                style="display: block; margin: 0 auto; max-height: 300px; width: auto; object-fit: contain;"
            />
        {:else}
            <img 
                src="https://placehold.co/200x300/1e293b/94a3b8?text=No+image"
                alt="Talent"
                style="display: block; margin: 0 auto; max-height: 300px; width: auto; object-fit: contain;"
            />
        {/if}
    </div>
    <div class="compact-table">
        <DataTable data={demand_of_talent} title="{inputs.selected_person.label}: Regional Demand">
            <Column id=flag contentType=image height=15px align=center/>
            <Column id=country title="Market"/>
            <Column id=market_standing_colored title="Market Standing" contentType=html/>
            <Column id=average_demand contentType=bar title="Demand Multiplier" fmt='#,##0.00'/>
            <Column id=market_travelability_index contentType=bar fmt=pct0 title="Travelability"/>
        </DataTable>
    </div>
</div>


```sql top_markets_chart
select 
    coalesce(cm.display_name, t.country) as country, 
    t.average_demand, 
    t.market_standing
from talent_demand t
left join country_metadata cm on t.country = cm.iso
where t.short_id = '${inputs.selected_person.value}' and t.country != 'WW'
order by t.average_demand DESC
limit 15
```

```sql top_travelability_chart
select 
    coalesce(cm.display_name, t.country) as country, 
    t.market_travelability_index,
    case when t.home_country = t.country then 'Origin' else 'Other Markets' end as market_type
from talent_demand t
left join country_metadata cm on t.country = cm.iso
where t.short_id = '${inputs.selected_person.value}' and t.country != 'WW' and t.market_travelability_index > 0
order by t.market_travelability_index desc
limit 15
```

<ButtonGroup name=chart_toggle>
    <ButtonGroupItem valueLabel="Demand" value="demand" default />
    <ButtonGroupItem valueLabel="Travelability" value="travelability" />
</ButtonGroup>

{#if inputs.chart_toggle === 'demand'}
<BarChart
    data={top_markets_chart}
    x=country 
    y=average_demand yFmt='#,##0.0'
    series=market_standing
    title="Demand Multiplier in Top Markets"
    sort=false swapXY=true
    labels=true stackTotalLabel=false labelPosition=outside
    seriesColors={{"Exceptional":"#05B462","Outstanding":"#14ADA4","Good":"#00A7F1","Average":"#9844FC","Below Average":"#FF5069"}}
/>
<Details title="Interpreting this chart">
Bars show the demand multiplier for {inputs.selected_person.label} in each of their top 15 markets. Colors indicate market standing (Exceptional, Outstanding, Good, Average, Below Average). Higher values mean stronger demand in that market.
</Details>
{:else}
<BarChart
    data={top_travelability_chart}
    x=country
    y=market_travelability_index
    series=market_type
    title="Market Travelability Indexed Against Home Country"
    yFmt=pct0 yMin=0 yMax=1.05
    sort=false swapXY=true
    labels=true stackTotalLabel=false labelPosition=outside
    seriesColors={{"Origin":"#14ADA4","Other Markets":"#9844FC"}}
/>
<Details title="Interpreting this chart">
Bars show how well {inputs.selected_person.label} travels in each market relative to their home country (100% = same level as at home). Origin is the home market; Other Markets are indexed against it. Higher values mean the talent performs closer to peak levels abroad.
</Details>
{/if}

```sql talent_regional_breakdown
select 
  cm.region,
  avg(t.average_demand) as regional_demand_multiplier,
  count(distinct t.country) as markets,
  max(t.market_standing) as best_standing
from talent_demand t
inner join country_metadata cm on t.country = cm.iso
where t.short_id = '${inputs.selected_person.value}' 
  and t.country != 'WW'
  and cm.region in ('Africa', 'Asia', 'Caribbean', 'Europe', 'Latin America', 'Middle East', 'North America', 'Oceania')
group by cm.region
order by regional_demand_multiplier desc
```

<BarChart
    data={talent_regional_breakdown}
    x=region
    y=regional_demand_multiplier
    yFmt='#,##0.00'
    title="{inputs.selected_person.label}: Demand Multiplier by Region"
    sort=false
    labels=true labelPosition=outside
    fillColor=#9844FC
/>
<Details title="Interpreting this chart">
Average demand multiplier for {inputs.selected_person.label} across all countries in each region. Use this to see which regions drive the most demand for this talent.
</Details>