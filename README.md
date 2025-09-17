# Hourly LB Searcher
# Vue 3 + TypeScript + Vite

A simple search tool to query Actively running Hourly Races to find if/where user placed.

LB data from :
 - https://cms4.wowvegas.com/api/ams/v1/leaderboards
 - https://cms4.wowvegas.com/api/ams/v1/leaderboards/:id

Query will iterate through ACTIVE LBs and search for the previous hour down till midnight, so if querying at 4:55pm it will query 4pm, 3pm, 2pm, etc
