# Data Science Technical Challenge: E-Commerce Recommender

**Note:**

You are completely free to use AI coding assistants like Cursor, GitHub Copilot, or Claude during this challenge. We value engineers who know how to use modern tools to speed up their operational work (like writing regex, generating plotting code, or scaffolding classes). If you use them, please add a short section at the end of your presentation explaining how you integrated them into your workflow. We care about your system design and logical choices, not how fast you can type boilerplate code from memory.

## Business Case

We want you to build a recommendation system for an E-commerce platform. To design this recommender, we are providing two different datasets:

1. **Amazon Product descriptions** — `marketing_sample_for_amazon_com-ecommerce__20200101_20200131__10k_data.csv.zip`
2. **Client purchase history** — `kz.csv.zip`

**Download the datasets** from https://github.com/PedPalencia/data-experimento and place them in the `data/` folder.

We want you to find creative ways to bridge any gap you find. If you need to, you can have access to an AI API key.

**The final output should be the top 10 recommended Amazon products for each client.**

## Requirements

Please present your analytical solution clearly. Include your reasoning for the specific approaches you chose in each stage: Business Understanding, EDA, Feature Engineering, Model Design, Validation, and Results Showcasing.

Along with the core recommender, the Product and Marketing teams need your help with the following tasks:

**1. Category Discovery**
The product manager wants to know if you can identify new, meaningful product categories beyond what is listed in the raw `Category` column.
*Hint:* You can use text data from `about_product` and `product_specification`. 

**2. Audience Definition**
Define distinct client audiences based on the output of your recommender system and their purchase history.
*Hint: We are looking for buyer personas*.

**3. Validation Approach**
Design the validation approach for your recommender system. How do we know your recommended products are actually good matches for these clients? Propose specific metrics or an experiment design.

## Optional Questions

- Given an existing product recommendation system, how would you design an A/B test to measure improvements?
- How would you scale this recommendation system to handle 5 million daily users and a catalog of 20 million products? *Hint:* If your solution uses embeddings, standard array calculations will fail at this scale. Mention the specific architectural approaches, databases, or indexing tools you would put in production.