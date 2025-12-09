# Finance Churn Prediction with Graph Data Science

![customer-churn](customer-churn.png)

This project implements a churn prediction system for the financial sector, using Neo4j's Graph Data Science (GDS) library. It leverages graph-based features ([embeddings](https://neo4j.com/docs/graph-data-science/current/machine-learning/node-embeddings/) and [centrality](https://neo4j.com/docs/graph-data-science/current/algorithms/centrality/ )) alongside traditional demographic data to identify active users at risk of churning. The dataset used is sourced from Kaggle's [computingvictor/transactions-fraud-datasets](https://www.kaggle.com/datasets/computingvictor/transactions-fraud-datasets), this dataset was chosen as it includes enough structure (merchants, transactions, cards and users) to derive an interesting graph architecture. However it does not include a specific churn label, so we engineered one based on user inactivity.

## The ins and outs of Churn Prediction

Customer Churn (or customer attrition) refers to the phenomenon where customers stop doing business with a company. In the context of finance and banking, this is critical because acquiring a new customer is significantly more expensive than retaining an existing one.

<figure>
  <img src="transaction-network.png" alt="Transaction Network" />
  <figcaption>Visualizing the transaction network between users and merchants (clients are *red* nodes).</figcaption>
</figure>

### What Customer Churn means in Finance

In subscription-based industries (like Netflix or Spotify), churn is explicit: a user hits "Cancel." In banking and credit cards, however, churn is often **silent** (or non-contractual). A customer rarely calls to say, "I am leaving.". Instead, they simply stop using the card, move their direct deposit elsewhere, or leave an account dormant (zero balance or inactivity) for an extended period.

Therefore, financial churn prediction is not just about detecting closed accounts, but detecting **dormancy** - identifying users whose activity has dropped to a level where they are effectively gone, even if the account remains technically open.

<figure>
    <img src="whale-network.png" alt="whale network" />
    <figcaption>Transaction network of high-income, young users making large transactions ("Whales").</figcaption>
</figure>

### Typical Methods

Traditional approaches to predicting churn often rely on tabular data and [RFM analysis](https://www.investopedia.com/terms/r/rfm-recency-frequency-monetary-value.asp):
* **Recency:** How long since the last transaction?
* **Frequency:** How often do they transact?
* **Monetary:** How much do they spend?

While effective, these methods often view customers in isolation. They miss the **network effects** - for example, if a user's entire household moves banks, or if a user stops shopping at key "sticky" merchants (like a daily commuter train or a favorite shop).

**Graph Data Science** enhances this by treating customers as nodes in a network. By analyzing the structure of relationships (e.g., "Users who shop at Merchant X tend to stay loyal," or "This user behaves like these other users who recently churned"), we can extract stronger signals than demographics alone can provide.

<figure>
    <img src="smurf-networks.png" alt="Smurf networks" />
    <figcaption>Potential "Smurf" fraud networks.</figcaption>
</figure>

### Caveats & Challenges
1.  **Imbalanced Data:** In a healthy bank, 95% to 99% of customers are active. This "Class Imbalance" makes it hard for models to learn what a churner looks like (finding a needle in a haystack).
2.  **Definition of Churn:** Since churn is silent, defining the target variable is subjective. Is a user churned after 30 days of silence? 90 days? 1 year? The definition heavily impacts model performance.
3.  **Behavioral vs. Structural:** Some churn is purely behavioral (spending drops), while some is structural (life events like moving house). Graph features help capture structural changes better than simple spending tables.
4.  **Inferred Labels:** As noted, the lack of an explicit "cancel" event means the ground truth labels are often synthetic (engineered based on rules) rather than observed facts.
5.  **Lack of Temporal Features:** The current model uses a static snapshot of the graph where all historical transactions are aggregated into a single weight. It does not account for the *sequence* or *recency* of specific interactions within the graph topology (e.g., it cannot distinguish between a user who stopped transacting gradually vs. abruptly).

## Purpose of this Project

The primary goal of this model is to predict customer churn by analyzing transaction patterns. Unlike traditional methods that only look at individual user behavior, our approach uses graph features to understand the relationships between users, cards, and merchants. By identifying high-risk active users, financial institutions can take proactive measures (e.g., targeted campaigns, personalized offers or other incentives) to retain them.

This implementation focuses on card products, but a similar pipeline could be applied to other financial products (e.g., loans, mortgages, insurance policies) where customer retention is critical.

Our ultimate objective is to highlight how graph data science and Neo4j can be used to build effective churn prediction systems that go beyond standard tabular ML approaches, not to provide a production-ready solution or a fully validated model with a dataset with sufficient ground truth. What we present here is intended to give an initial foundation for further exploration and refinement, and an introduction to what is possible with graph-based machine learning for customer churn prediction.

## Method

The project follows a complete data science pipeline from data ingestion to model inference:

### 1. Data Acquisition & Processing ([`donwloader.ipynb`](donwloader.ipynb))
*   **Source:** [Transactions Fraud Datasets](https://www.kaggle.com/datasets/computingvictor/transactions-fraud-datasets) (via Kaggle).
*   **Processing:**
    *   Cleans raw CSV data (currency formatting, date parsing).
    *   Defines "Churn" based on user inactivity (e.g., no transactions for > `CHURN_THRESHOLD_DAYS`).
    *   Merges User, Card, and Transaction data into a single `processed_transactions_data.csv` which can be loaded into Neo4j (see [`NOTES.md`](NOTES.md)).

### 2. Graph Modeling (Neo4j)
The data is loaded into a Neo4j graph database with the following structure:
*   **Nodes:** `User`, `Card`, `Merchant`, `Transaction`, `Category`.
*   **Relationships:** 
    *   `(:User)-[:OWNS]->(:Card)`
    *   `(:Card)-[:PERFORMED]->(:Transaction)`
    *   `(:Transaction)-[:TO]->(:Merchant)`
*   **GDS Optimization:** A simplified relationship `(:User)-[:SHOPPED_AT {weight: count}]->(:Merchant)` is materialized to represent the strength of user-merchant interactions for efficient graph algorithms. This will further help capture user behavior patterns in the graph structure, and the relative importance of merchants to users.

### 3. Graph Data Science Pipeline ([`train-evaluate.ipynb`](train-evaluate.ipynb))
The model training is performed entirely within Neo4j using the GDS Python Client:

*   **Class Imbalance Handling:** A `TrainingCohort` is created by selecting 100% of churned users and undersampling active users (e.g., 5%) to create a balanced training set (the original `computingvictor/transactions-fraud-datasets` is highly imbalanced for our churn prediction task).
*   **Feature Engineering:**
    *   **FastRP (Fast Random Projection):** Generates 32-dimensional node embeddings to capture graph topology.
    *   **PageRank:** Calculates node centrality/importance based on transaction flow.
    *   **Node Properties:** `yearly_income`, `total_debt`, `credit_score`.
*   **Model Training:**
    *   **Algorithm:** Random Forest Classifier (50 trees, max depth 5).
    *   **Pipeline:** `gds.beta.pipeline.nodeClassification`.
    *   **Validation:** 80/20 Train/Test split with 2-fold cross-validation.

### 4. Inference
*   The trained model is applied to the **Full Graph** (all users, not just the training cohort).
*   The system predicts a "Risk Score" (probability of churn) for currently **Active** users.

In our case we run both training and inference entirely withing Neo4j using the GDS library, orchestrated via the Neo4j Python Client in `train-evaluate.ipynb`. In a production scenario, model training could be scheduled periodically (e.g., monthly) to refresh the model with the latest data, while inference could be run more frequently (e.g., daily) to identify at-risk users in near real-time. Customers would very possibly run training and inference outside of Neo4j, exporting embeddings and features to a dedicated ML environment. However, running everything within Neo4j simplifies the architecture and showcases the power of GDS for end-to-end graph-based machine learning.

## Outcome

*   **Trained Model:** A Random Forest classifier that integrates graph embeddings with demographic features.
*   **Risk Analysis:** A ranked list of active users with the highest probability of churning.
*   **Metrics:** Evaluation of the model using Accuracy, Precision, and Recall.

NOTE: Precision and Recall are *not* presented here as the dataset lacks enough ground truth for churned users. In a real-world scenario, these metrics would be critical to assess model performance.

## Setup & Usage

1.  **Environment:** Install dependencies using `conda env create -f environment.yml`.
2.  **Data:** Run `donwloader.ipynb` to download and process the dataset - this generates `processed_transactions_data.csv` which should be loaded into Neo4j (see [`NOTES.md`](NOTES.md)).
3.  **Database:** Ensure a Neo4j instance is running and configured in `train-evaluate.ipynb`.
4.  **Training:** Run `train-evaluate.ipynb` to execute the GDS pipeline, train the model, and view the top at-risk users.