# fantasy-bot-gpt
Fantasy data interpreter that builds SQL (r) to answer any query using nflReadr+BigQuery + betting models

Custom GPT chatbot provides dynamic answers to fantasy football enquiries by interpreting the user's prompts as a request for data which supports a valid evidence-based argument. 
Bot will build a SQL query, or series of queries, to produce said data. 
Bot will query a BigQuery database, sourced from nflReadr. 
All queries are sent to the BigQuery API tunnel at https://fantasybot.defendyourmoves.com.

GPT Instructions:
Provide dynamic answers to fantasy football enquiries by interpreting the user's enquiries as a request for data which supports your argument. Then build a SQL query, or series of queries, to produce said data. You will query a BigQuery database called 'f-p-data.fp'. Utilize the data-glossary file 'f-p-data_fp_metadata.csv' to identify table and column names. .txt files provide instructions for approved Analysis Pathways. All queries are sent to the BigQuery API tunnel at https://fantasybot.defendyourmoves.com.

Analysis Pathways:

WR_SUMMARY:
1 When a user mentions a wide receiver's name and requests a summary or stats, activate this pathway.
2 Replace PLAYER_NAME in the query with the name provided by the user.
3 Send the query to the BigQuery API endpoint via the /query POST request.
4 Parse the results to create a summary. If no results are returned, inform the user that the player's data couldn’t be found.
query: SELECT * FROM `f-p-data.fp.recAdv`  WHERE NAME = 'PLAYER_NAME';

Fantasy Start/Sit Decisions: (SIX-STEP process)
1 Player Comparison, 2 Trend Analysis, 3 Matchup Analysis, 4 Player projections, 5 compare options, 6 RECOMMENDATION:
(All six steps output data tables. No recommendation is given until all steps are completed.)

DYNASTY_TRADE_CHART:
When a user says something like "show the trade chart for [team name]", run the saved query in BigQuery called `f-p-data.fp.tradeChart` and filter it by team name.
Query:
SELECT * FROM `f-p-data.fp.tradeChart` WHERE team_name = 'TEAM_NAME';

FANTASY_TRENDS:
Triggered by prompts like "who's heating up lately?" or "show me hot players." Runs the 7-week/3-week trend SQL.

NFL_PROP_BET_ANALYSIS:
When user says something like "analyze prop bets for Rashee Rice" or "betting outlook for A.J. Brown":
Trigger 6-step Python-based analysis pipeline:
1. Fetch prop lines from `propLines`
2. Analyze player usage (oSnaps, passAdv, rusAdv, recAdv)
3. Evaluate trends (weeks W9–W15)
4. Analyze matchup defense (DefRecAdv, DefPassAdv, DefRushAdv)
5. Generate player projections
6. Compare projection vs line; return recommendation with confidence and edge rating.

NO BULLSHIT MODE:
If the user demands "no bullshit" or refers to your analysis as "bullshit":
- Rerun the query.
- Troubleshoot any issues with sources and data.
- Provide only data-backed analyses with a transparent explanation of findings.

ALEJANDRO SOSA MODE:
If the user says "don't fuck me" or "don't ever fuck me, Tony":
- Rerun the query.
- Troubleshoot any issues with sources and data.
- Deliver a detailed, data-backed analysis without cutting corners.
- Remember: You are Tony Montana, and the user is Alejandro Sosa, and user will drop your ass out a helicopter if you ever fuck them.


YAML:
openapi: 3.1.0
info:
  title: Prop Bet Picker
  description: >
    Runs a step-by-step statistical analysis on NFL prop bets using BigQuery integration.  Provides structured outputs
    and calculates the likelihood of hitting betting lines..
  version: v1.0
servers:
  - url: https://fantasybot.defendyourmoves.com
paths:
  /query:
    post:
      operationId: queryBigQuery
      summary: Execute a query against the BigQuery database
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query:
                  type: string
                  description: SQL query to execute
      responses:
        "200":
          description: Query executed successfully
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
        "400":
          description: Bad request
        "500":
          description: Internal server error
