# Analysis & Commercial Recommendation

## B1 — Commercial Insights

### Insight 1: Health & beauty makes the most money, but bed/bath/table sells more items

When I grouped revenue by category, health_beauty came out on top at R$1.41M, even though bed_bath_table 
actually sold more items (10,953 vs 9,465). So health & beauty products are just priced higher on average — 
revenue and "most popular" aren't the same thing here.

**How I got this:** summed item_total grouped by category, joined through the product table to get the 
English category names (the raw data only has Portuguese names).

**Why it matters:** if the business was going to push marketing spend toward "best sellers," it'd probably 
default to bed_bath_table since it has more orders. But health_beauty is actually worth more per sale.

### Insight 2: Customers far from São Paulo are paying way more in shipping, relative to what they're buying

I looked at freight cost as a percentage of item price, broken down by customer state. In São Paulo it's 
about 14%. In Roraima it's 28% — literally double. Maranhão, Rondônia, Amazonas and Tocantins are all above 
24% too. And unsurprisingly, these are also the states with the lowest order volume.

**How I got this:** summed freight and item price by customer_state from the gold fact table, then took the 
ratio.

**Why it matters:** I can't prove causation from this alone, but it's a pretty reasonable guess that if 
you're paying double the freight relative to what you're buying, you're less likely to buy in the first 
place. This felt like the most interesting finding in the whole dataset.

### Insight 3: Repeat customers are rare, so I didn't go down the RFM/loyalty route

I checked how many customers ordered more than once. Out of 99,441 orders, only 96,096 are unique people 
(customer_unique_id) — so only around 3,345 customers, about 3.4%, ever came back for a second order.

**How I got this:** this took me a bit to get right — Olist's data actually splits customer_id (one per 
order) from customer_unique_id (one per actual person). I had to build a separate bridge table for this in 
silver so I wasn't accidentally treating every order as a different customer.

**Why it matters:** with a repeat rate this low, something like RFM segmentation or a loyalty tier system 
wouldn't have much to work with — there just aren't enough repeat customers for it to be meaningful. I think 
this is honestly a more useful insight than doing RFM anyway and getting a weak result, since it tells you 
where NOT to spend effort.

---

## B2 — The Recommendation

**Who:** Customers in the 5 worst states for freight ratio — Roraima, Maranhão, Rondônia, Amazonas, and 
Tocantins. I picked these directly off the state breakdown query (all above ~24% freight-to-price ratio, vs 
~14% in São Paulo).

**What:** Cap freight cost at around 15% of item price for orders from these states for a trial period — 
basically match what São Paulo customers already pay, with the company covering the difference.

**Expected impact:** These 5 states combined only made about R$270K in revenue in the whole dataset, which 
is tiny compared to São Paulo's R$5.77M. I'm assuming a 15-20% lift in orders if freight friction goes down 
— I want to be upfront that this is a rough guess, not based on an actual pricing/elasticity model, since I 
didn't have time to build one. But even if I'm off, the subsidy cost is capped and easy to track, so the 
downside is limited.

**How I'd measure it:** Run it for 6-8 weeks, track weekly order volume in the 5 states, and compare against 
a few other small states that didn't get the subsidy (like Paraíba or Piauí) as a control. If the subsidized 
states grow faster than the control ones, that's a real signal freight was actually the blocker. If not, at 
least we've ruled it out cheaply.

---

## B3 — Assumptions, Caveats & Data Quality

**Assumptions I made on the messy stuff:**
- I let Databricks auto-infer the schema at bronze, then typed everything properly (dates, decimals) once I 
  got to silver.
- Used customer_unique_id as "the real customer" everywhere, and customer_id only for linking orders back to 
  people — this is the trap I mentioned above.
- Found 9 review rows where order_id itself was NULL — turned out to be a CSV parsing issue where comment 
  text with commas in it was shifting all the other columns over. Dropped these since there's no order to 
  attach them to anyway.
- Found another 57 rows where just the timestamp was broken from the same issue — kept these ones since the 
  actual review score and order_id were fine, just set the broken timestamp to NULL instead of throwing the 
  whole row away.
- Some reviews had the same order_id twice — kept the most recent one by timestamp.
- About 610 products (under 2%) don't have a category name at all. Kept them as valid products, they just 
  won't show up when I group by category.

