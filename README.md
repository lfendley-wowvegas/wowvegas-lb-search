# WOW Vegas Hourly LB Searcher
# Vue 3 + TypeScript + Vite

A simple search tool to query WOW Vegas (Active) Hourly Races to find if/where username placed.

LB data from https://cms4.wowvegas.com/api/ams/v1/leaderboards

Query will iterate through ACTIVE LBs and search for the previous hour down till midnight, so if querying at 4:55pm it will query 4pm, 3pm, 2pm, etc