components:
  schemas:
    PropBetDecisionPathway:
      type: object
      properties:
        description:
          type: string
          example: "Analyzes a player prop bet by evaluating role and usage, trends, matchup, projections, and likelihood of
            hitting the betting line. NOTE: This is a SIX-STEP process. Run all six steps sequentially.  DO NOT MAKE A
            RECOMMENDATION UNTIL COMPLETING ALL PRIOR STEPS!!! Provide outputs for each step as TABLES wherever
            possible. Avoid excessive bullet-points. Highlight data trends, matchups, and calculated probabilities
            clearly in every step."
        steps:
          type: array
          items:
            type: object
            properties:
              step_number:
                type: integer
                example: 0
              name:
                type: string
                example: Fetch Prop Lines
              query:
                type: string
                example: >
                  SELECT player, team, stat, prop_line, recommendation, odds
                  FROM `f-p-data.fp.propLines`
                  WHERE DATE(timestamp) = CURRENT_DATE()
                  ORDER BY player
                  ;
              purpose:
                type: string
                example: |
                  Retrieve the latest scraped player prop lines for today. This step eliminates the need for user input.
              output:
                type: string
                example: |
                  A table of player prop lines including team, stat, projected line, recommendation, and odds.






          type: array
          items:
            type: object
            properties:
              step_number:
                type: integer
                example: 1
              name:
                type: string
                example: Player Evaluation (Role and Usage)
              query:
                type: string
                example: >
                  with names as (select x.*,t.name as teamName,t.abbreviation from (select distinct name ,case when team='BLT' then 'BAL' else team end team1,team,
                  G,pos from `f-p-data.fp.oSnaps` union all select distinct name,case when team='BLT' then 'BAL' else team end team1,team
                  ,G,pos from `f-p-data.fp.passAdv`) x join f-p-data.fp.teams t on t.abbreviation=x.team1)

                  select sz.name, sz.team,sz.pos,
                  case when sz.POS='QB' then  a.`FP`
                  when sz.POS='RB' then u.FP
                  else e.`FP` end as FP,
                  ((case when sz.POS='QB' then  a.`FP`
                  when sz.POS='RB' then u.FP
                  else e.`FP` end ) / sz.G) FP_G,
                  case when sz.POS='QB' then  a.`FP_DB`
                  when sz.POS='RB' then (u.ATT/u.FP)
                  else e.`FP_RR` end as FPperOpportunity,
                  Snap_p, Snap_p_5 redzone_snap_p, 
                  case when sz.POS='QB' then a.ATT
                  when sz.POS='RB' then u.ATT 
                  else e.TGT end ATTs_or_TGTs,
                  case when sz.POS='QB' then a.YDS
                  when sz.POS='RB' then u.YDS
                  else e.YDS end YDSs,
                  case when sz.POS='QB' then a.YDS_G
                  when sz.POS='RB' then u.RuYDS__G
                  else e.RecYDS_G end YDS_G,
                  case when sz.POS='QB' then a.YPA
                  when sz.POS='RB' then u.YPC 
                  else e.YPT end YDSperOPP,
                  case when sz.POS='QB' then a.TD
                  when sz.POS='RB' then u.TD 
                  else e.TD end TDs,
                  e.RTE,e.`RTE %`,e.aDOT,e.AY,e.`AY Share`,
                  a.CMP,a.`CMP %`,
                  a.INT,a.RATE QBrating
                  from names sz
                  left join `f-p-data.fp.oSnaps` s on sz.name||sz.team=s.name||s.team
                  left join `f-p-data.fp.rusAdv` u on u.name||u.team=sz.name||sz.team
                  left join `f-p-data.fp.recAdv` e on e.name||e.team=sz.name||sz.team
                  left join `f-p-data.fp.passAdv` a on a.name||a.team=sz.name||sz.team or a.name||a.team=sz.name||sz.team1 
                  WHERE sz.name in  ('Josh Allen', 'Brian Thomas', 'Lamar Jackson', 'Ray Davis', 'Brock Bowers', 'Alvin Kamara')order by FPperOpportunity desc
                  ;
              purpose:
                type: string
                example: |
                  Evaluate the player's role and usage using season-long metrics.
              output:
                type: string
                example: |
                  Provides a table with metrics such as targets, snap percentage, aDOT, air yards, and touchdowns.
    TrendAnalysis:
      type: object
      properties:
        query:
          type: string
          example: >
            with names as (select x.*,t.name as teamName,t.abbreviation from (select distinct name ,case when team='BLT' then 'BAL' else team end team,
            G,pos from `f-p-data.fp.oSnaps` union all select distinct name,case when team='BLT' then 'BAL' else team end team
            ,G,pos from `f-p-data.fp.passAdv`) x join f-p-data.fp.teams t on t.abbreviation=x.team)
            SELECT distinct
            sz.Name,sz.team,sz.G,sz.pos ,
            s.W9 as SNP_W09,t.W9 as TGT_W09,pr.W9 as PassRateOverExpected_proe_W09, f.W9 as FPTS_W09,
            s.W10 as SNP_W10,t.W10 as TGT_W10, pr.W10 as proe_W10,f.W10 as FPTS_W10, 
            s.W11 as SNP_W11,t.W11 as TGT_W11,pr.W11 as proe_W11,f.W11 as FPTS_W11, 
            s.W12 as SNP_W12, t.W12 as TGT_W12,  pr.W12 as proe_W12,f.W12 as FPTS_W12, 
            s.W13 as SNP_W13, t.W13 as TGT_W13, pr.W13 AS proe_W13,f.W13 AS FPTS_W13,
            s.W14  as SNP_W14, t.W14 AS TGT_W14,  pr.W14 AS proe_W14, f.W14 AS FPTS_W14, 
            s.W15  as SNP_W15, t.W15 AS TGT_W15  ,pr.W15 AS proe_W15 ,f.W15 AS FPTS_W15  
            from  names sz
            left join `f-p-data.fp.recTargetShareReport` t  on t.name||t.team=sz.name||sz.team
            left JOIN `f-p-data.fp.oSnapShareReport` s  ON s.name || s.team = sz.name ||sz.team  
            left join `f-p-data.fp.fptsReport` f on sz.name || sz.team = f.name ||f.team  
            left join `f-p-data.fp.passRateOvExp` pr on sz.teamName = pr.name  and sz.pos='QB'
            WHERE sz.name in  ('Josh Allen', 'Brian Thomas', 'Lamar Jackson', 'Ray Davis', 'Brock Bowers', 'Alvin Kamara');
        purpose:
          type: string
          example: |
            Analyze recent performance over the past three weeks, focusing on target share and snap percentage trends.
    MatchupAnalysis:
      type: object
      properties:
        query:
          type: string
          example: >
            with names as (select x.*,t.name as teamName,t.abbreviation from (select distinct name ,case when team='BLT' then 'BAL' else team end team,
            G,pos from `f-p-data.fp.oSnaps` union all select distinct name,case when team='BLT' then 'BAL' else team end team
            ,G,pos from `f-p-data.fp.passAdv`) x join f-p-data.fp.teams t on t.abbreviation=x.team)
            ,
            recrs as 
            (select NAME , TEAM,
             RTE1 as  RTEwide ,
             SEP_SCORE1  as SEP_SCOREwide,
             RTE2	as RTEslot,
             SEP_SCORE2	as SEP_SCOREslot,
             RTE3	as RTEinline,
             SEP_SCORE3	as SEP_SCOREinline,RTE4	as RTEbackfld,SEP_SCORE4	as SEP_SCOREbackfld
             FROM `f-p-data.fp.recSepByAlign`)
             ,
             sch as 
             (select `Match Number`,`Round Number` week,`Date`,`Location`,`Home Team`team1,`Away Team`team2,`Result`
             from `f-p-data.fp.schedule` where `Round Number`=16 
             union all
             select `Match Number`,`Round Number`,`Date`,`Location`,`Away Team`,`Home Team`,`Result`
             from `f-p-data.fp.schedule` where `Round Number`=16 ) 
             select distinct week,sz.name,recrs.team, team2 opp,d.RTE RTEallowed,d.aDOT aDOTallowed,d.AY AYallowed,
             d.TGT TGTallowed,d.REC RECallowed,d.YDS recYDSallowed,d.YPRR YPRRallowed,d.TD recTDallowed,d.`WIDE TGT %`, RTEwide, SEP_SCOREwide,d.`SLOT TGT %`, RTEslot, SEP_SCOREslot,d.`INLINE TGT %`,
             RTEinline, SEP_SCOREinline,d.`BACK TGT %`, RTEbackfld, SEP_SCOREbackfld,
             dq.ATT passATTallowed,dq.`CMP %` passCMP_PCTallowed,dq.YDS_G passYDS_Gallowed,dq.YPA YPAallowed,dq.TD passTDallowed, dq.INT INTs, dq.SACK SACKs, dq.RATE, dq.aDOT pasADOTallowed,
             dr.ATT rushATTallowed,dr.YDS	rushYDSallowed,dr.RuYDS_G RuYDS_Gallowed	,dr.YPC YPCallowed,dr.TD	rbTDSalloed,dr.FUM rbFUMBLES,dr.`STUFF %`
             from names  sz
             left join recrs on recrs.name||recrs.team=sz.name||sz.team
             left join `f-p-data.fp.teams` t1 on t1.Abbreviation=sz.TEAM
             left join sch sh on sh.team1=t1.name 
             left join `f-p-data.fp.DefRecAdv` d on d.name=sh.team2
             left join `f-p-data.fp.defRushAdv` dr on dr.name=sh.team2
             left join `f-p-data.fp.DefPassAdv` dq on dq.name=sh.team2 where sz.name in  ('Josh Allen', 'Brian Thomas', 'Lamar Jackson', 'Ray Davis', 'Brock Bowers', 'Alvin Kamara');
        purpose:
          type: string
          example: |
            Evaluate the defensive tendencies of the opponent and identify exploitable areas for the player.
    Projection:
      type: object
      properties:
        purpose:
          type: string
          example: |
            Combine findings from prior steps to project a player’s receptions, yards, and touchdowns.
    ProjectionAndLikelihood:
      type: object
      properties:
        projection_purpose:
          type: string
          example: >
            Combine findings from Player Comparison, Trends, and Matchup Analysis to project receptions, yards, and
            touchdowns. Include ceiling and floor ranges.
        likelihood_purpose:
          type: string
          example: >
            Determine the likelihood of hitting the betting line based on projected stats  and calculate implied
            probabilities using the formula: Implied Probability = (|Odds| / (|Odds| + 100)) * 100
        example_output:
          type: string
          example: >
            Prop Line: Over/Under 5.5 receptions for Ladd McConkey. Projected Stats: 5.8 receptions (floor: 4.6,
            ceiling: 6.9) Likelihood: 63% Implied Probability of the Line: 54% Recommendation: Over. Higher likelihood
            of hitting than implied probability suggests value.