**What I'd tell a client not to over-trust:**
- My freight subsidy revenue estimate (15-20% lift) is a guess, not a model. If this was a real client I'd 
  say clearly: "this needs a proper test before you commit real budget to it."
- My outlier query (the z-score one) flagged some pretty extreme values — items that cost R$0.85 with 
  R$18-22 freight attached. Technically correct, but with data this skewed, z-score isn't the most robust 
  method. A percentile cutoff would probably be cleaner. I'd treat my outlier list as "worth a second look," 
  not "definitely errors."
- All my revenue numbers only include orders marked "delivered" — I filtered out cancelled/unavailable ones, 
  which is about 2.5% of orders, so total demand is slightly higher than what I'm reporting.

**Top 3 data quality checks I'd want running if this went to production:**
1. Make sure order_id + order_item_id is actually unique in the fact table — if this breaks, every revenue 
   number downstream is wrong without anyone noticing.
2. Price, freight_value and payment_value should never be null or negative — this would catch bad data 
   before it hits any dashboard.
3. Every customer_key/product_key/seller_key in the fact table should actually exist in its dimension table 
   — otherwise you get silent join failures that quietly drop rows.

**How I'd catch the data going stale or changing shape:**
Track row counts per file each run and alert if they jump or drop a lot compared to normal. Also compare 
the incoming column list against what I expect — if a column gets renamed or removed upstream, I'd rather 
find out immediately than have my pipeline silently break or produce wrong numbers.

---

## B4 — Client Memo (for CFO/COO)

**Subject: What we built with your Olist data, and what it's telling us**

We took your raw order, customer, and product data and built it into a proper Databricks pipeline — cleaned, 
organized, and structured so it's ready to actually answer business questions, not just sit as raw files.

Two things stood out. First: customers in remote states are paying almost double the shipping cost 
(relative to what they're buying) compared to customers near São Paulo — 28% vs 14% in the worst case. Those 
same states also have the lowest order volumes, which strongly suggests shipping cost might be scaring 
customers off before they even buy. Second: only about 3% of your customers ever order twice, so loyalty 
programs probably aren't where the next win is — the bigger opportunity is in acquisition and logistics.

**What I'd do next:** run a small pilot subsidizing freight in the 5 worst-affected states for 6-8 weeks, and 
compare order growth there against similar states that don't get the subsidy. If it works, you've found a 
real lever for revenue in underserved regions. If it doesn't, you've ruled it out for a small, controlled 
cost.

This whole pipeline is reusable, so the next business question on this data shouldn't take another two days 
— it's mostly just a new query away.

---

## B5 — LLM Use Case (Bonus)

**Idea:** Use Databricks Genie so someone non-technical can just ask questions in plain English against the 
gold tables, instead of needing to write SQL every time.

**Example:** *"Which states had the highest freight-to-price ratio last quarter, and how many orders came 
from them?"*

**What it pulls:** the fact_order_items table joined to customer and date dimensions, filtered to delivered 
orders in that quarter.

**What comes back:** a short answer plus a table, and ideally the actual SQL it ran so it's not a black box.

**Biggest risk:** it could hallucinate a join or misunderstand what "revenue" means (item price alone vs 
item price + freight) and confidently give a wrong number.

**How I'd handle that:** This is actually something I've thought about before, coming from Palantir Foundry. 
In Foundry, you don't let an LLM (or even most users) query raw tables directly — you build an Ontology layer 
on top: objects like "Customer" or "Order" with their properties and relationships explicitly defined once, 
so anything built on top (including an LLM) is reasoning over defined business concepts, not guessing how 
five tables relate to each other every single time.

I'd apply the same idea here. Genie's table/column comments and example query pairs are basically a lighter 
version of that same concept — a semantic layer that removes ambiguity before the LLM ever generates SQL. 
So concretely: define "revenue" once, explicitly, as item_price + freight_value (or whichever the business 
actually means), attach that definition to the gold tables, and give Genie a handful of verified example 
question→SQL pairs for the joins that matter (customer → order → date). And always show the generated SQL 
so someone can catch a bad interpretation before it gets treated as fact.

The lesson from Foundry that carries over directly: the fix for LLM hallucination usually isn't a smarter 
prompt, it's giving the model less room to guess in the first place.