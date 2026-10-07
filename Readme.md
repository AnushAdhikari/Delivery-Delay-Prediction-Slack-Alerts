
# MapleFreight Delivery Delay Prediction & Slack Alerts

**Author:** Anush Adhikari | **Student ID:** c0968893

## Problem
MapleFreight Logistics ships about 8,000 packages a week across Ontario, Quebec and the Prairies. On-time performance fell from 94% to 88%, and delays are only discovered when the customer calls. This project predicts which in-transit shipments will be late and sends a Slack alert to dispatch while there is still time to re-route or warn the customer.

## Dataset
`maplefreight_delivery_delay_dataset.csv`: 6,035 shipments, 23 columns. Target: `delivered_late` (1 = late, about 21%).

Cleaning applied:
- 35 duplicate shipments removed (32 identical rows, 3 differing only in spelling), leaving 6,000.
- `service_level` spellings standardised (6 variants down to 3).
- 40 weights entered in grams instead of kg were divided by 1,000.
- `actual_transit_hours` dropped, because it is only known after delivery (data leakage).
- Missing values (driver experience, weather, traffic index, fuel cost) filled inside a pipeline using training data only.

## How to run
1. Clone the repo and create a virtual environment:
- python -m venv .venv
- .venv\Scripts\activate
- pip install -r requirements.txt
2. Create a Slack app with Incoming Webhooks, add a webhook to `#dispatch-alerts`, and copy the URL.
3. Copy `.env.example` to `.env` and put your webhook URL in it:
  - SLACK_WEBHOOK_URL=your_webhook_url_here
4. Open `MapleFreight_Delivery_Delay_Prediction_c0968893.ipynb`, select the `.venv` kernel, and run all cells from the top.

`.env` is listed in `.gitignore`. No webhook URL is stored in the repo or printed in the notebook.

## Method
- EDA, then new features (required speed, stops per 100 km, severe-weather flag, late-pickup flag).
- Stratified 80/20 train/test split (4,800 / 1,200 rows).
- Baselines: a dummy model that always predicts on time, and logistic regression with class weights.
- Stronger models: random forest and gradient boosting, also with class weights.
- Explainability: permutation importance and logistic regression coefficients.

## Results (test set, default 0.5 cutoff)

| Model | Precision | Recall | F1 | ROC-AUC |
|---|---|---|---|---|
| Dummy (always on time) | 0.000 | 0.000 | 0.000 | 0.500 |
| Logistic regression (baseline) | 0.445 | 0.741 | 0.556 | 0.830 |
| Random forest | 0.500 | 0.623 | 0.555 | 0.812 |
| Gradient boosting | 0.460 | 0.684 | 0.550 | 0.814 |

The dummy model scores 79% accuracy but catches no late shipments. The more complex models did not beat logistic regression, so logistic regression is the final model.

## Alert threshold: 0.50 vs 0.60

I compared two thresholds on the test set. The default 0.50 was the starting point, and I then used 5-fold cross-validation on the training set only to look for a better one. F1 peaked around 0.55-0.60, so I tested 0.60.

| Threshold | Alerts | Late caught | Missed | False alarms | Precision | Recall | F1 |
|---|---|---|---|---|---|---|---|
| 0.50 | 411 | 183 | 64 | 228 | 0.445 | 0.741 | 0.556 |
| **0.60 (chosen)** | 319 | 161 | 86 | 158 | 0.505 | 0.652 | 0.569 |

**What changed when I raised it to 0.60:**
- Alerts fell by 92 (411 to 319).
- False alarms fell by 70 (228 to 158).
- Precision rose from 0.445 to 0.505, so about half of alerts are now truly late.
- Recall fell from 0.741 to 0.652, so 22 more late shipments are missed (64 to 86).
- F1 improved slightly (0.556 to 0.569).

**Why I chose 0.60:** dispatch will stop trusting alerts if about 55% of them are false. Raising the threshold removes 70 false alarms at the cost of 22 missed late shipments. Shipments scoring 0.50-0.60 are shown as a "watch list" count (92) in the Slack message, so they are not lost.

**Trade-off:** if the cost of a missed late delivery is higher than the cost of a false alarm, 0.50 would be the better choice. This is a business decision and can be changed in one line of code.

**Why 0.60:** the threshold was chosen with 5-fold cross-validation on the training set only. F1 peaks around 0.55-0.60. At 0.60 the model catches about 65% of late shipments while keeping alerts more trustworthy, so dispatch is less likely to ignore them. A threshold of 0.50 catches more late shipments but sends 92 more alerts, most of them false. Shipments scoring 0.50-0.60 are listed as a "watch list" count in the Slack message.

Confusion matrix at 0.60 (rows = actual, columns = predicted): on time 795 / 158, late 86 / 161.

## Key findings
- Strongest drivers: distance, severe weather, traffic, carrier and number of stops.
- Contract owner-operators are late about 36% of the time vs 13% for the MapleFreight Fleet.
- Late rate is 14% in clear weather, 34% in snow and 47% in storms.
- Late rate rises from 12% with no stops to 29% with five.
- Customer tier and origin/destination city have little effect.

## Slack alert
Screenshot: `Screenshots/slack_alert.png`. The message gives the number of at-risk shipments and the top 5 with ID, destination, carrier and risk score.

## Recommendations
See Section 12 of the notebook: use the alerts each shift, review owner-operator contracts, check the weather before dispatch, limit stops on time-critical routes, act on late pickups, set realistic Economy windows, and complete a full pre-trip inspection of every truck.

## AI usage

I used **Perplexity** as an AI assistant to plan the workflow, draft code, debug, and write explanations. I reviewed and ran every piece of code myself, and I checked the numbers in the text against my notebook output.

- **Connecting to Slack.** I had trouble connecting the notebook to Slack at first. The AI assistant walked me through it step by step: creating the Slack app, turning on Incoming Webhooks, adding a webhook to `#dispatch-alerts`, and storing the URL in a `.env` file instead of in the code. I then tested it with a short message before sending the real alert. The final setup worked because I followed each step and checked the result before moving on.
- 
**Where the AI was wrong or differed from my results:**

1. **Code error.** The helper function `evaluate()` ended with `return pd.Series(row).round(3)`. The dictionary included the model name (a string), so `round` failed with `TypeError: type str doesn't define __round__ method`. I found it when I ran the cell. The fix was to leave out the "Model" entry before rounding.

2. **Estimated numbers did not match the output.** The AI estimated about 105 shipments for the watch list (scores 0.50-0.60). The real output was 92. The AI's numbers were guesses made before the code ran, so I used the notebook output and corrected the text.

3. **Output varied between runs of the same prompt.** When I asked the AI the same question twice, it gave slightly different suggestions, such as different feature ideas and different wording for the same recommendation. I treated its answers as drafts, tested each one in the notebook, and kept only what worked.

**How I handled it:**
- I ran every cell and compared the AI's claims with the real output.
- I verified recommendations against the data. For example, the AI's pre-trip inspection suggestion is not backed by the dataset, so I labelled it as based on industry practice.
- I did not copy any numbers that I could not find in my own output.

**Lesson:** the AI is useful for speed, but it can be confidently wrong, and its answers can differ from one run to the next. The results in this project are the ones my code produced.
