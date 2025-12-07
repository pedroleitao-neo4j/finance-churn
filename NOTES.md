# The Graph Data Model

## Step 1 - Constraints

```cypher
// Create Constraints (Run these one by one)
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT card_id IF NOT EXISTS FOR (c:Card) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT merchant_id IF NOT EXISTS FOR (m:Merchant) REQUIRE m.id IS UNIQUE;
CREATE CONSTRAINT transaction_id IF NOT EXISTS FOR (t:Transaction) REQUIRE t.id IS UNIQUE;
CREATE CONSTRAINT category_id IF NOT EXISTS FOR (c:Category) REQUIRE c.code IS UNIQUE;
```

## Step 2 - Loading Data

```cypher
// Load the file from the import directory
LOAD CSV WITH HEADERS FROM 'file:///processed_transactions_data.csv' AS row

// Process in batches of 2000 rows to prevent OutOfMemory errors
CALL {
    WITH row

    // -----------------------------------------------------
    // User & Demographics
    // -----------------------------------------------------
    MERGE (u:User {id: toInteger(row.id_user)})
    ON CREATE SET 
        u.current_age = toInteger(row.current_age),
        u.retirement_age = toInteger(row.retirement_age),
        u.gender = row.gender,
        u.yearly_income = toFloat(row.yearly_income),
        u.credit_score = toInteger(row.credit_score),
        u.address = row.address,
        u.latitude = toFloat(row.user_latitude),
        u.longitude = toFloat(row.user_longitude),
        u.per_capita_income = toFloat(row.per_capita_income),
        u.yearly_income = toFloat(row.yearly_income),
        u.total_debt = toFloat(row.total_debt),
        u.credit_score = toInteger(row.credit_score),
        u.num_credit_cards = toInteger(row.num_credit_cards),
        u.churned = toInteger(row.churned)

    // -----------------------------------------------------
    // Card & Ownership
    // -----------------------------------------------------
    MERGE (c:Card {id: toInteger(row.card_id)})
    ON CREATE SET 
        c.brand = row.card_brand,
        c.type = row.card_type,
        c.limit = toFloat(row.credit_limit),
        c.expires = datetime(replace(row.expires, ' ', 'T')),
        c.has_chip = row.has_chip,
        c.on_dark_web = row.card_on_dark_web

    MERGE (u)-[:OWNS]->(c)

    // -----------------------------------------------------
    // Merchant & Category
    // -----------------------------------------------------
    // Handle Category first so we can link Merchant to it
    MERGE (cat:Category {code: toInteger(row.mcc)})
    ON CREATE SET cat.name = row.merchant_category

    MERGE (m:Merchant {id: toInteger(row.merchant_id)})
    ON CREATE SET 
        m.city = row.merchant_city,
        m.state = row.merchant_state,
        m.zip = toInteger(row.zip),
        m.latitude = toFloat(row.latitude),
        m.longitude = toFloat(row.longitude)

    MERGE (m)-[:IN_CATEGORY]->(cat)

    // -----------------------------------------------------
    // Transaction Event
    // -----------------------------------------------------
    MERGE (t:Transaction {id: toInteger(row.id_transaction)})
    ON CREATE SET 
        t.amount = toFloat(row.amount),
        t.date = datetime(replace(row.date, ' ', 'T')),
        t.use_chip = row.use_chip,
        t.errors = row.errors

    // Connect the Event
    MERGE (c)-[:PERFORMED]->(t)
    MERGE (t)-[:TO]->(m)

} IN TRANSACTIONS OF 2000 ROWS;
```

## Example Queries

**Recent Transactions by Churned Users**

```cypher
MATCH (u:User)-[:OWNS]->(c:Card)-[:PERFORMED]->(t:Transaction)-[:TO]->(m:Merchant)
WHERE u.churned = 1
RETURN u.id AS user_id, c.id AS card_id, t.id AS transaction_id, t.amount AS amount, t.date AS date, m.id AS merchant_id
ORDER BY t.date DESC
LIMIT 50;
```

**Transaction Network for a Specific Card**

```cypher
MATCH p=(u:User)-[:OWNS]->(c:Card {id:3840})-[:PERFORMED]->(:Transaction)-[:TO]->(:Merchant)-[:IN_CATEGORY]->(:Category)
RETURN p
```

**High Spending Users**

```cypher
MATCH (u:User)-[:OWNS]->(c:Card)-[:PERFORMED]->(t:Transaction)
WITH u, SUM(t.amount) AS total_spent
WHERE total_spent > 10000
RETURN u.id AS user_id, total_spent
ORDER BY total_spent DESC;
```

**Transaction Network for a Given Transaction Amount**

```cypher
MATCH p = (u:User)-[:OWNS]->(c:Card)-[:PERFORMED]->(t:Transaction)-[:TO]->(m:Merchant)-[:IN_CATEGORY]->(cat:Category)
WHERE t.amount > 3000
RETURN p
```

**The "Whale" Journey**

Transaction networks of high-income, young users making large transactions.

```cypher
// Find High Income Users and show their high-value network
MATCH p = (u:User)-[:OWNS]->(c:Card)-[:PERFORMED]->(t:Transaction)-[:TO]->(m:Merchant)-[:IN_CATEGORY]->(cat:Category)
WHERE u.yearly_income > 100000 AND u.current_age < 35 
  AND t.amount > 1000
RETURN p
```

**Identify Merchants with High Churned User Activity**

**The Theory**: A "Bust-Out" fraud happens when users build up credit, max out their cards, and then disappear (Churn). If multiple Churned Users all maxed out their cards at the same Merchant, that Merchant might be complicit (laundering money) or the target of a coordinated attack.

This query utilizes the churned column we engineered earlier.

```cypher
// Find Merchants where multiple 'Churned' users transacted
MATCH (u:User {churned: 1})-[:OWNS]->(c:Card)-[:PERFORMED]->(t:Transaction)-[:TO]->(m:Merchant)

// Aggregating to find the convergence point
WITH m, count(DISTINCT u) AS num_churned_users, sum(t.amount) AS total_busted_amount
WHERE num_churned_users > 3 // Threshold: More than 3 churned users at one shop

RETURN m.id, m.city, m.state, num_churned_users, total_busted_amount
ORDER BY total_busted_amount DESC;
```

**Smurfing Pattern Detection**

**The Theory**: "Smurfing" involves breaking up large transactions into many smaller ones to avoid detection. A common pattern is multiple small transactions at the same Merchant within a short time frame.

```cypher
MATCH (c:Card)-[:PERFORMED]->(t:Transaction)-[:TO]->(m:Merchant)

// Filter for the "Smurfing" pattern (small amounts)
WHERE t.amount < 50 

// Aggregate by Card, Merchant, AND the specific Day
WITH c, m, 
     date.truncate('day', t.date) AS tx_day, // Buckets transactions by calendar day
     count(t) AS daily_tx_count, 
     sum(t.amount) AS daily_total_amount,
     collect(t.date) AS timestamps // Optional: to see the exact times

// Apply the threshold (e.g., >= 5 swipes at the same shop in one day)
WHERE daily_tx_count >= 5 

RETURN 
    c.id AS Card_ID, 
    m.id AS Merchant_ID,
    m.city AS Merchant_City, 
    tx_day, 
    daily_tx_count, 
    daily_total_amount,
    timestamps
ORDER BY daily_tx_count DESC;
```