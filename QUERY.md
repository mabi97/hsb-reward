# HSB Reward — Snowflake Queries

## Database Connection

```
Account   : JCIRNXC-PU79913
Database  : HSB
Schema    : RAW
Warehouse : COMPUTE_WH
Role      : ACCOUNTADMIN
Username  : (see GitHub Secrets)
Password  : (see GitHub Secrets)
```

---

## Tables

### data_raw
"with c_type as
    (select customer_id, min(report_time) first_active
    from report_business
    where customer_id like '95%' and report_timeframe = 'daily' and (deposit_success_customer_cc > 0 or rt_active_trader_open_cc > 0)
    group by all)
    
, rebate_user_task_details as
    (select * EXCLUDE (start_date, end_date),
        min(start_date) over (partition by award_period) start_date,
        max(end_date) over (partition by award_period) end_date,
    from rebate_user_task_detail)
--trading retention part
, a1 as
    (select customer_id, report_time, rt_lot_open_sa,
        lead(report_time) over (partition by customer_id order by report_time asc) lead_time
    from report_trading_deal_real
    where report_timeframe = 'weekly' and report_time >= '2025-12-01' and customer_id like '95%' and rt_active_trader_all_cc > 0)
, b1 as
    (select distinct client_id, start_date, end_date, award_period
    from rebate_user_task_details
    where task_type = 1 and start_date >= '2025-12-07')
, c1 as
    (select *
    from a1
    left join b1
    on a1.customer_id = b1.client_id::string and date(b1.start_date) <= dateadd(day, 7, a1.report_time) and date(b1.end_date) >= dateadd(day, 7, a1.report_time))
    
