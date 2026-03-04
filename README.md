

# 🛒 NextCart — A Shopping Engine That Knows What's Next

Most recommendation engines tell you what's popular. **NextCart** tells you what's *next*. Built on sequential purchase patterns and price behavior, NextCart predicts the natural next step in each client's shopping journey — and surfaces the right product at the right moment.




# E-Commerce Recommender System — Solution Proposal
**Addi Technical Assessment — Shop Challenge**

This notebook documents the reasoning and architecture behind a recommender system that bridges two datasets: KZ client purchase history and an Amazon product catalog. No full production code — pseudocode where it helps clarify the logic.

---

## 1. Business Understanding

### The Problem

We have two datasets:
- **`kz.csv`** — purchase history of our clients (who they are, what they bought, when).
- **Amazon catalog** — ~10K products across 939 categories that our clients have never interacted with.

Goal: generate **top 10 Amazon product recommendations per client**.

The challenge is that these two datasets don't speak the same language — different categories, different markets, no shared client-product interactions. Bridging that gap is the core design problem.

### Key Design Decision: One Recommendation Per Client

A time-aware system would generate a new recommendation after every purchase event. That's more precise but also more complex. For this proposal, we generate **one recommendation per client based on their current state** (full history, emphasis on most recent purchase).

In production, this gets recalculated every time a client makes a new purchase — so it's effectively dynamic, just simpler to build and validate.

### What "Good" Looks Like

A good recommendation:
1. Belongs to a category the client has shown interest in, or one that logically follows from their recent purchases.
2. Matches their typical price range within that category.

---

## 2. EDA — Key Questions

Before any modeling, EDA is about validating assumptions and finding data quality issues.

### On `kz.csv`

| Question | Why it matters |
|---|---|
| How many unique clients? | Defines output scope |
| Purchase frequency distribution? | Many one-time buyers vs. recurrent? |
| How many unique categories? | ~510 expected — how concentrated are they? |
| Is there a timestamp? | Required for sequential analysis |
| Price distribution per category? | Needed for price-tier segmentation |

**Hypothesis to validate:** A small number of categories (~20%) account for most purchases (~80%). If true, we only need to map those top categories to Amazon — the long tail is noise for v1.

### On the Amazon Catalog

| Question | Why it matters |
|---|---|
| How are 939 categories structured? | Hierarchical? Redundant? |
| How many Amazon direct-seller products? | Size of the trusted pool |
| Quality of `about_product` / `product_specification`? | Input for category discovery |
| Price distribution? | Needed for price tier matching |

**Thing to watch:** A product can belong to multiple Amazon categories. We need a rule for how to handle that — take the most specific one, or use all of them as tags.

---
## 3. Feature Engineering

Three things to build before we can recommend anything.

### 3.1 Category Mapping — Bridging the Two Datasets

510 KZ categories vs. 939 Amazon categories, no shared vocabulary. The approach:

**Step 1:** Frequency analysis on KZ categories. Find the top N that cover ~80% of all purchases. This is probably 20–40 categories.

**Step 2:** For each of those, define regex keywords to match against Amazon's category column, product name, and text fields.

```
category_map = {
    "KZ_ELECTRONICS": ["phone", "smartphone", "laptop", "tablet", "cable", "charger"],
    "KZ_FASHION":     ["clothing", "shirt", "shoes", "dress", "jacket"],
    "KZ_HOME":        ["kitchen", "furniture", "home", "decor", "bedding"],
    "KZ_SPORTS":      ["sport", "fitness", "gym", "outdoor", "exercise"],
    # ... ~20 categories total
}

# Each Amazon product gets a KZ-equivalent label
# if any keyword appears in its text fields
```

**Step 3:** Spot-check a sample of mapped products. The goal isn't perfect coverage — it's good enough for the categories that actually matter.

**What if an Amazon product doesn't map to any KZ category?** It falls into a general pool. If a client's target categories have no mapped products, we fall back to the most popular KZ category that does have Amazon coverage.

**Trade-off:** This is manual and brittle to new categories. 

---

### 3.2 Category Transition Matrix (Sequential Patterns)

