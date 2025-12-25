# Amazon Big Data Challenge: My First Machine Learning Project

## 📌 Project Overview
This project represents a personal milestone: **My first attempt at building a Machine Learning model completely within a Cloud Data Warehouse.**

To challenge myself technically, I ingested the massive "Amazon US Customer Reviews" dataset (~54GB, 37 categories) into **Google BigQuery**. My goal was not just to analyze the data, but to engineer a full pipeline—from answering ad-hoc "stakeholder" questions to training a Boosted Tree Regressor using only SQL.

## 🎯 Objectives
* **Handle Big Data:** Work with a dataset (54GB) too large for standard local memory, utilizing cloud infrastructure.
* **Simulate Stakeholder Requests:** Interview family members to gather "business questions" and translate them into SQL queries.
* **Build an ML Model:** Create, train, and evaluate a regression model directly inside BigQuery (BQML) to predict product ratings.

---

## 🗣️ Part 1: The "Stakeholder" Analysis
To simulate a real-world business environment, I treated my family members as stakeholders. I interviewed them about their current shopping needs and translated their "natural language" questions into precise SQL queries.

### 🚲 Stakeholder 1: The Father (Value Analysis)
* **The Request:** *"I need new wheels for my bike. I don't want to buy 'cheap junk,' but I don't want to overpay. Where is the sweet spot?"*
* **SQL Strategy:**
    * **Precision Filtering:** Used `REGEXP_CONTAINS` to isolate "Bike Wheels" while explicitly excluding 30+ accessory keywords (e.g., *tubes, caps, covers, repair kits*) to ensure clean data.
    * **Value Hunting:** Instead of just sorting by price, I searched the `review_body` for semantic value signals using the regex pattern: `r'(?i)good value|great price|bang for the buck|worth the money'`.
    * **Ranking:** Ranked products by the volume of these "value-confirming" reviews rather than raw star rating alone.
* **Key Finding:** Identified top-performing wheelsets that maintained >4.5 stars while remaining in the lower 30th percentile of pricing.

### 👶 Stakeholder 2: The Sister (Seasonality & Trends)
* **The Request:** *"I buy a lot of baby clothes. Is there a specific time of year where prices/activity seem to drop?"*
* **SQL Strategy:**
    * **Temporal Aggregation:** Used `EXTRACT(MONTH FROM review_date)` on the `Baby` category to aggregate millions of rows into annual trends.
    * **Noise Reduction:** Filtered specifically for `verified_purchase = true` to remove bot traffic and identify genuine seasonal purchasing spikes.
* **Key Finding:** The data revealed distinct seasonal dips, allowing for recommendations on the optimal months to stock up on essentials.

### 💄 Stakeholder 3: The Partner (Product Discovery)
* **The Request:** *"I want to try new makeup and beauty products, but I'm overwhelmed by the options. What are the safest 'sure bets' to try?"*
* **SQL Strategy:** I queried the `Beauty` category to find "Crowd Favorites." I filtered for products with high "Verified Purchase" counts and low variance in ratings (standard deviation) to identify products that consistently perform well for a wide audience, rather than niche items.
* **Key Finding:** Generated a curated list of "Holy Grail" products that have maintained >4.8 stars over 1,000+ verified reviews.

*(See `sql_queries/family_questions.sql` for the complete analysis code)*

---

## 🧠 Part 2: My First Machine Learning Model
For the second phase, I wanted to see if I could predict a review's **Star Rating** based on the review's content and context.

* **Model Type:** Boosted Tree Regressor (BigQuery ML)
* **Target:** `star_rating` (1-5)

### 🛠 Feature Engineering (The "Secret Sauce")
Since raw text cannot be fed directly into this model, I used SQL `REGEXP` functions to engineer a custom **"Sentiment Score"** feature.

```sql
-- My Custom Sentiment Formula
(
  -- Count positive words (love, great, excellent...)
  IFNULL(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(LOWER(review_body), r'love|great|excellent|amazing...')), 0)
  - 
  -- Subtract negative words (hate, worst, terrible, bad...)
  IFNULL(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(LOWER(review_body), r'hate|worst|terrible|bad...')), 0)
) as sentiment_score