, c11 as
    (select *,
        case when (select max(1) from c_type where customer_id = c1.customer_id::string and left(first_active, 7) = left(dateadd(day, 6, report_time), 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type
    from c1)

, d1 as
    (select dateadd(day, 6, report_time) period, 
        customer_type,
        count(distinct client_id) task1_assigned,
        count(distinct case when lead_time = dateadd(day, 7, report_time) then client_id else null end) task1_assigned_retention,
        count(distinct case when client_id is null then customer_id else null end) task1_non_assigned,
        count(distinct case when client_id is null and lead_time = dateadd(day, 7, report_time) then customer_id else null end) task1_non_assigned_retention
    from c11
    group by all
    union all
    select dateadd(day, 6, report_time) period, 
        'All customer' customer_type,
        count(distinct client_id) task1_assigned,
        count(distinct case when lead_time = dateadd(day, 7, report_time) then client_id else null end) task1_assigned_retention,
        count(distinct case when client_id is null then customer_id else null end) task1_non_assigned,
        count(distinct case when client_id is null and lead_time = dateadd(day, 7, report_time) then customer_id else null end) task1_non_assigned_retention
    from c11
    group by all)

-- deposit retention part  
, a21 as
    (select distinct start_date
    from rebate_user_task_details
    where start_date >= '2025-12-07'
    union all select dateadd(day, 7, max(start_date)) from rebate_user_task_details)
, b21 as
    (select create_date, client_id
    from bo_client_deposit_proposal
    where proposal_status = 2 and create_date >= '2025-11-21')

, c21 as
    (select distinct start_date, client_id, 1 deposit
    from a21
    left join b21
    on a21.start_date >= b21.create_date and dateadd(day, -7, a21.start_date) <= b21.create_date
    union all
    select distinct start_date, client_id, 0 deposit
    from rebate_user_task_details
    where start_date >= '2025-12-07')
, c22 as
    (select *,
    (select min(start_date) from c21 where start_date > c2s.start_date and client_id = c2s.client_id and deposit = 1) lead_time
    from c21 c2s)

, c23 as
    (select date(start_date) period, client_id, 
        case when max(deposit) = 1 then 1 else 0 end deposit,
        case when min(deposit) = 0 then 1 else 0 end task,
        sum(case when date(lead_time) = dateadd(day, 7, date(start_date)) then 1 else 0 end) lead_time
    from c22
    group by all)
    
, c24 as
    (select *,
        case when (select max(1) from c_type where customer_id = c23.client_id::string and left(first_active, 7) = left(period, 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type
    from c23)

, d2 as
    (select period,
        customer_type,
        count(distinct case when task = 1 then client_id else null end) task2_assigned,
        count(distinct case when lead_time > 0 and task = 1 then client_id else null end) task2_assigned_retention,
        count(distinct case when task = 0 then client_id else null end) task2_non_assigned,
        count(distinct case when lead_time > 0 and task = 0 then client_id else null end) task2_non_assigned_retention
    from c24
    group by all
    union all
    select period,
        'All customer' customer_type,
        count(distinct case when task = 1 then client_id else null end) task2_assigned,
        count(distinct case when lead_time > 0 and task = 1 then client_id else null end) task2_assigned_retention,
        count(distinct case when task = 0 then client_id else null end) task2_non_assigned,
        count(distinct case when lead_time > 0 and task = 0 then client_id else null end) task2_non_assigned_retention
    from c24
    group by all)


--trading growth part
,a3 as
    (select client_id, max(task_status) task_status, date(start_date) start_date,
        (select max(1) from rebate_user_task_details where client_id = a.client_id and task_type = 1 and date(start_date) = dateadd(day, -7, date(a.start_date))) last_week,
    from rebate_user_task_details a
    where task_type = 1 and date(start_date) >= '2025-12-07'
    group by all)
, a1 as
    (select customer_id, report_time, rt_lot_open_sa,
        lead(report_time) over (partition by customer_id order by report_time asc) lead_time
    from report_trading_deal_real
    where report_timeframe = 'weekly' and report_time >= '2025-12-01' and customer_id like '95%' and rt_active_trader_all_cc > 0)
, b3 as
    (select *,
        (select sum(rt_lot_open_sa) from a1 where customer_id = a3.client_id::string and report_time = dateadd(day, -6, a3.start_date)) lot_last_week,
        (select sum(rt_lot_open_sa) from a1 where customer_id = a3.client_id::string and report_time = dateadd(day, 1, a3.start_date)) lot_this_week,
    from a3)
, c3 as
    (select *,
        case when (select max(1) from c_type where customer_id = b3.client_id::string and left(first_active, 7) = left(start_date, 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type
    from b3)
, d3 as
    (select start_date period, 
        customer_type,
        avg(case when last_week is null then ifnull(lot_last_week, 0) else null end) task_1_non_task_last_week,
        avg(case when last_week is null then ifnull(lot_this_week, 0) else null end) task_1_non_task_this_week,
        avg(case when task_status in (1,2) then ifnull(lot_last_week, 0) else null end) task_1_completed_last_week,
        avg(case when task_status in (1,2) then ifnull(lot_this_week, 0) else null end) task_1_completed_this_week,
        avg(case when task_status not in (1,2) then ifnull(lot_last_week, 0) else null end) task_1_uncompleted_last_week,
        avg(case when task_status not in (1,2) then ifnull(lot_this_week, 0) else null end) task_1_uncompleted_this_week,
    from c3
    group by all
    union all
    select start_date period, 
        'All customer' customer_type,
        avg(case when last_week is null then ifnull(lot_last_week, 0) else null end) task_1_non_task_last_week,
        avg(case when last_week is null then ifnull(lot_this_week, 0) else null end) task_1_non_task_this_week,
        avg(case when task_status in (1,2) then ifnull(lot_last_week, 0) else null end) task_1_completed_last_week,
        avg(case when task_status in (1,2) then ifnull(lot_this_week, 0) else null end) task_1_completed_this_week,
        avg(case when task_status not in (1,2) then ifnull(lot_last_week, 0) else null end) task_1_uncompleted_last_week,
        avg(case when task_status not in (1,2) then ifnull(lot_this_week, 0) else null end) task_1_uncompleted_this_week,
    from c3
    group by all)
--deposit growth part
, a4 as
    (select distinct client_id, task_status, start_date, end_date,
        ifnull((select sum(amount_converted) from bo_client_deposit_proposal where client_id = a.client_id and proposal_status = 2 and create_date between start_date and end_date), 0) amount_this_week,
        ifnull((select sum(amount_converted) from bo_client_deposit_proposal where client_id = a.client_id and proposal_status = 2 and create_date between dateadd(day,-7,start_date) and start_date), 0) amount_last_week
    from rebate_user_task_details a 
    where task_type = 2 and date(start_date) >= '2025-11-20')
, b4 as
    (select *,
        case when (select date(max(start_date)) from a4 where client_id = a4s.client_id and start_date < a4s.start_date) = dateadd(day, -7, date(start_date)) then 1 else 0 end last_week
    from a4 a4s)
, c4 as
    (select *,
        case when (select max(1) from c_type where customer_id = b4.client_id::string and left(first_active, 7) = left(start_date, 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type
    from b4)
, d4 as
    (select date(start_date) period, 
        customer_type,
        avg(case when last_week = 0 then ifnull(amount_last_week, 0) else null end) task_2_non_task_last_week,
        avg(case when last_week = 0 then ifnull(amount_this_week, 0) else null end) task_2_non_task_this_week,
        avg(case when task_status in (1,2) then ifnull(amount_last_week, 0) else null end) task_2_completed_last_week,
        avg(case when task_status in (1,2) then ifnull(amount_this_week, 0) else null end) task_2_completed_this_week,
        avg(case when task_status not in (1,2) then ifnull(amount_last_week, 0) else null end) task_2_uncompleted_last_week,
        avg(case when task_status not in (1,2) then ifnull(amount_this_week, 0) else null end) task_2_uncompleted_this_week,
    from c4
    group by all
    union all
    select date(start_date) period, 
        'All customer' customer_type,
        avg(case when last_week = 0 then ifnull(amount_last_week, 0) else null end) task_2_non_task_last_week,
        avg(case when last_week = 0 then ifnull(amount_this_week, 0) else null end) task_2_non_task_this_week,
        avg(case when task_status in (1,2) then ifnull(amount_last_week, 0) else null end) task_2_completed_last_week,
        avg(case when task_status in (1,2) then ifnull(amount_this_week, 0) else null end) task_2_completed_this_week,
        avg(case when task_status not in (1,2) then ifnull(amount_last_week, 0) else null end) task_2_uncompleted_last_week,
        avg(case when task_status not in (1,2) then ifnull(amount_this_week, 0) else null end) task_2_uncompleted_this_week,
    from c4
    group by all)
    
select distinct
    TO_VARCHAR(DATEADD(day, 6, d1.period),'YYYYMMDD') period,
    d1.customer_type,
    task1_assigned, task1_assigned_retention, task1_non_assigned, task1_non_assigned_retention,
    task2_assigned, task2_assigned_retention, task2_non_assigned, task2_non_assigned_retention,
    task_1_non_task_last_week, task_1_non_task_this_week, task_1_completed_last_week, task_1_completed_this_week, task_1_uncompleted_last_week, task_1_uncompleted_this_week,
    task_2_non_task_last_week, task_2_non_task_this_week, task_2_completed_last_week, task_2_completed_this_week, task_2_uncompleted_last_week, task_2_uncompleted_this_week
from d1
left join d2 on d1.period = d2.period and d1.customer_type = d2.customer_type
left join d3 on d1.period = d3.period and d1.customer_type = d3.customer_type
left join d4 on d1.period = d4.period and d1.customer_type = d4.customer_type
order by 1, 2"


### data_raw_click
"with c_type as
    (select customer_id, min(report_time) first_active
    from report_business
    where customer_id like '95%' and report_timeframe = 'daily' and (deposit_success_customer_cc > 0 or rt_active_trader_open_cc > 0)
    group by all)
    
, rebate_user_task_details as
    (select * EXCLUDE (start_date, end_date),
        min(start_date) over (partition by award_period) start_date,
        max(end_date) over (partition by award_period) end_date,
    from rebate_user_task_detail)
--trading assigned retention
, a11 as
    (select distinct client_id, start_date, end_date, award_period
    from rebate_user_task_details
    where task_type = 1 and start_date >= '2025-12-07')

, a12 as
    (select *,
        case when (select max(1) from c_type where customer_id = a11.client_id::string and left(first_active, 7) = left(start_date, 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a11.client_id::string and week_group = award_period) click_count
    from a11)
    
, a13 as
    (select award_period period, customer_type, 
        count(distinct client_id) task1_assigned,
        count(distinct case when click_count is not null then client_id else null end) task1_assigned_click,
        sum(click_count) task1_assigned_click_count,
    from a12
    group by all)
    
--trading non_assigned retention 
, a21 as
    (select distinct customer_id, report_time,
    to_varchar(dateadd(day, 12, report_time), 'YYYYMMDD') period 
    from report_trading_deal_real
    where report_timeframe = 'weekly' and report_time >= '2025-12-01' and customer_id like '95%' and rt_lot_open_sa > 0)
, a22 as
    (select *,
        case when (select max(1) from c_type where customer_id = a21.customer_id and left(first_active, 7) = left(dateadd(day, 7, report_time), 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a21.customer_id and week_group = period) click_count
    from a21
    where (select min(client_id) from a11 where award_period = a21.period and client_id::string = a21.customer_id) is null)

, a23 as
    (select period, customer_type, 
        count(distinct customer_id) task1_non_assigned,
        count(distinct case when click_count is not null then customer_id else null end) task1_non_assigned_click,
        sum(click_count) task1_non_assigned_click_count,
    from a22
    group by all)
    
--transaction assigned retention
, a31 as
    (select distinct client_id, start_date, end_date, award_period
    from rebate_user_task_details
    where task_type = 2 and start_date >= '2025-12-07')

, a32 as
    (select *,
        case when (select max(1) from c_type where customer_id = a31.client_id::string and left(first_active, 7) = left(start_date, 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a31.client_id::string and week_group = award_period) click_count
    from a31)
    
, a33 as
    (select award_period period, customer_type, 
        count(distinct client_id) task2_assigned,
        count(distinct case when click_count is not null then client_id else null end) task2_assigned_click,
        sum(click_count) task2_assigned_click_count,
    from a32
    group by all)

--transaction non-assigned retention
, a41 as
    (select distinct customer_id, report_time,
    to_varchar(dateadd(day, 12, report_time), 'YYYYMMDD') period 
    from report_transaction_deposit
    where report_timeframe = 'weekly' and report_time >= '2025-12-01' and customer_id like '95%' and deposit_success_amount_inusd_sa > 0)
, a42 as
    (select *,
        case when (select max(1) from c_type where customer_id = a41.customer_id and left(first_active, 7) = left(dateadd(day, 7, report_time), 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a41.customer_id and week_group = period) click_count
    from a41
    where (select min(client_id) from a31 where award_period = a41.period and client_id::string = a41.customer_id) is null)

, a43 as
    (select period, customer_type, 
        count(distinct customer_id) task2_non_assigned,
        count(distinct case when click_count is not null then customer_id else null end) task2_non_assigned_click,
        sum(click_count) task2_non_assigned_click_count,
    from a42
    group by all)

--trading non_assigned growth
, a51 as
    (select distinct client_id, start_date, end_date, award_period,
    lag(start_date) over (partition by client_id order by start_date) lag_date
    from rebate_user_task_details
    where task_type = 1 and start_date >= '2025-12-07')
    
, a52 as    
    (select *,       
        case when (select max(1) from c_type where customer_id = a51.client_id::string and left(first_active, 7) = left(a51.start_date, 7)) = 1 
            then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a51.client_id::string and week_group = a51.award_period) click_count
    from a51
    where date(start_date) != dateadd(day, 7, date(lag_date)) or lag_date is null)
    
, a53 as
    (select award_period period, customer_type, 
        count(distinct client_id) task1_non_assigned_last_week,
        count(distinct case when click_count is not null then client_id else null end) task1_non_assigned_last_week_click,
        sum(click_count) task1_non_assigned_last_week_click_count,
    from a52
    group by all)
--trading completed 
, a61 as
    (select distinct client_id, start_date, end_date, award_period, task_status
    from rebate_user_task_details
    where task_type = 1 and start_date >= '2025-12-07')

, a62 as
    (select *,
        case when (select max(1) from c_type where customer_id = a61.client_id::string and left(first_active, 7) = left(start_date, 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a61.client_id::string and week_group = award_period) click_count
    from a61)
    
, a63 as
    (select award_period period, customer_type, 
        count(distinct case when task_status in (1,2) then client_id else null end) task1_completed,
        count(distinct case when click_count is not null and task_status in (1,2) then client_id else null end) task1_completed_click,
        sum(case when task_status in (1,2) then click_count else 0 end) task1_completed_click_count,
        count(distinct case when task_status not in (1,2) then client_id else null end) task1_non_completed,
        count(distinct case when click_count is not null and task_status not in (1,2) then client_id else null end) task1_non_completed_click,
        sum(case when task_status not in (1,2) then click_count else 0 end) task1_non_completed_click_count,
    from a62
    group by all)

--transaction non_assigned growth
, a71 as
    (select distinct client_id, start_date, end_date, award_period,
    lag(start_date) over (partition by client_id order by start_date) lag_date
    from rebate_user_task_details
    where task_type = 2 and start_date >= '2025-12-07')
    
, a72 as    
    (select *,       
        case when (select max(1) from c_type where customer_id = a71.client_id::string and left(first_active, 7) = left(start_date, 7)) = 1 
            then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a71.client_id::string and week_group = award_period) click_count
    from a71
    where date(start_date) != dateadd(day, 7, date(lag_date)) or lag_date is null)
    
, a73 as
    (select award_period period, customer_type, 
        count(distinct client_id) task2_non_assigned_last_week,
        count(distinct case when click_count is not null then client_id else null end) task2_non_assigned_last_week_click,
        sum(click_count) task2_non_assigned_last_week_click_count,
    from a72
    group by all)

--transaction completed 
, a81 as
    (select distinct client_id, start_date, end_date, award_period, task_status
    from rebate_user_task_details
    where task_type = 2 and start_date >= '2025-12-07')

, a82 as
    (select *,
        case when (select max(1) from c_type where customer_id = a81.client_id::string and left(first_active, 7) = left(start_date, 7)) = 1 
        then 'New customer' else 'Old Customer' end customer_type,
        (select sum(click_count) from smart_reward_client_event where user_id = a81.client_id::string and week_group = award_period) click_count
    from a81)
    
, a83 as
    (select award_period period, customer_type, 
        count(distinct case when task_status in (1,2) then client_id else null end) task2_completed,
        count(distinct case when click_count is not null and task_status in (1,2) then client_id else null end) task2_completed_click,
        sum(case when task_status in (1,2) then click_count else 0 end) task2_completed_click_count,
        count(distinct case when task_status not in (1,2) then client_id else null end) task2_non_completed,
        count(distinct case when click_count is not null and task_status not in (1,2) then client_id else null end) task2_non_completed_click,
        sum(case when task_status not in (1,2) then click_count else 0 end) task2_non_completed_click_count,
    from a82
    group by all)

select a13.*,
    task1_non_assigned, task1_non_assigned_click, task1_non_assigned_click_count,
    task2_assigned, task2_assigned_click, task2_assigned_click_count,
    task2_non_assigned, task2_non_assigned_click, task2_non_assigned_click_count,
    task1_non_assigned_last_week, task1_non_assigned_last_week_click, task1_non_assigned_last_week_click_count,
    task1_completed, task1_completed_click, task1_completed_click_count, task1_non_completed, task1_non_completed_click, task1_non_completed_click_count,
    task2_non_assigned_last_week, task2_non_assigned_last_week_click, task2_non_assigned_last_week_click_count,
    task2_completed, task2_completed_click, task2_completed_click_count, task2_non_completed, task2_non_completed_click, task2_non_completed_click_count,
from a13
left join a23 on a13.period = a23.period and a13.customer_type = a23.customer_type
left join a33 on a13.period = a33.period and a13.customer_type = a33.customer_type
left join a43 on a13.period = a43.period and a13.customer_type = a43.customer_type
left join a53 on a13.period = a53.period and a13.customer_type = a53.customer_type
left join a63 on a13.period = a63.period and a13.customer_type = a63.customer_type
left join a73 on a13.period = a73.period and a13.customer_type = a73.customer_type
left join a83 on a13.period = a83.period and a13.customer_type = a83.customer_type
order by 1 desc, 2
"


### Task_Group_Click_Detail
"with a as
    (select client_id, group_name,min(start_date) start_date, max(end_date) end_date, award_period
    from rebate_user_task_detail
    where award_period >= '20260509'
    group by all)
, b as
    (select *,
        (select min(to_timestamp(nullif(first_deposit_time, 0)/1000)) from crm_customer_base where client_id = a.client_id) first_deposit_time,
        (select count(*) from smart_reward_client_event where user_id = a.client_id and week_group = award_period) click_c
    from a)

select award_period, group_name, 

    count(distinct case when first_deposit_time between start_date and end_date then client_id end) active_this_week,
    count(distinct case when first_deposit_time between start_date and end_date and click_c > 0 then client_id end) active_click_this_week_user,
    sum(case when first_deposit_time between start_date and end_date then click_c end) active_click_this_week_count,
    
    count(distinct case when first_deposit_time not between start_date and end_date then client_id end) non_active_this_week,
    count(distinct case when first_deposit_time not between start_date and end_date and click_c > 0 then client_id end) non_active_click_this_week_user,
    sum(case when first_deposit_time not between start_date and end_date then click_c end) non_active_click_this_week_count,
from b
group by all
order by 1,2"