The core insight: what a client buys next is not random — it follows patterns. Someone who buys a phone often buys a case next. Someone who buys running shoes often buys sportswear after.

```
# For each client, sort purchases by timestamp
# Record every category transition
for each client:
    sequence = [cat_t0, cat_t1, cat_t2, ...]
    transitions: (cat_t0 → cat_t1), (cat_t1 → cat_t2), ...

# Aggregate across all clients
# transition_matrix[A][B] = P(client buys from B | just bought from A)
```

This gives us a probability table: given that someone just bought from category A, what are the most likely next categories?

**Analogy from credit risk:** This is the same logic as a roll-rate matrix, where we track how clients move between delinquency buckets. Here we track how they move between purchase categories.

**Limitation:** Clients with only one purchase have no observed transition. That's handled in the fallback logic.

---

### 3.3 Price Tier — Within-Category Sensitivity

Price sensitivity is relative, not absolute. A client who spends $500 on electronics and $20 on beauty is not inconsistent — they just operate at different price points per category.

```
# For each (client, category) pair:
median_spend = median of all purchases in that category
percentile = where that median falls in the category's global distribution

# Assign tier:
# LOW  → below p33
# MID  → p33 to p66
# HIGH → above p66

# Result: client_price_profile = {"ELECTRONICS": "MID", "FASHION": "LOW", ...}
```

This tier becomes a filter on the Amazon catalog. If a client is a MID spender in electronics, we only recommend MID-priced Amazon electronics to them.

**Why percentiles, not absolute prices?**  Relative position within a category is comparable across markets.

---

## 4. Recommendation Logic

With those three features built, the recommendation is straightforward.

### 4.1 Core Logic

```
function recommend(client_id):

    # Step 1: Define target categories
    last_category   = client's most recent purchase category
    next_categories = top 3 from transition_matrix[last_category]  # what usually comes next
    base_categories = client's top 3 most purchased categories      # what they always buy
    target_categories = union(next_categories, base_categories)
    # next_categories = discovery / novelty
    # base_categories = safe bets we know they like

    # Step 2: Pull Amazon candidates
    candidates = amazon_catalog WHERE mapped_kz_category IN target_categories
    # If a category has no mapped Amazon products → replace with most popular KZ category that does

    # Step 3: Apply price filter
    for each category in target_categories:
        keep only candidates WHERE amazon_price_tier == client_price_profile[category]

    # Step 4: Rank
    score = weighted combination of:
        - transition_probability  (how likely is this category after last purchase)
        - client_category_frequency  (how often has this client bought in this category)

    return top_10_by_score
```

### 4.2 Fallback Strategy

| Client type | Strategy |
|---|---|
| Has purchase history (1+ purchases) | Use transition matrix for next categories. Single-purchase clients still have a "next likely category" from population patterns. |
| No purchase history (new client) | Top 3 most popular KZ categories × 3 price tiers = 9 products. Fill to 10 with overall best-rated product. |

The key point: the transition matrix is built from all clients, not per client. So even a one-purchase client gets a meaningful "what comes next" signal — it's just based on what everyone else did after buying the same thing.

---

## 5. Category Discovery

**Goal:** Find meaningful product groupings in the Amazon catalog beyond the raw 939 categories — many of which are redundant, overlapping, or too granular.

**Data source:** Free text from `about_product` and `product_specification`.

### How It Works

**Step 1 — Text cleaning:**
```
for each product:
    text = about_product + " " + product_specification
    remove stopwords  # using NLTK's built-in English stopword list
    remove punctuation and numbers
    lowercase everything
```

**Step 2 — Vectorization (TF-IDF):**

TF-IDF converts each product's text into a vector of numbers. The logic is:
- A word that appears a lot in *this product's* description but rarely across the full catalog → high weight. It's distinctive.
- A word like "the" or "product" that appears everywhere → weight near zero. It's noise.

Result: each product becomes a vector where the highest values represent its most distinctive words.

```
product_vectors = TF-IDF(cleaned_texts)
# shape: (n_products, n_vocabulary_words)
```

**Step 3 — Clustering (K-Means):**

