# Olist Data Project — Summary for Stakeholders

## What we did

We took Olist's raw order, customer, product and payment data — spread across 9 separate files — and built 
it into a structured data pipeline on Databricks. The data now flows through three stages:

1. **Raw** — the original files, untouched, just landed in one place
2. **Cleaned** — types fixed, duplicates removed, messy rows handled properly
3. **Business-ready** — organized into a format built for answering questions fast, not just storing data

This means any future question about this data — "how did we do last quarter," "which region is 
underperforming," "who are our top sellers" — can be answered in minutes with a query, instead of someone 
manually digging through raw CSV files.

## What we found

**1. Shipping costs are hurting sales in certain regions.**  
Customers in remote states — Roraima, Maranhão, Rondônia, Amazonas, Tocantins — pay close to double the 
shipping cost (relative to what they're buying) compared to customers near São Paulo. 28% of the item price 
in the worst case, versus 14% near São Paulo. These same states also have by far the lowest order volumes. 
The two are likely connected — high shipping cost is a common reason people abandon a purchase.

**2. Most customers only buy once.**  
Out of roughly 99,000 orders, only about 3,345 unique customers — just 3.4% — ever came back for a second 
purchase. This tells us that loyalty or rewards programs won't have much room to work with here. The bigger 
opportunity is bringing in new customers and reducing friction in how orders get delivered, not trying to 
retain existing ones.

**3. Not all product categories are equal.**  
Health & beauty products generate more total revenue than any other category, even though bed/bath/table 
items sell in slightly higher volume. Revenue and popularity aren't always the same thing, and it's worth 
keeping that distinction in mind for where marketing effort goes.

## What we recommend

Run a small pilot: cap shipping cost at a fairer rate (roughly matching what São Paulo customers already 
pay) for the five worst-affected states, for about 6-8 weeks. Track whether order volume grows faster in 
those states compared to a few similar states that don't get the discount.

If order volume grows noticeably more in the discounted states, that's a real signal that shipping cost was 
holding back sales in those regions, and it's worth expanding the program. If it doesn't move much, we've 
ruled out an idea cheaply, with a small and controlled cost, and can look elsewhere.

## Why this matters going forward

The pipeline built here isn't a one-time report — it's reusable infrastructure. The next business question 
about this data shouldn't take another two-day project to answer; it's now mostly a matter of writing a new 
query against data that's already clean, organized, and ready to use.

## A note on confidence

Given the time-box on this project, there are a couple of places I'd want more validation before treating 
them as settled: the expected impact of the shipping subsidy (estimated, not modeled from historical 
elasticity), and the outlier detection method (a more robust statistical approach might change which orders 
get flagged). I've called these out explicitly rather than presenting rough estimates as precise numbers.