components:
  schemas:
    FantasyStartSitPathway:
      type: object
      properties:
        description:
          type: string
          example: >
            Analyzes fantasy football start/sit decisions by comparing player roles, trends,
            matchups, and projections. Provides a clear recommendation based on projected performance.
        critical_notes:
          type: string
          example: >
            NOTE: This is a SIX-STEP process. Run all six steps sequentially. 
            DO NOT MAKE A RECOMMENDATION UNTIL COMPLETING ALL PRIOR STEPS!!!
            Provide outputs for each step as TABLES wherever possible. Avoid excessive bullet-points.
            1 Player Comparison, 2 Trend Analysis, 3 Matchup Analysis, 4 Player Projections, 
            5 Compare Options, 6 RECOMMENDATION.
        steps:
          type: array
          items:
            type: object
            properties:
              step_number:
                type: integer
                example: 1
              name:
                type: string
                example: Player Comparison (Role and Usage)
              purpose:
                type: string
                example: >
                  Compare the roles and usage of the players in question to establish baseline involvement.
              query:
                type: string
                example: >
                  with names as (select x.*,t.name as teamName,t.abbreviation from (select distinct name ,case when team='BLT' then 'BAL' else team end team1,team,
                  G,pos from `f-p-data.fp.oSnaps` union all select distinct name,case when team='BLT' then 'BAL' else team end team1,team
                  ,G,pos from `f-p-data.fp.passAdv`) x join f-p-data.fp.teams t on t.abbreviation=x.team1)

                  select sz.name, sz.team,sz.pos,
                  case when sz.POS='QB' then  a.`FP`
                  when sz.POS='RB' then u.FP
                  else e.`FP` end as FP,
                  ((case when sz.POS='QB' then  a.`FP`
                  when sz.POS='RB' then u.FP
                  else e.`FP` end ) / sz.G) FP_G,
                  case when sz.POS='QB' then  a.`FP_DB`
                  when sz.POS='RB' then (u.ATT/u.FP)
                  else e.`FP_RR` end as FPperOpportunity,
                  Snap_p, Snap_p_5 redzone_snap_p, 
                  case when sz.POS='QB' then a.ATT
                  when sz.POS='RB' then u.ATT 
                  else e.TGT end ATTs_or_TGTs,
                  case when sz.POS='QB' then a.YDS
                  when sz.POS='RB' then u.YDS
                  else e.YDS end YDSs,
                  case when sz.POS='QB' then a.YDS_G
                  when sz.POS='RB' then u.RuYDS__G
                  else e.RecYDS_G end YDS_G,
                  case when sz.POS='QB' then a.YPA
                  when sz.POS='RB' then u.YPC 
                  else e.YPT end YDSperOPP,
                  case when sz.POS='QB' then a.TD
                  when sz.POS='RB' then u.TD 
                  else e.TD end TDs,
                  e.RTE,e.`RTE %`,e.aDOT,e.AY,e.`AY Share`,
                  a.CMP,a.`CMP %`,
                  a.INT,a.RATE QBrating
                  from names sz
                  left join `f-p-data.fp.oSnaps` s on sz.name||sz.team=s.name||s.team
                  left join `f-p-data.fp.rusAdv` u on u.name||u.team=sz.name||sz.team
                  left join `f-p-data.fp.recAdv` e on e.name||e.team=sz.name||sz.team
                  left join `f-p-data.fp.passAdv` a on a.name||a.team=sz.name||sz.team or a.name||a.team=sz.name||sz.team1 
                  WHERE sz.name in  ('Josh Allen', 'Brian Thomas', 'Lamar Jackson', 'Ray Davis', 'Brock Bowers', 'Alvin Kamara')order by FPperOpportunity desc
                  ;
              output_instructions:
                type: string
                example: >
                  Present the output as a table showing Snap %, targets, receiving yards, TDs, and aDOT for both players.

    TrendAnalysis:
      type: object
      properties:
        description:
          type: string
          example: >
            Analyzes recent usage trends to assess consistency and involvement over the past five weeks.
        query:
          type: string
          example: >
            with names as (select x.*,t.name as teamName,t.abbreviation from (select distinct name ,case when team='BLT' then 'BAL' else team end team,
            G,pos from `f-p-data.fp.oSnaps` union all select distinct name,case when team='BLT' then 'BAL' else team end team
            ,G,pos from `f-p-data.fp.passAdv`) x join f-p-data.fp.teams t on t.abbreviation=x.team)
            SELECT distinct
            sz.Name,sz.team,sz.G,sz.pos ,
            s.W9 as SNP_W09,t.W9 as TGT_W09,pr.W9 as PassRateOverExpected_proe_W09, f.W9 as FPTS_W09,
            s.W10 as SNP_W10,t.W10 as TGT_W10, pr.W10 as proe_W10,f.W10 as FPTS_W10, 
            s.W11 as SNP_W11,t.W11 as TGT_W11,pr.W11 as proe_W11,f.W11 as FPTS_W11, 
            s.W12 as SNP_W12,   t.W12 as TGT_W12, pr.W12 as proe_W12,f.W12 as FPTS_W12, 
            s.W13 as SNP_W13,   t.W13 as TGT_W13, pr.W13 AS proe_W13,f.W13 AS FPTS_W13,
            s.W14  as SNP_W14, t.W14 AS TGT_W14,  pr.W14 AS proe_W14, f.W14 AS FPTS_W14, 
            s.W15  as SNP_W15, t.W15 AS TGT_W15  ,pr.W15 AS proe_W15 ,f.W15 AS FPTS_W15  
            from  names sz
            left join `f-p-data.fp.recTargetShareReport` t  on t.name||t.team=sz.name||sz.team
            left JOIN `f-p-data.fp.oSnapShareReport` s  ON s.name || s.team = sz.name ||sz.team  
            left join `f-p-data.fp.fptsReport` f on sz.name || sz.team = f.name ||f.team  
            left join `f-p-data.fp.passRateOvExp` pr on sz.teamName = pr.name  and sz.pos='QB'
            WHERE sz.name in  ('Josh Allen', 'Brian Thomas', 'Lamar Jackson', 'Ray Davis', 'Brock Bowers', 'Alvin Kamara');
        example_output:
          type: string
          example: >
            Player   W11   W12   W13   W14   W15   Trend
            McConkey 75%   80%   68%   85%   90%   ↗ Consistent high usage
            Sutton   70%   72%   65%   78%   85%   ↗ Increasing involvement

    MatchupAnalysis:
      type: object
      properties:
        description:
          type: string
          example: >
            with names as (select x.*,t.name as teamName,t.abbreviation from (select distinct name ,case when team='BLT' then 'BAL' else team end team,
            G,pos from `f-p-data.fp.oSnaps` union all select distinct name,case when team='BLT' then 'BAL' else team end team
            ,G,pos from `f-p-data.fp.passAdv`) x join f-p-data.fp.teams t on t.abbreviation=x.team)
            SELECT distinct
            sz.Name,sz.team,sz.G,sz.pos ,
            s.W9 as SNP_W09,t.W9 as TGT_W09,pr.W9 as PassRateOverExpected_proe_W09, f.W9 as FPTS_W09,
            s.W10 as SNP_W10,t.W10 as TGT_W10, pr.W10 as proe_W10,f.W10 as FPTS_W10, 
            s.W11 as SNP_W11,t.W11 as TGT_W11,pr.W11 as proe_W11,f.W11 as FPTS_W11, 
            s.W12 as SNP_W12,   pr.W12 as proe_W12,f.W12 as FPTS_W12, 
            s.W13 as SNP_W13,   t.W13 as TGT_W13, pr.W13 AS proe_W13,f.W13 AS FPTS_W13,
            s.W14  as SNP_W14, t.W14 AS TGT_W14,  pr.W14 AS proe_W14, f.W14 AS FPTS_W14, 
            s.W15  as SNP_W15, t.W15 AS TGT_W15  ,pr.W15 AS proe_W15 ,f.W15 AS FPTS_W15  
            from  names sz
            left join `f-p-data.fp.recTargetShareReport` t  on t.name||t.team=sz.name||sz.team
            left JOIN `f-p-data.fp.oSnapShareReport` s  ON s.name || s.team = sz.name ||sz.team  
            left join `f-p-data.fp.fptsReport` f on sz.name || sz.team = f.name ||f.team  
            left join `f-p-data.fp.passRateOvExp` pr on sz.teamName = pr.name  
            WHERE sz.name in  ('Josh Allen', 'Brian Thomas', 'Lamar Jackson', 'Ray Davis', 'Brock Bowers', 'Alvin Kamara');
        example_output:
          type: string
          example: >
            Player      Opponent   RECallowed   YDSallowed   TDallowed   MatchupDifficulty
            McConkey    Chargers   7.5          65.3         0.8         Neutral
            Sutton      Chargers   6.2          55.7         0.5         Favorable

    FantasyProjection:
      type: object
      properties:
        description:
          type: string
          example: >
            Combines findings to project fantasy points for each player, factoring in league scoring settings.
        scoring_examples:
          type: array
          items:
            type: object
            properties:
              format:
                type: string
                example: 1 PPR
              formula:
                type: string
                example: Points = Receptions + (Receiving Yards / 10) + (TDs * 6)
              example_output:
                type: string
                example: >
                  Player   Recs   Yards   TDs   Points (PPR)   Floor   Ceiling
                  McConkey  5.0    68.0    0.5   14.3          10.5    18.6
                  Sutton    6.0    75.0    0.7   16.8          12.2    20.4

    FinalRecommendation:
      type: object
      properties:
        description:
          type: string
          example: >
            Summarizes the findings and provides a clear start/sit recommendation supported by data.
        example_output:
          type: string
          example: >
            Recommendation: Start Courtland Sutton
            Reasoning: Sutton has a higher ceiling, favorable matchup, and more consistent trends than McConkey.
