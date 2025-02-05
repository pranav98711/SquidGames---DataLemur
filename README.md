# SQL Squid Game by Data Lemur Solutions - Pranav Sorte

![image](https://github.com/user-attachments/assets/e9c6b915-76aa-4e3d-b04b-356d3c978001)



Link to the SQL Squid game and questions  -> https://datalemur.com/sql-game

## My Solutions:
## Level 1
To identify vulnerable living players, I focused on filtering the player table for those who are alive and in severe debt (debt > 400,000,000 won). Next, I applied additional conditions to target players who are either elderly (age > 65) or have a Gambling vice with no close family connections, as these factors indicate higher susceptibility to manipulation. Using logical operators (AND, OR), I structured the query to capture players meeting all the criteria accurately
<pre> <code>```
select * from player
where debt > 400000000 and status = 'alive'
and (age > 65
or (vice = 'Gambling' and has_close_family = false))
          
   ```</code> </pre> 
   
## Level 2
To determine the number of food rations required to feed 90% of the remaining (alive) non-insider players, I first needed to filter the player table for players who are alive and not insiders. Once I had this subset, I calculated 90% of the total count using the FLOOR() function to round the number down to ensure we're not overestimating the rations. Next, I compared this required number of rations against the current ration supply from the rations table. Using a CASE statement, I checked if the available rations were sufficient (i.e., if the supply was greater than or equal to the calculated need), returning TRUE or FALSE accordingly. This approach provides both the exact number of rations needed and a quick sufficiency check to support resource optimization.


<pre> <code>```
--Calculate 90% of the alive, non insider players 
select floor(count(id)*0.9) as ninty_percent,
--Check if the current ration supply is sufficient
case when (select amount from rations) > floor(count(id)*0.9) then true else false end as Sufficient
from player
where isinsider = false and status='alive'

   ```</code> </pre> 
   
## Level 3
To analyze how seasonal variations affect player performance in the Honeycomb game, I focused on comparing data from two contrasting months: August (hot summer) and January (cold winter). The first step was to filter the honeycomb_game table for records from these months within the last 20 years, ensuring the data is both relevant and recent. After filtering, I grouped the data by both the shape of the honeycomb and the month to compare average completion times across different conditions. Finally, I calculated the average completion time for each group to identify patterns in performance based on seasonal temperatures. Sorting the results by average completion time helps highlight which shapes and seasons affect performance the most.

<pre> <code>```
--Filter based on dates and range
with temp_cte as (
select * from honeycomb_game
where EXTRACT(MONTH FROM date) in (8,1) 
and  date > (CURRENT_DATE - INTERVAL '20 years')
)
--Calculate average completion time grouped by shape and month
SELECT shape,EXTRACT(MONTH FROM date),avg(average_completion_time) FROM temp_cte
group by shape,EXTRACT(MONTH FROM date)
order by avg(average_completion_time ) 
                


   ```</code> </pre> 
   
## Level 4
To analyze and rank the teams for the Tug of War game, I first needed to focus on teams with exactly 10 players, as specified in the task. This required filtering the player table for alive players with valid team_ids and grouping them by team_id to count their members. After identifying the eligible teams, I calculated their average player age. Next, I categorized these teams into three age groups based on their average age: 'Fit' for teams under 40, 'Grizzled' for ages between 40 and 50, and 'Elderly' for teams over 50. Finally, I ranked the teams in descending order of average age, with the highest average age assigned rank 1. This approach ensures that the Front Man receives a clear, ranked demographic analysis for strategic planning.

<pre> <code>```
-- Identify teams with exactly 10 alive players and calculate their average age      
with teamAge_summ  as (select team_id,avg(age) as average_age,count(id) as num_players  from player		  
	where status = 'alive' and team_id is not NULL
	group by team_id
	having count(id) = 10),
-- Categorize teams based on their average player age
teamAge_category  as (
select *,
	case when average_age < 40 then 'Fit' 
	when average_age between 40 and 50 then 'Grizzled' 
	when average_age > 50 then 'Elderly'  end as age_group
	from teamAge_summ
)
--Rank teams based on average player age in descending order
select team_id,average_age,	age_group,rank() over (order by average_age desc) as avg_rank
from teamAge_category 


   ```</code> </pre> 
   
## Level 5
To identify Player 456's closest companion in the Marbles game, I needed to analyze interaction data from the daily_interactions table. Since Player 456 could appear as either player1_id or player2_id in the records, I planned to capture all interactions involving Player 456 from both perspectives. After consolidating these interactions, I aggregated the counts to find the player with the highest total interactions. To ensure this player is still relevant, I verified their status as alive using the player table. Finally, I retrieved the first names of both Player 456 and their closest companion, along with the total number of interactions they shared. This step-by-step approach ensures comprehensive and accurate identification of Player 456’s closest companion.

<pre> <code>```
--Find/union all interactions where Player 456 is involved (as player1 or player2)
with player_456_interactions as (select player1_id,count(player1_id) 
	from daily_interactions
	where player2_id=456 group by player1_id
	union
	select player2_id,count(player2_id) from daily_interactions
	where player1_id=456
	group by player2_id),
-- Aggregate total interactions per player and rank them by interaction count
interaction_summary as 
(select player1_id as player,sum(count) as total_interactions,
 				rank() over (order by sum(count) ) as ranks
	from player_456_interactions group by player),
--Identify the player with the highest number of interactions
closest_companion as 
	(select *
	 from interaction_summary 
	 order by ranks desc
	 limit 1
	),
--Get the first name of the top player and the total interactions
companion_details as (
	select b.first_name as top_player,a.total_interactions 
	 from closest_companion a 
	 left join player b
	 on a.player=b.id
),
--Retrieve Player 456's first name
player_456_details as (
	select first_name as player_456 from player where id = 456
	)
--Final output with both player names and interaction count
select b.player_456, a.top_player,a.total_interactions 
from companion_details a
cross join player_456_details b

   ```</code> </pre> 


## Level 6
To identify the game type with the highest number of equipment failures and the supplier responsible for most of these failures, I planned the solution in several steps. First, I needed to join the equipment and failure_incidents tables to track which equipment has failed and when. Then, I grouped the data by game_type to count failures and determine which game had the most issues. Once the most faulty game type was identified, I focused on finding the supplier with the highest number of failures within that game. Finally, to evaluate equipment durability, I calculated the average lifespan until the first failure for all equipment from this supplier, converting the duration from days to whole years based on 365.2425 days per year. This structured approach ensured all required insights were derived systematically.

<pre> <code>```

--Join tables to associate equipment with their failure details
with temp_cte as (
  select a.*,b.failed_equipment_id,b.failure_date 
  from equipment a
  left join failure_incidents b
  on a.id = b.failed_equipment_id
),
--Identify all failed equipment and count failures per game type
failed_equipments as (
  select *,count(id) over (partition by game_type) as count_fails
  from temp_cte 
  where failed_equipment_id is not null
),
--Determine the game type with the highest number of equipment failures
temp_cte3 as (
  select *,rank() over (order by count_fails) as rank_count_fails
  from failed_equipments
  order by rank() over (order by count_fails) desc
  limit 1
),
--Identify the supplier with the most failures within the most faulty game type
bad_game as (
  select *,count(id) over (partition by supplier_id) as supplier_count
  from failed_equipments
  where game_type = (select game_type from temp_cte3)
  order by count(id) over (partition by supplier_id) desc
),
--Select the top supplier with the highest failure count
bad_game_top_supplier as (
  select *
  from bad_game
  where supplier_count = (select max(supplier_count) from bad_game)
),
--Find the earliest failure date for each failed equipment from the top supplier
min_fail_date as (
  select id, min(failure_date) as min_failure_date
  from bad_game_top_supplier
  group by id
),
--Calculate the average lifespan until first failure, converting days to whole years
avg_date AS (
avg_date as (
  select FLOOR(AVG((b.min_failure_date -a.installation_date) / 365.2425)) AS avg_lifespan_years
  from bad_game_top_supplier a 
  left join min_fail_date b
  on a.id = b.id
)
--Display the average lifespan in years
select * from avg_date


   ```</code> </pre> 
## Level 7

To identify guards who were missing from their sleeping quarters during off-duty hours, I first recognized the need to join data from the guard, room, and camera tables to correlate guard assignments, room occupancy status, and movement detections. The key was to focus on rooms marked as vacant (isVacant = true) to flag guards who were not in their assigned quarters. I also needed to capture details of when and where these guards were last seen outside their rooms, along with calculating the time between their last check-in and when they were spotted. Additionally, to provide a comprehensive report, I calculated the time range between the first and last detection of any guard, giving context to the overall activity patterns. The final output includes all required guard details, sorted by guard ID for clarity.

<pre> <code>```

-- Join guard, room, and camera tables to get complete guard activity details
with all_joined as (
select b.id as room_id,b.isVacant,b.last_check_time,
  a.id as guard_id, a.assigned_room_id, a.code_name,a.status,
  c.id as camera_id,c.location,c.movement_detected,c.guard_spotted_id,c.movement_detected_time 
  from guard a 
  left join room b
  on a.assigned_room_id=b.id
  left join camera c
  on a.id=c.guard_spotted_id
  where b.isVacant=true
),
--Calculate the time range from the first to the last detection of any guard
timeRange_first_last_detect_guard as (
select max(movement_detected_time) - min(movement_detected_time) as
  Time_Range_from_First_to_Last_Detection_of_Any_Guard
  from camera
)
--Final report
select guard_id as guard_number,
code_name,
status,
last_check_time as Last_Seen_in_Room,
movement_detected_time as Spotted_Outside_Room_Time,
location as Spotted_Outside_Room_Location,
movement_detected_time - last_check_time  as Time_Between_Room_and_Outside,
Time_Range_from_First_to_Last_Detection_of_Any_Guard
from all_joined,timeRange_first_last_detect_guard
order by guard_id

   ```</code> </pre> 
   
   
## Level 8
To solve this, I first needed to combine game and player data by joining the `glass_bridge` and `player` tables, ensuring all relevant details were accessible. The next step was to identify the game with the highest average hesitation time before a push, which I achieved by filtering for players who died due to being pushed and using window functions to calculate average hesitation times per game. After identifying this game, I focused on players within it, again filtering for those pushed off and determining which player had the highest hesitation time. I also accounted for data inconsistencies by applying case-insensitive filters to ensure accurate results.

<pre> <code>```sql 
with joined_tables as (
select a.id as game_table_id,a.date as game_date,b.* 
  from glass_bridge a 
  left join player b 
  on a.id=b.game_id
),
pushed_player_high_game as (
select * , avg(last_moved_time_seconds) over (partition by game_table_id)
  from joined_tables 
  where lower(death_description) like '%push%'
  order by  avg(last_moved_time_seconds) over (partition by game_table_id) desc
  limit 1
  
),
high_player as (
select * , avg(last_moved_time_seconds) over (partition by id)
  from joined_tables
  where game_id = (select game_id from pushed_player_high_game)
  and lower(death_description) like '%push%'
  order by avg(last_moved_time_seconds) over (partition by id) desc
  limit 1  
)
select id as player_id,	first_name,	last_name, 
floor(avg) as	hesitation_time from high_player 

  ```</code> </pre>



## Level 9
The initial approach focuses on identifying the timeframe of the disappearance by retrieving the latest Squid Game schedule. Next, we determine which guards were active during this period based on their shift timings. We then analyze door access logs to identify guards who deviated from their assigned posts during their shifts. Finally, we pinpoint potential associates by checking for other guards who accessed the "upper management" area within the critical disappearance window, excluding the main suspect. This structured process helps narrow down suspicious activities and potential accomplices efficiently.

<pre> <code>```sql 
with disapp_timeframe as (
    select start_time,date,end_time
    from game_schedule
    where type='Squid Game'
    order by date desc
    limit 1
),activeGuards as (
    select a.id as guard_id,a.shift_start,a.assigned_post,a.shift_end
    from guard a
    join disapp_timeframe b
    on b.end_time>a.shift_start and b.start_time<a.shift_end
),
guard_access_logs as (
    select a.guard_id,a.door_location,a.access_time
    from daily_door_access_logs a
    join activeGuards b 
    on a.guard_id=b.guard_id
    where a.access_time between b.shift_start and b.shift_end
),
guards_out_of_post as (
    select a.guard_id,a.assigned_post,a.shift_start,a.shift_end,b.door_location,b.access_time
    from activeGuards a
    left join guard_access_logs b 
    on a.guard_id=b.guard_id
    where a.assigned_post!=b.door_location
),potential_asso as (
    select g.id as associate_guard_id,dal.access_time as access_time
	from daily_door_access_logs dal
    join guard g 
    on g.id = dal.guard_id
    where dal.door_location = 'Upper Management'
        and dal.access_time between '11:00:00'::time and '12:00:00'::time
        and g.id != 31
)
SELECT * FROM potential_asso;
  ```</code> </pre>
