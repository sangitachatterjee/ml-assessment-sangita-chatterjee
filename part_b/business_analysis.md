# Part B: Business Case Analysis - Promotion Effectiveness

## B1. Problem Formulation (8 marks)

### B1(a) Formulate this as an ML problem (3 marks)

I see this as a supervised learning task: estimate units sold for a store-month under a chosen promotion.

The target is items_sold at the store-month-promotion level.

Inputs should include:
- Store context like store size, competition density, and demographics if available
- Calendar context like month, quarter, weekends, festivals, and month-end effects
- Promotion details like promotion type
- Historical behavior like lag sales, rolling averages, and past response to each promotion type

Since items_sold is continuous, this is a regression setup.

Operationally, the workflow is simple: score every promotion option for a store-month, then pick the one with the highest predicted units.

### B1(b) Why items_sold is more reliable than total revenue (3 marks)

For this decision, items_sold is the safer target than revenue.

Revenue moves for many reasons at once: price changes, discount depth, bundling, and mix shifts. That noise makes attribution harder.

Items sold is cleaner because it captures volume response directly. If the question is "which promotion moves product," this is the right KPI.

### B1(c) Why one global model is not enough + better alternative (2 marks)

A single global model usually washes out local behavior.

For example:
- Urban and rural stores may react very differently to the same promotion
- High-competition areas may need stronger offers to get the same lift

A better setup is one of these:
- Build segmented models by location type or store cluster
- Use one shared model but add interaction terms between promotion and store context
- Use a hierarchical model that shares signal across stores while allowing local variation

All three options handle heterogeneous promotion response better than a one-size-fits-all model.

## B2. Data and EDA Strategy (10 marks)

### B2(a) Join plan, modeling grain, and aggregations (4 marks)

Given the tables:
- Transactions
- Store attributes
- Promotion details
- Calendar

Join order:
1. Transactions with store attributes on store_id
2. Add promotion details using promotion key
3. Add calendar features using transaction date

For modeling grain, keep one row per store-month-promotion combination.

Key aggregations:
- Monthly target as sum of items_sold
- Calendar summaries like weekend and festival counts or ratios
- Context features as static or monthly aggregates depending on whether they change
- Time features like lag sales and rolling mean sales

### B2(b) EDA to run before modeling (4 marks)

Before training, run a focused EDA pass:

1. Data quality and missingness
   - Null checks by store type, location, and promotion
   - Decide imputation strategy and whether missing flags are useful

2. Target distribution
   - Histograms and boxplots for items_sold overall and by segments
   - Outlier and skew checks

3. Time behavior
   - Monthly trends overall and by segment
   - Check if seasonality patterns are visible

4. Promotion effect signal
   - Compare average and median items_sold across promotion types
   - Stratify by location and store size to avoid misleading averages

5. Interaction clues
   - Quick checks like competition density versus sales under different promotions

Why this matters in practice:
- If relationships are non-linear, tree models usually work better
- If target is highly skewed, log transform or robust methods may help
- If promo effect changes by segment, interactions or segmented models become important

### B2(c) Handling the 80% no promotion imbalance (2 marks)

If most rows are no-promotion, the model tends to memorize baseline demand and miss promotion lift.

A practical fix is to combine:
1. Row weighting so promoted rows matter more during training
2. Sampling strategy to reduce dominance of no-promo rows
3. Lift-style modeling relative to baseline no-promo outcome
4. Evaluation broken down by promotion type, not just one global metric

## B3. Evaluation and Deployment (12 marks)

### B3(a) Split, metrics, and why random split is wrong here (4 marks)

With three years of store-level monthly data, the split should be chronological:
- Train on earlier months
- Test on the latest months

Random split is a bad idea here because it leaks future patterns into training and makes offline performance look better than reality.

Core metrics:
- RMSE to penalize large misses
- MAE for average absolute error in units sold

Because recommendations are the real output, also track decision metrics:
- Top-1 recommendation accuracy (did we pick the best promo)
- Regret or lost units versus the best possible promotion

Prediction metrics show forecast quality. Decision metrics show business usefulness.

### B3(b) Explaining different recommendations for same store in different months (4 marks)

If the same store gets one promotion in December and another in March, explain it this way:

1. Score all promotion options for both months
2. Generate local explanations for each month using SHAP or permutation methods
3. Compare top contributing features side by side

How to present this to marketing:
- Top drivers for each month in plain language
- Short comparison table showing which factors changed
- One clear takeaway, for example festival season signal pushes a different promotion in December

That format is usually enough for stakeholders to trust and use the recommendation.

### B3(c) Deployment workflow: save, score monthly, monitor drift (4 marks)

A practical deployment loop:
1. Train and version the full preprocessing plus model pipeline
2. Refresh monthly features every cycle
3. Score all candidate promotions for each store-month
4. Select the promotion with highest predicted items_sold
5. Monitor for:
   - Data drift in feature distributions
   - Error drift once actual sales arrive
   - Business drift in uplift or regret over time
6. Retrain when thresholds are breached

This keeps the system stable as behavior, competition, and seasonality shift over time.

