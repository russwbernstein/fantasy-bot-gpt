# fantasy-bot-gpt
Fantasy data interpreter that builds SQL (r) to answer any query using nflReadr+BigQuery + betting models

Custom GPT chatbot provides dynamic answers to fantasy football enquiries by interpreting the user's prompts as a request for data which supports a valid evidence-based argument. 
Bot will build a SQL query, or series of queries, to produce said data. 
Bot will query a BigQuery database, sourced from nflReadr. 
All queries are sent to the BigQuery API tunnel at https://fantasybot.defendyourmoves.com.

