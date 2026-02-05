# NNAF: Talent Analysis

<!-- Import tables -->
```sql talent
select * from talent.talent
```

```sql demand_vs_travelability
select * from talent.demand_vs_travelability
```

```sql talent_demand
select * from talent.talent_demand
```

```sql talent_imgs
select * from talent.talent_imgs
```

```sql flags
select * from talent.flags
```

```sql averages
select avg(average_home_demand) as avg_home,
avg(average_travelability) as avg_trvl
FROM talent.demand_vs_travelability
```

## Summary

<ScatterPlot 
    data={demand_vs_travelability}
    x=average_travelability
    y=average_home_demand
    series=name
    title="Talent Travelability"
    chartAreaHeight=300
    xMin=0
    yMin=0
>
    <ReferenceLine 
        data={averages} 
        y=avg_home 
        label="Average Home Talent Demand"
        color=cyan
    />
    <ReferenceLine 
        data={averages} 
        x=avg_trvl 
        label="Average Talent Travelability"
        color=cyan
    />
</ScatterPlot>

## Talent Leaderboard

```sql countries
select distinct country from talent_demand
```

This table compares talent demand in a specific country.

<Dropdown
    data={countries}
    name=selected_country
    value=country
    defaultValue="GB"
/>

```sql leaderboard
select distinct name, home_country, country, average_demand, market_standing
from talent_demand
where country = '${inputs.selected_country.value}'
group by all
```

<DataTable data={leaderboard} sort="average_demand desc">
    <Column id=name/>
    <Column id=home_country/>
    <Column id=market_standing/>
    <Column id=average_demand contentType=bar/>
</DataTable>

## Talent Showcase: {inputs.selected_person.label}

This section examines the performance of individuals across different markets. 

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
select t.short_id, t.name, t.country, t.average_demand, t.market_standing, t.market_travelability_index, first(f.flag) as flag,
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
where t.short_id = '${inputs.selected_person.value}' and t.country != 'WW'
group by t.short_id, t.name, t.country, t.average_demand, t.market_standing, t.market_travelability_index
order by t.average_demand DESC
```

```sql travelability
select t.short_id, t.name, t.home_country, t.country, t.market_travelability_index, first(f.flag) as flag
from talent_demand t
left join flags f on t.country = f.country
where t.short_id = '${inputs.selected_person.value}' and t.country != 'WW' and t.market_travelability_index > 0
group by t.short_id, t.name, t.country, t.market_travelability_index, t.home_country
order by t.market_travelability_index desc
limit 15
```

<div style="display: grid; grid-template-columns: 2fr 3fr; gap: 1rem; align-items: center;">
    <div>
        {#if selected_person_image[0]}
        <img 
            src={selected_person_image[0].img_url}
            alt={selected_person_image[0].name}
            height="600"
            style="display:block; margin:0 auto;"
        />
        {:else}
        <img 
            src="https://placehold.co/200x300/1e293b/94a3b8?text=No+image"
            alt="Talent"
            height="200"
            style="display:block; margin:0 auto;"
        />
        {/if}
    </div>
    <div>
        <DataTable data={demand_of_talent} title="{inputs.selected_person.label}: Regional Demand">
            <Column id=flag contentType=image height=25px align=center/>
            <Column id=country/>
            <Column id=market_standing_colored title="Market Standing" contentType=html/>
            <Column id=average_demand contentType=bar/>
            <Column id=market_travelability_index contentType=bar fmt=pct0/>
        </DataTable>
    </div>
</div>

```sql top_markets_chart
select 
    t.country, 
    t.average_demand, 
    t.market_standing
from talent_demand t
where t.short_id = '${inputs.selected_person.value}' and country != 'WW'
order by t.average_demand DESC
limit 15
```

```sql top_travelability_chart
select 
    t.country, 
    t.market_travelability_index,
    case when t.home_country = t.country then 'Origin' else 'Other Markets' end as market_type
from talent_demand t
where t.short_id = '${inputs.selected_person.value}' and t.country != 'WW' and t.market_travelability_index > 0
order by t.market_travelability_index desc
limit 15
```

<Grid cols=2>


<BarChart
    data={top_markets_chart}
    x=country
    y=average_demand
    series=market_standing
    title="Market Standing by Region"
    sort=false
    seriesColors={{"Exceptional":"#05B462","Outstanding":"#14ADA4","Good":"#00A7F1","Average":"#9844FC","Below Average":"#FF5069"}}
/>
<BarChart
    data={top_travelability_chart}
    x=country
    y=market_travelability_index
    series=market_type
    title="Market Travelability"
    yFmt=pct0
    sort=false
    seriesColors={{"Origin":"#05B462","Other Markets":"#00A7F1"}}
/>
</Grid>