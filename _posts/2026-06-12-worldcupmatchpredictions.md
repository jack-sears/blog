---
layout: post
title: "Predicting the World Cup"
date: 2026-06-12
categories: [notes]
---

<script type="text/javascript" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>

In honour of the start of the 2026 Fifa World Cup and my competitive nature to try and win my office score prediction pool, I decided to quickly throw together a "fun" analysis using some of my math, programming, and most importantly, claude background. 

## Scoring

The league is pretty straight forward. Players make a score prediction for every game in the world cup and are scored based on the following criteria:

1. 3 points if you predict the exact score
1. 1.5 points if you get the result right and your score is close
1. 1 point if your result is right but your score isn't close
1. 0 points if you don't get the result right

What "your score is close" means is beyond me and I didn't feel like checking so here we are. Also note that you can make the predictions anytime before the game starts. Anyways shoutout [SuperBru][superbru] for this prediction game and let's get into it.

## My Approach

I aim to go for a hybrid approach of using data as well as fan intuition to make my predictions. The goal will be to create baseline predictions using available data and then develop a framework that uses added information to hopefully enhance the predictions. To keep things simple I am going to use [1X2][1x2] and [over/under][overunder] betting odds for each game to base the predictions off of. Then I will use some simple historical facts/stats to create a guideline to complete my decision making for the final scores to predict.

### Betting Odds

<div style="display:flex; gap:1.5rem; flex-wrap:wrap; margin:1.5rem 0;">

  <div style="flex:1; min-width:220px; border:1px solid #e8e8e8; border-radius:6px; padding:1.2rem 1.4rem;">
    <p style="margin:0 0 0.5rem; font-size:0.72rem; font-weight:600; letter-spacing:0.1em; text-transform:uppercase; color:#888;">Term</p>
    <p style="margin:0 0 0.75rem; font-size:1.3rem; font-weight:600; color:#111;">1X2 Odds</p>
    <p style="margin:0; font-size:0.92rem; line-height:1.7; color:#444;">
      A three-way market where <strong>1</strong> = home win, <strong>X</strong> = draw, <strong>2</strong> = away win.
      Each outcome is priced in decimal odds, so dividing 1 by the odds to get the implied probability.
      A price of 1.65 implies a 60.6% chance.
    </p>
  </div>

  <div style="flex:1; min-width:220px; border:1px solid #e8e8e8; border-radius:6px; padding:1.2rem 1.4rem;">
    <p style="margin:0 0 0.5rem; font-size:0.72rem; font-weight:600; letter-spacing:0.1em; text-transform:uppercase; color:#888;">Term</p>
    <p style="margin:0 0 0.75rem; font-size:1.3rem; font-weight:600; color:#111;">Over / Under</p>
    <p style="margin:0; font-size:0.92rem; line-height:1.7; color:#444;">
      A market on total goals scored by both teams. The most common line is <strong>2.5</strong> —
      you bet on whether the game finishes with 3 or more goals (over) or 2 or fewer (under). Same idea as 1X2 for finding implied probability.
    </p>
  </div>

</div>

I found [The Odds Api][oddsapi], which is a super cool api that gives you access to upcoming and historical sports betting odds. It has a free subscription tier where you get 500 free credits and get decent access to their data. Unfortunately, only paid members can get historical odds so I was not able to check that out, but otherwise it was easy to use and had everything I needed. Below is a simple python script showing how to extract 1X2 and over/under odds from the api. You can also check out the website above which has detailed documentation about using the API.

```python
import requests

#get you key from their website when signing up and reference documenation for sport
API_KEY = "your_key_here"
SPORT   = "soccer__fifa_world_cup"  

#request the correct API endpoint for odds you want
response = requests.get(
    "https://api.the-odds-api.com/v4/sports/{}/odds".format(SPORT),
    params={
        "apiKey":      API_KEY,
        "regions":     "uk",
        "markets":     "h2h,totals",
        "oddsFormat":  "decimal",
    }
)

# to check how many credits you have left (optional)
response.raise_for_status()
    remaining = response.headers.get("x-requests-remaining", "?")
    print(f"  [API] Requests remaining this month: {remaining}\n") 

# get each home and away odds from the first book maker
for match in response.json():
    home = match["home_team"]
    away = match["away_team"]

    for bookmaker in match["bookmakers"]:
        for market in bookmaker["markets"]:

            if market["key"] == "h2h":
                odds = {o["name"]: o["price"] for o in market["outcomes"]}
                print(f"{home} vs {away}")
                print(f"  H: {odds[home]}  D: {odds['Draw']}  A: {odds[away]}")

            if market["key"] == "totals":
                for o in market["outcomes"]:
                    if o["name"] == "Over":
                        print(f"  O/U {o['point']}: {o['price']}")
        break  # first bookmaker only
```
My idea was that if I had the bookies odds for every team's winning chances, as well as the line of over/under 2.5 goals in a game, then I figured I could do some sort of reverse engineering to predict each team's winning probability and number of goals expected to score. A main theme was trying to keep the model as simple, but meaningful as possible.

