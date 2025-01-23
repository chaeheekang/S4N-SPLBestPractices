# S4N-SPLBestPractices

## SPL#1 > Success Purchase Actions for ip
index=main sourcetype=access_combined action=purchase status=200 125.17.14.100

## SPL#2 > Exploring the TERM directive
index=main sourcetype=access_combined action=purchase status=200 TERM(125.17.14.100)

## SPL#3 > tstats with PREFIX
- classic search using stats
```
index=_internal group=thruput name=thruput
| bin span=1767s _time
| stats
sum(kb) as indexer_kb
avg(instantaneous_kbps) as instantaneous_kbps
avg(load_average) as load_avg
by host _time
tstats search
|tstats sum(PREFIX(kb=)) AS indexer_kb avg(PREFIX(instantaneous_kbps=)) AS instantaneous_kbps avg(PREFIX(load_average=)) AS load_avg
WHERE
index=_internal
TERM(group=thruput)
TERM(name=thruput)
BY host _time
span=1767s
```
## SPL#4 > makeresults and eval
```
| makeresults
| eval distance_km=200,time_hrs=10,
speed_kmph=distance_km/time_hrs,
speed_metpersec=speed_kmph/3.6,
speed_metpersec=tostring(speed_metpersec)." m/s"
| table distance_km, time_hrs, speed_kmph, speed_metpersec
```
- Task using SPL#4 > Using eval to customize outputs
```
index=main sourcetype=access_combined action=purchase status=200
| top category limit=3
| eval customer_sentiment=case(category="Books","Reading Books",category="Gifts","Sharing Gifts", category="Clothing","Trying New Clothes")
| eval "Final Message"="Most of Our Customers Love ".customer_sentiment
| fields "Final Message"
```

- Backup lookup command to be used only in case the lookup is not automatically added.##
| lookup product_codes.csv product_id

## SPL#5 > eval and the Field Name Trick
```
index=main sourcetype=access_combined status=200
| eval is_{category}=1
```
- Challenge solution
```
index=main sourcetype=access_combined action=purchase status=200
| eval Total_Sales_of_{category} = product_price
| stats sum(Total_Sales_of_*) as Total_Sales_of_*
```
## SPL#6 > Macros
- Example 1
```
index=main sourcetype=access_combined action=purchase status=200
| `ConvertUSD`
```
- To preview whats behind macro you can press ctrl+shift+e for Win or command+shift+e for mac.
## SPL#7 stats, eventstats and streamstats - search examples
- stats
```
index=main sourcetype=access_combined action=purchase status!=200
| stats count as "Total Unsuccessful Requests"
```
- eventstats
```
index=main sourcetype=access_combined action=purchase status!=200
| eventstats count as "Total Unsuccessful Requests"
```
- streamstats
```
index=main sourcetype=access_combined action=purchase status!=200
| streamstats count as "Total Unsuccessful Requests"
```
## SPL#8 > Popularity Ranking
```
index=main sourcetype=access_combined action=purchase status=200
| stats count by product_name, category
| sort - count
| streamstats count as rank by category
| stats list(rank) as "Category Rank", list(product_name) as "Product Name" by category
```
## SPL#9 > Task Using streamstats and eventstats
```
index=main sourcetype=access_combined action=purchase status=200
| reverse
| streamstats sum(product_price) as revenue_tracker
| eval txn_count = case(revenue_tracker<500, " <500", revenue_tracker>=500 AND revenue_tracker<1000, "500<1000", revenue_tracker>=1000, ">1000"),
txn_count_{txn_count}=txn_count
| stats count(txn_count_*) as "Transaction Count *" count as "Total Transaction Count" sum(product_price) as "Total Sales"
```
## SPL#10 > Spath as SPL Command
```
index=sample_data application_performance
| spath input=_raw output=new_json_field path=application_performance{}
```
## SPL#11 > spath as an eval function
- Search 1
```
index=sample_data application_performance
| eval new_metricName=spath(_raw,"application_performance{}.metricName")
```
- Search 2
```
index=sample_data application_performance
| eval new_metricName=spath(_raw,"application_performance{}.metricName")
| eval new_metricValue=spath(_raw,"application_performance{}.metricValues{}.current")
```
- Search 3 Validate it a new field was created
```
index=sample_data application_performance earliest=0
| spath input=_raw output=new_json_field path=application_performance{}
| eval new_metricName=spath(new_json_field,"metricName")
```
- Search 4
```
index=sample_data application_performance
| spath input=_raw output=new_json_field path=application_performance{}
| table new_json_field
```

## SPL#12 > mvexpand command
- Search 1:
```
index=sample_data application_performance
| spath input=_raw output=new_json_field path=application_performance{}
| mvexpand new_json_field
| table new_json_field
```
- Search 2 :
```
index=sample_data application_performance
| spath input=_raw output=new_json_field path=application_performance{}
| mvexpand new_json_field
| eval new_metricName=spath(new_json_field,"metricName")
```
## SPL#13 > Introducing the rex command
- Example 1
```
index=sample_data healthrule_violations
| rex field=appd_app_uuid "^\w+\:\d+\:[\w\-]+\.(?<delivery>\w+)\."
```
## SPL#14 > Using the rex commMd with SED Mode
```
index=sample_data healthrule_violations earliest=0
| rex field=healthrule_violations{}.description "Used\s\%\'s\<\/b\>\svalue\s\<b\>(?<mem_used_value>(.+?))\<"
| rex mode=sed field=application_id "s/(\d{3})(\d{2})/\1xx/g"
| stats list(mem_used_value) AS "List of Memory Heap Values", avg(mem_used_value) AS "Average Memory Heap Used" BY application_id
| rename application_id AS "Application ID"
```