K-Means groups products by similarity of their TF-IDF vectors. Products that use similar distinctive words end up in the same cluster — even if their original category labels were different.

```
# Try K between 15 and 30
# Use silhouette score to pick optimal K
clusters = KMeans(product_vectors, k=optimal_K)
```

**Step 4 — Label the clusters:**
```
for each cluster:
    top_words = most frequent words in cluster  # excluding stopwords
    proposed_label = human review of top_words
```

**Expected output:** 15–30 macro-categories that actually describe what products are. These also feed back into the category mapping from Step 3.1 — a cleaner taxonomy makes the recommender more reliable.

---

## 6. Audience Definition — Buyer Personas

Using the features already computed, we can define client segments that marketing can act on.

### Dimensions

| Dimension | Description |
|---|---|
| Dominant category | What does this client mostly buy? |
| Purchase frequency | One-time, occasional, or recurrent buyer? |
| Price tier | Budget, mid, or premium within their categories? |

```
for each client:
    dominant_category  = category with most purchases
    frequency_tier     = LOW / MID / HIGH by purchase count percentile
    price_tier         = from feature engineering (3.3)

# Run K-Means (K=4–6) on these three dimensions
# Label each cluster with a human-readable persona name
```

### Example Personas *(hypothetical — to be validated with data)*

| Persona | Category | Frequency | Price | Description |
|---|---|---|---|---|
| 🛍️ Trend Chaser | Fashion | High | Mid | Buys clothing often, price-aware but not budget-only |
| 📱 Tech Enthusiast | Electronics | Mid | High | Less frequent but high-value purchases |
| 🏠 Homebody | Home | Low | Mid | Occasional home purchases, not price-driven |
| 💸 Bargain Hunter | Mixed | High | Low | Buys a lot across categories, always cheapest option |
| 🎯 One-Category Buyer | Any | Low | Any | Narrow interest, one-time or very infrequent |

Each persona implies a slightly different recommendation strategy — the Tech Enthusiast gets premium products, the Bargain Hunter gets budget picks across categories.

---

## 7. Validation

The fundamental problem: we have no clicks, no ratings, no explicit feedback. Only purchase history. So validation has to be built from what we have.

### Offline — Leave-Last-Out

Hide the client's most recent purchase. Ask the model to recommend. Did the right category appear in the top 10?

```
for each client with 2+ purchases:
    train  = all purchases except the last
    truth  = category of last purchase

    recs   = recommend(client, using train only)
    hit    = 1 if truth in [r.mapped_category for r in recs] else 0

hit_rate = mean(hit) across all clients
```

**Metric: Hit Rate @10** — what % of clients had their actual next category somewhere in the top 10.

**Baselines to beat:**
- Random: 10 random categories per client
- Popularity: always recommend the 10 most purchased categories globally

If our model doesn't beat both, it's not adding value.

**Limitation:** We validate at the category level, not the exact product. We also are only validating on clients with >=2 purchases.

---

## 8. Summary

The system is built on three features that feed into a content-based recommender:

1. **Category mapping** — connects KZ categories to Amazon products via regex on text fields.
2. **Transition matrix** — tells us which categories clients tend to buy into next, based on population patterns.
3. **Price tier** — filters Amazon products to match each client's relative price position within a category.

Recommendations are generated by combining "what usually comes next after the last purchase" with "what this client has always bought," then filtering by price and ranking by signal strength.

### Trade-offs

| Decision | What we chose | What we gave up |
|---|---|---|
| Recommendation scope | One per client, based on latest state | Per-event precision |
| Category bridging | Manual regex, top 80% of categories | Brittle to new categories |
| Price matching | Relative percentile within category | Exact cross-market price accuracy |
| Complexity | Simple, explainable | May underperform neural approaches |



---

## 9. AI Tool Usage

This proposal was built collaboratively with Claude. The AI was used to:
- Challenge assumptions during design (e.g., the price matching approach).
- Suggest standard terminology for ideas I had already conceptualized (e.g., "hit rate @10", "leave-last-out", "transition matrix").
- Write and structure the document based on decisions I made in conversation.

Every design choice in this document is mine and I can defend it in a technical review.

---
*End of proposal*