## Math Behind the Model

The model uses the Poisson distribution to estimate underlying win and goal probabilites from the betting odds. The poisson distribution is a discrete probability distribution, meaning that it gives the likelihood of a countable outcome occurring [(definition here)][poisson]. The outcome in our case is the number of goals scored in a given fixture and this value is represented by λ. The poisson distribution can be used if the following two conditions are met: 

1. Individual events happen at random and are independent.
1. We know the mean (average) number of events occurring within a given time interval.

Number 1 is an assumption of our model. What it means is that to use the poisson distribution, goals in a soccer game must happen randomly and the impact of a goal must not affect the likelihood of who scores the next goal. In reality this is not necessarily the case as a team that concedes might push extra hard for an equalizer or blow up and concede more, however scoring in soccer is random enough to justify making this assumption.

Number 2 says we must know the average number of goals occurring in a game and although we do not know this, by using the over/under 2.5 goals odds, we can solve and estimate it. 

---

### Step 1: Vig Removal

Just before we dive into estimating the number of goals per game, we are going to address something called [vigorish][vig] (or vig for short). Vig is a commission added to the betting odds to ensure the bookmakers can take a profit. For a three-way market (home/draw/away) the implied probabilities, calculated as `1 / decimal_odds`, sum to more than 1. That excess is the bookmaker's cut. Before using the odds for anything we strip it out by normalising:

$$\hat{p}_i = \frac{p_i}{\sum_j p_j}$$

For example, odds of 1.65 / 3.80 / 5.50 produce raw probabilities that sum to 1.051. After normalisation: home 57.7%, draw 25.0%, away 17.3%.

---

### Step 2: Solving for Expected Goals (λ)

We model total goals as a Poisson random variable with unknown mean λ. The Poisson PMF is:

$$P(X = k) = \frac{\lambda^k e^{-\lambda}}{k!}$$

The over/under 2.5 market gives us P(goals ≥ 3) directly. We need the λ that satisfies:

$$P(X \geq 3 \mid \lambda) = 1 - \sum_{k=0}^{2} \frac{\lambda^k e^{-\lambda}}{k!} = p_{\text{over}}$$

There is no closed-form solution, so we define:

$$f(\lambda) = P(X \geq 3 \mid \lambda) - p_{\text{over}} = 0$$

and solve numerically using [Brent's method][brent]. There is no possible way to rearange for λ, so we have to use a numerical method like Brent's. A market implying 55% chance of over 2.5 goals solves to λ ≈ 2.88.

---

### Step 3: Splitting λ Between Teams

We now have total expected goals but need to assign them to each team. We use the 1X2 win probabilities as a proxy for relative attacking strength, the stronger team gets a proportionally larger share:

$$s = \frac{\hat{p}_{\text{home}}}{\hat{p}_{\text{home}} + \hat{p}_{\text{away}}}$$

$$\lambda_h = \lambda \cdot s \qquad \lambda_a = \lambda \cdot (1 - s)$$

For our example: s = 0.577 / (0.577 + 0.173) = 0.769, giving λ_h = 2.21 and λ_a = 0.67.

---

### Known Limitations

**Draw probability is discarded in the split.** The proportional split only uses win probabilities. A 25% draw probability implies the two teams' lambdas should be close together; 10% implies they should be far apart. This information is currently unused.

**No team news or lineups.** The model doesn't know if a key player is injured or if a team is already qualified or knocked out. The market prices result probabilities but not always the scoreline distribution cleanly.

## Making the predictions

The following structure will be used to make predictions:
1. For the first round use models predictions as is regardless of team news or injury.
1. At the conclusion of round 1, look over injuries/team news to see if any outliers can be identified. I.e predictions under or over predicting goals.
1. Get updated betting odds from API before the start of round 2.
1. Use model predictions again, but adjust by a goal up or down depending on whether teams are missing key players. 
1. Same idea for final round, depending on how predictions are going two options. If good then continue with the current method, then take a look at most common score lines so far in tournament and use those for each prediction in hopes to make up some points be hitting correct scores.

Below are the following predictions for each group stage game and will be updated and dated as they are played.

### Group Stage 1

Last updated: June 12, 2026

<style>
.wc-table { width: 100%; border-collapse: collapse; font-size: 14px; }
.wc-table th { text-align: left; font-weight: 500; font-size: 11px; text-transform: uppercase; letter-spacing: 0.08em; color: #888; padding: 6px 10px; border-bottom: 1px solid #e8e8e8; }
.wc-table td { padding: 8px 10px; border-bottom: 1px solid #f0f0f0; vertical-align: middle; }
.wc-table tr:last-child td { border-bottom: none; }
.wc-table tr:hover td { background: #fafafa; }
.wc-score { font-weight: 600; font-size: 15px; font-family: monospace; white-space: nowrap; }
.wc-actual { font-family: monospace; font-size: 14px; white-space: nowrap; }
.wc-xg { font-size: 12px; color: #999; font-family: monospace; white-space: nowrap; }
.wc-date { font-size: 11px; color: #aaa; white-space: nowrap; }
.wc-hit { font-size: 11px; padding: 1px 6px; border-radius: 3px; white-space: nowrap; }
.wc-hit.exact  { background: #edf7ee; color: #2d7a35; }
.wc-hit.result { background: #fdf3e3; color: #a06010; }
.wc-hit.wrong  { background: #fdf0f0; color: #a03030; }
.wc-hit.tbd    { color: #ccc; }
</style>

<table class="wc-table">
  <thead>
    <tr>
      <th>Date</th>
      <th>Match</th>
      <th>Predicted</th>
      <th>xG</th>
      <th>Actual</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="wc-tbody"></tbody>
</table>

<script>
(function() {
  const m = [
    ["Jun 11","Mexico","South Africa",2.08,0.34,"2-0"],
    ["Jun 12","South Korea","Czech Republic",1.25,1.14,"2-1"],
    ["Jun 12","Canada","Bosnia & Herz.",1.74,0.69,""],
    ["Jun 13","USA","Paraguay",1.62,0.75,""],
    ["Jun 13","Qatar","Switzerland",0.24,2.74,""],
    ["Jun 13","Brazil","Morocco",1.95,0.59,""],
    ["Jun 14","Haiti","Scotland",0.56,2.16,""],
    ["Jun 14","Australia","Turkey",0.64,1.93,""],
    ["Jun 14","Germany","Curaçao",4.19,0.11,""],
    ["Jun 14","Netherlands","Japan",1.70,0.94,""],
    ["Jun 14","Ivory Coast","Ecuador",0.83,1.22,""],
    ["Jun 15","Sweden","Tunisia",1.63,0.73,""],
    ["Jun 15","Spain","Cape Verde",3.46,0.15,""],
    ["Jun 15","Belgium","Egypt",2.00,0.61,""],
    ["Jun 15","Saudi Arabia","Uruguay",0.42,2.20,""],
    ["Jun 16","Iran","New Zealand",1.68,0.67,""],
    ["Jun 16","France","Senegal",2.21,0.45,""],
    ["Jun 16","Iraq","Norway",0.25,2.85,""],
    ["Jun 17","Argentina","Algeria",2.31,0.36,""],
    ["Jun 17","Austria","Jordan",2.66,0.38,""],
    ["Jun 17","Portugal","DR Congo",2.61,0.31,""],
    ["Jun 17","England","Croatia",1.86,0.66,""],
    ["Jun 17","Ghana","Panama",1.48,0.90,""],
    ["Jun 18","Uzbekistan","Colombia",0.36,2.27,""],
  ];

  function outcome(pred, actual) {
    if (!actual) return '<span class="wc-hit tbd">–</span>';
    if (pred === actual) return '<span class="wc-hit exact">exact</span>';
    const pr = pred.split('-'), ar = actual.split('-');
    const ps = Math.sign(pr[0]-pr[1]), as = Math.sign(ar[0]-ar[1]);
    return ps === as
      ? '<span class="wc-hit result">result</span>'
      : '<span class="wc-hit wrong">wrong</span>';
  }

  const tb = document.getElementById('wc-tbody');
  m.forEach(function(r) {
    const lh=r[3], la=r[4], actual=r[5];
    const pred = Math.round(lh)+'-'+Math.round(la);
    tb.innerHTML +=
      '<tr>' +
      '<td class="wc-date">'+r[0]+'</td>' +
      '<td>'+r[1]+' <span style="color:#ccc">vs</span> '+r[2]+'</td>' +
      '<td class="wc-score">'+pred+'</td>' +
      '<td class="wc-xg">'+lh.toFixed(2)+' / '+la.toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(pred, actual)+'</td>' +
      '</tr>';
  });
})();
</script>

### Group Stage 2

TBP (To be predicted)

### Group Stage 3

TBP

[superbru]: https://www.superbru.com/
[historical]: https://www.footballhistory.org/world-cup/statistics.html
[historical2022]: https://www.thesoccerworldcups.com/statistics/wc_by_wc.php
[oddsapi]: https://the-odds-api.com/
[poisson]: https://www.scribbr.com/statistics/poisson-distribution/
[vig]: https://www.legalsportsreport.com/how-to-bet/vigorish/
[brent]: https://mathworld.wolfram.com/BrentsMethod.html
[1x2]: https://help.smarkets.com/hc/en-gb/articles/214108649-What-is-1X2-betting
[overunder]: https://www.cbssports.com/betting/news/over-under-betting/

## Wrap Up

Updated June 12, 2026

I don't have much to say right now other than GO CANADA, and hopefully these predictions can serve me well. I will at the least update this post at the end of every groupstage game, but hopefully more often than that. Anyways I appreciate those who made it this far and best of luck to you and your team you will be cheering on this World Cup! 