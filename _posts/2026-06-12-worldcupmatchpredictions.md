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

where `p_i = 1 / decimal_odds_i` is the raw implied probability for outcome `i`, and the sum in the denominator runs over all outcomes `j` in the market. For example, odds of 1.65 / 3.80 / 5.50 produce raw probabilities that sum to 1.051. After normalisation: home 57.7%, draw 25.0%, away 17.3%.

---

### Step 2: Solving for Expected Goals (λ)

We model total goals as a Poisson random variable with unknown mean λ (the average number of goals we expect in the match). The Poisson PMF is:

$$P(X = k) = \frac{\lambda^k e^{-\lambda}}{k!}$$

where X is the total number of goals scored, k is a specific goal count, and e is Euler's number. The over/under 2.5 market gives us P(goals ≥ 3) directly. We need the λ that satisfies:

$$P(X \geq 3 \mid \lambda) = 1 - \sum_{k=0}^{2} \frac{\lambda^k e^{-\lambda}}{k!} = p_{\text{over}}$$

where `p_over` is the normalised implied probability from the over 2.5 odds. There is no closed-form solution, so we define:

$$f(\lambda) = P(X \geq 3 \mid \lambda) - p_{\text{over}} = 0$$

and solve numerically using [Brent's method][brent]. There is no possible way to rearrange for λ, so we have to use a numerical method like Brent's. A market implying 55% chance of over 2.5 goals solves to λ ≈ 2.88.


---

### Step 3: Splitting λ Between Teams

We now have total expected goals but need to assign them to each team. We use the 1X2 win probabilities as a proxy for relative team strength, the stronger team gets a proportionally larger share:

$$s = \frac{\hat{p}_{\text{home}}}{\hat{p}_{\text{home}} + \hat{p}_{\text{away}}}$$

$$\lambda_h = \lambda \cdot s \qquad \lambda_a = \lambda \cdot (1 - s)$$

where s is the home team's share of the two-outcome (home/away) probability, `p_home` and `p_away` are the normalised win probabilities from Step 1, and `lambda_h`, `lambda_a` are the resulting expected goals for the home and away team respectively. For our example: s = 0.577 / (0.577 + 0.173) = 0.769, giving λ_h = 2.21 and λ_a = 0.67.

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
1. Same idea for final round, depending on how predictions are going two options. If good then continue with the current method, then take a look at most common score lines so far in tournament and use those for each prediction in hopes to make up some points by hitting correct scores.

Below are the following predictions for each group stage game and will be updated and dated as they are played.

### Group Stage 1

Last updated: June 19, 2026

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
    ["Jun 12","Canada","Bosnia & Herz.",1.74,0.69,"1-1"],
    ["Jun 13","USA","Paraguay",1.62,0.75,"4-1"],
    ["Jun 13","Qatar","Switzerland",0.24,2.74,"1-1"],
    ["Jun 13","Brazil","Morocco",1.95,0.59,"1-1"],
    ["Jun 14","Haiti","Scotland",0.56,2.16,"0-1"],
    ["Jun 14","Australia","Turkey",0.64,1.93,"2-0"],
    ["Jun 14","Germany","Curaçao",4.19,0.11,"7-1"],
    ["Jun 14","Netherlands","Japan",1.70,0.94,"2-2"],
    ["Jun 14","Ivory Coast","Ecuador",0.83,1.22,"1-0"],
    ["Jun 15","Sweden","Tunisia",1.63,0.73,"5-1"],
    ["Jun 15","Spain","Cape Verde",3.46,0.15,"0-0"],
    ["Jun 15","Belgium","Egypt",2.00,0.61,"1-1"],
    ["Jun 15","Saudi Arabia","Uruguay",0.42,2.20,"1-1"],
    ["Jun 16","Iran","New Zealand",1.68,0.67,"2-2"],
    ["Jun 16","France","Senegal",2.21,0.45,"3-1"],
    ["Jun 16","Iraq","Norway",0.25,2.85,"1-4"],
    ["Jun 17","Argentina","Algeria",2.31,0.36,"3-0"],
    ["Jun 17","Austria","Jordan",2.66,0.38,"3-1"],
    ["Jun 17","Portugal","DR Congo",2.61,0.31,"1-1"],
    ["Jun 17","England","Croatia",1.86,0.66,"4-2"],
    ["Jun 17","Ghana","Panama",1.48,0.90,"1-0"],
    ["Jun 18","Uzbekistan","Colombia",0.36,2.27,"1-3"],
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
      '<td>'+r[1]+' <span style="color:#ccc">vs</span> '+r[2]+'</td>' +
      '<td class="wc-score">'+pred+'</td>' +
      '<td class="wc-xg">'+lh.toFixed(2)+' / '+la.toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(pred, actual)+'</td>' +
      '</tr>';
  });
})();
</script>

#### Group Stage 1 Review

After 24 games the model is holding up reasonably well on results (46% correct W/D/L vs 33% random baseline) but is systematically underestimating goals, missing by 0.42 per game on average.

The bias is largely outlier-driven — Germany 7-1, Sweden 5-1, England 4-2, and USA 4-1 account for 11 of the 23 missing goals. Strip those blowouts out and the model is close to calibrated. For competitive games it's performing as expected.

The bigger pattern is a **goal distribution mismatch**: the model clusters predictions around 3-goal games but the tournament is running hot, with 42% of matches producing 4+ goals and 1-1 appearing six times as the most common scoreline. The model generated zero sub-2-goal predictions; in reality 4 games finished with 1 goal or fewer.

For matchday 2 I'll be nudging λ upward selectively for mismatched fixtures rather than applying a global adjustment — the blowout games were all heavy favourites against weak opposition, so that's where the model leaves the most on the table. Although 1-1 is the most common scoreline and my model rarely predicting, I am not going to adjust anything here yet and let the model run on the updated odds once again.

### Group Stage 2

Updated June 24

Only change I've made is a few 3-0 games bumped to 4-0. Purely based on strong teams playing weaker and the estimated xG being above 3 for the stronger team.

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
      <th>Match</th>
      <th>Predicted</th>
      <th>xG</th>
      <th>Actual</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="wc-tbody-r2"></tbody>
</table>

<script>
(function() {
  const m = [
    ["Jun 18","Czech Republic","South Africa",1.60,0.76,"1-1"],
    ["Jun 18","Switzerland","Bosnia & Herzegovina",2.01,0.55,"4-1"],
    ["Jun 18","Canada","Qatar",2.36,0.29,"6-0"],
    ["Jun 19","Mexico","South Korea",1.73,0.67,"1-0"],
    ["Jun 19","USA","Australia",1.83,0.71,"2-0"],
    ["Jun 19","Scotland","Morocco",0.76,1.58,"0-1"],
    ["Jun 20","Brazil","Haiti",3.69,0.15,"3-0"],
    ["Jun 20","Turkey","Paraguay",1.41,0.92,"0-1"],
    ["Jun 20","Netherlands","Sweden",2.02,0.64,"5-1"],
    ["Jun 20","Germany","Ivory Coast",2.17,0.60,"2-1"],
    ["Jun 21","Ecuador","Curaçao",2.74,0.24,"0-0"],
    ["Jun 21","Tunisia","Japan",0.59,1.80,"0-4"],
    ["Jun 21","Spain","Saudi Arabia",3.12,0.15,"4-0"],
    ["Jun 21","Belgium","Iran",2.30,0.40,"0-0"],
    ["Jun 21","Uruguay","Cape Verde",2.23,0.43,"2-2"],
    ["Jun 22","New Zealand","Egypt",0.63,1.77,"1-3"],
    ["Jun 22","Argentina","Austria",2.01,0.61,"2-0"],
    ["Jun 22","France","Iraq",3.12,0.15,"3-0"],
    ["Jun 23","Norway","Senegal",1.60,1.02,"3-2"],
    ["Jun 23","Jordan","Algeria",0.51,2.15,"1-2"],
    ["Jun 23","Portugal","Uzbekistan",2.87,0.32,"5-0"],
    ["Jun 23","England","Ghana",2.57,0.36,"0-0"],
    ["Jun 23","Panama","Croatia",0.50,2.07,"0-1"],
    ["Jun 24","Colombia","DR Congo",2.03,0.41,"1-0"],
  ];

  const pred = {
    "Czech Republic vs South Africa": "2-1",
    "Switzerland vs Bosnia & Herzegovina": "2-1",
    "Canada vs Qatar": "3-0",
    "Mexico vs South Korea": "2-1",
    "USA vs Australia": "2-1",
    "Scotland vs Morocco": "1-2",
    "Brazil vs Haiti": "4-0",
    "Turkey vs Paraguay": "2-1",
    "Netherlands vs Sweden": "2-1",
    "Germany vs Ivory Coast": "2-1",
    "Ecuador vs Curaçao": "3-0",
    "Tunisia vs Japan": "0-2",
    "Spain vs Saudi Arabia": "3-0",
    "Belgium vs Iran": "2-0",
    "Uruguay vs Cape Verde": "2-0",
    "New Zealand vs Egypt": "1-2",
    "Argentina vs Austria": "2-1",
    "France vs Iraq": "4-0",
    "Norway vs Senegal": "1-1",
    "Jordan vs Algeria": "1-2",
    "Portugal vs Uzbekistan": "3-0",
    "England vs Ghana": "3-0",
    "Panama vs Croatia": "0-2",
    "Colombia vs DR Congo": "2-0",
  };

  function outcome(p, actual) {
    if (!actual) return '<span class="wc-hit tbd">–</span>';
    if (p === actual) return '<span class="wc-hit exact">exact</span>';
    const pr = p.split('-'), ar = actual.split('-');
    const ps = Math.sign(pr[0]-pr[1]), as = Math.sign(ar[0]-ar[1]);
    return ps === as
      ? '<span class="wc-hit result">result</span>'
      : '<span class="wc-hit wrong">wrong</span>';
  }

  const tb = document.getElementById('wc-tbody-r2');
  m.forEach(function(r) {
    const key = r[1]+' vs '+r[2];
    const p = pred[key] || (Math.round(r[3])+'-'+Math.round(r[4]));
    const actual = r[5];
    tb.innerHTML +=
      '<tr>' +
      '<td>'+r[1]+' <span style="color:#ccc">vs</span> '+r[2]+'</td>' +
      '<td class="wc-score">'+p+'</td>' +
      '<td class="wc-xg">'+r[3].toFixed(2)+' / '+r[4].toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(p, actual||'')+'</td>' +
      '</tr>';
  });
})();
</script>

After round 2, a few notes. The model has predicted 29/48 games, increasing accuracy to around 60%. I can't be mad at this and it is probably performing a bit better than I expected. Unfortunately, my idea of bumping some 3-0 games up to 4-0 has backfired because I bumped the wrong 3-0's up and left the ones that actually went 4-0 as 3-0. I should have probably either bumped all to 4-0 or left all at 3-0, to try and collect as many exact scores as possible.

Where the model has dissapointed is in exact scores. I have only got 3 exact scores so far, which means I am missing out on a lot of points. The tournament has seen nineteen 4+ goal games and I have only predicted 3. There were also a combined twelve 0 or 1 goal games where I predicted none. The model predicted thirty-one 3 goal games and only six occurred. 2 goal games were predicted fourteen times and actually happened eleven times. It clear we are predicting 3 goal games far too often and neglecting low scoring games. I think due to variance its much harder to predict high scoring games, so for round 3 we should focus our efforts on predicting more of the low-scoring outcomes.

### Group Stage 3

Updated June 24

So now what? Going into the final group stage games I have created a decision tree to help make my predictions. I will go back to the model predictions in the knockouts.

Are win probabilities within 20% of each other?
 - → Yes: does one team have a clear incentive?
 - -  → Yes: predict 1-0
 - -  → No: predict 0-0 or 1-1, whichever's closer to prediction.
 - → No (clear favourite): Does model predict 3-0 or higher?
 - -  → Yes: drop to 2-0 
 - -  → No: drop predicted goals by 1 (2-1 → 1-0, 2-0 → 1-0)

With this set of logic the idea is to target the low-scoring games we have missed out on thus far. Given the final round I think a lot of teams will be set up very defensively with their tournaments on the line so I don't see why low-scoring shouldn't continue to happen. With that being said here are my picks:

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
      <th>Match</th>
      <th>Predicted</th>
      <th>xG</th>
      <th>Actual</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="wc-tbody-r3"></tbody>
</table>

<script>
(function() {
  const m = [
    ["Jun 24","Bosnia & Herzegovina","Qatar",2.65,0.47,"3-1"],
    ["Jun 24","Switzerland","Canada",1.44,1.05,"2-1"],
    ["Jun 24","Scotland","Brazil",0.33,2.47,"0-3"],
    ["Jun 24","Morocco","Haiti",2.94,0.21,"4-2"],
    ["Jun 25","Czech Republic","Mexico",0.87,1.71,"0-3"],
    ["Jun 25","South Africa","South Korea",0.57,2.00,"1-0"],
    ["Jun 25","Curaçao","Ivory Coast",0.20,3.05,"0-2"],
    ["Jun 25","Ecuador","Germany",0.93,1.93,"2-1"],
    ["Jun 25","Japan","Sweden",2.00,0.87,"1-1"],
    ["Jun 25","Tunisia","Netherlands",0.16,3.17,"1-3"],
    ["Jun 26","Paraguay","Australia",1.14,0.84,"0-0"],
    ["Jun 26","Turkey","USA",1.01,1.89,"3-2"],
    ["Jun 26","Norway","France",0.78,2.18,"1-4"],
    ["Jun 26","Senegal","Iraq",2.93,0.29,"5-0"],
    ["Jun 27","Cape Verde","Saudi Arabia",1.39,1.17,"0-0"],
    ["Jun 27","Uruguay","Spain",0.49,2.23,"0-1"],
    ["Jun 27","New Zealand","Belgium",0.23,2.95,"1-5"],
    ["Jun 27","Egypt","Iran",1.26,0.84,"1-1"],
    ["Jun 27","Croatia","Ghana",1.81,0.58,"2-1"],
    ["Jun 27","Panama","England",0.24,3.13,"0-2"],
    ["Jun 27","Colombia","Portugal",0.82,1.73,"0-0"],
    ["Jun 27","DR Congo","Uzbekistan",1.87,0.81,"3-1"],
    ["Jun 28","Algeria","Austria",0.89,1.22,"3-3"],
    ["Jun 28","Jordan","Argentina",0.24,2.86,"1-3"],
  ];

  const pred = {
    "Bosnia & Herzegovina vs Qatar": "2-0",
    "Switzerland vs Canada": "1-1",
    "Scotland vs Brazil": "0-1",
    "Morocco vs Haiti": "2-0",
    "Czech Republic vs Mexico": "0-1",
    "South Africa vs South Korea": "0-1",
    "Curaçao vs Ivory Coast": "0-2",
    "Ecuador vs Germany": "0-1",
    "Japan vs Sweden": "1-0",
    "Tunisia vs Netherlands": "0-2",
    "Paraguay vs Australia": "1-1",
    "Turkey vs USA": "0-1",
    "Norway vs France": "0-1",
    "Senegal vs Iraq": "2-0",
    "Cape Verde vs Saudi Arabia": "1-1",
    "Uruguay vs Spain": "0-1",
    "New Zealand vs Belgium": "0-2",
    "Egypt vs Iran": "0-1",
    "Croatia vs Ghana": "1-0",
    "Panama vs England": "0-2",
    "Colombia vs Portugal": "0-1",
    "DR Congo vs Uzbekistan": "1-0",
    "Algeria vs Austria": "1-1",
    "Jordan vs Argentina": "0-2",
  };

  function outcome(p, actual) {
    if (!actual) return '<span class="wc-hit tbd">–</span>';
    if (p === actual) return '<span class="wc-hit exact">exact</span>';
    const pr = p.split('-'), ar = actual.split('-');
    const ps = Math.sign(pr[0]-pr[1]), as = Math.sign(ar[0]-ar[1]);
    return ps === as
      ? '<span class="wc-hit result">result</span>'
      : '<span class="wc-hit wrong">wrong</span>';
  }

  const tb = document.getElementById('wc-tbody-r3');
  m.forEach(function(r) {
    const key = r[1]+' vs '+r[2];
    const p = pred[key] || (Math.round(r[3])+'-'+Math.round(r[4]));
    const actual = r[5];
    tb.innerHTML +=
      '<tr>' +
      '<td>'+r[1]+' <span style="color:#ccc">vs</span> '+r[2]+'</td>' +
      '<td class="wc-score">'+p+'</td>' +
      '<td class="wc-xg">'+r[3].toFixed(2)+' / '+r[4].toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(p, actual||'')+'</td>' +
      '</tr>';
  });
})();
</script>

### Round of 32

Updated June 29

Round of 32 up next. I ended up going 17/24 correct outcomes with 3 exact last round. The method of predicting low scoring, got us a good return but not much different that the model in round 2. Because of this and the fact that knockouts begin, I am going to go back to using the model predictions as is. We will reassess after the round of 32. In the league I am playing in I currently in 67/411. So not great but not terrible either. Last round shot us up a bunch so hopefully we will conintue to climb. Anyways here are my predictions.

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
      <th>Match</th>
      <th>Predicted</th>
      <th>xG</th>
      <th>Actual</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="wc-tbody-r16"></tbody>
</table>

<script>
(function() {
  const m = [
    ["Jun 28","South Africa","Canada",0.54,1.85,"1-2"],
    ["Jun 29","Brazil","Japan",1.98,0.65,"2-1"],
    ["Jun 29","Germany","Paraguay",2.58,0.35,"1-1"],
    ["Jun 30","Netherlands","Morocco",1.52,0.91,"1-1"],
    ["Jun 30","Ivory Coast","Norway",0.96,1.76,"1-2"],
    ["Jun 30","France","Sweden",2.91,0.33,"3-0"],
    ["Jul 1","Mexico","Ecuador",1.30,0.78,"2-0"],
    ["Jul 1","England","DR Congo",2.50,0.24,"2-1"],
    ["Jul 1","Belgium","Senegal",1.47,0.87,"3-2"],
    ["Jul 2","USA","Bosnia & Herzegovina",2.43,0.36,"2-0"],
    ["Jul 2","Spain","Austria",2.42,0.26,"3-0"],
    ["Jul 2","Portugal","Croatia",1.77,0.67,"2-1"],
    ["Jul 3","Switzerland","Algeria",1.79,0.78,"2-0"],
    ["Jul 3","Australia","Egypt",0.89,1.19,"1-1"],
    ["Jul 3","Argentina","Cape Verde",2.91,0.17,"3-2"],
    ["Jul 4","Colombia","Ghana",1.92,0.45,"1-0"],
  ];

  function outcome(p, actual) {
    if (!actual) return '<span class="wc-hit tbd">–</span>';
    if (p === actual) return '<span class="wc-hit exact">exact</span>';
    const pr = p.split('-'), ar = actual.split('-');
    const ps = Math.sign(pr[0]-pr[1]), as = Math.sign(ar[0]-ar[1]);
    return ps === as
      ? '<span class="wc-hit result">result</span>'
      : '<span class="wc-hit wrong">wrong</span>';
  }

  const tb = document.getElementById('wc-tbody-r16');
  m.forEach(function(r) {
    const p = Math.round(r[3])+'-'+Math.round(r[4]);
    const actual = r[5];
    tb.innerHTML +=
      '<tr>' +
      '<td>'+r[1]+' <span style="color:#ccc">vs</span> '+r[2]+'</td>' +
      '<td class="wc-score">'+p+'</td>' +
      '<td class="wc-xg">'+r[3].toFixed(2)+' / '+r[4].toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(p, actual||'')+'</td>' +
      '</tr>';
  });
})();
</script>

We continue to climb getting up to around 30th in my office pool. Best round yet with 68% accuracy and 37.5% exact scorelines. I'll chalk it up that we were just due for some good fortune, but we will leave the model as is going into the next round and see how it holds. 

### Round of 16

Updated July 4

Nothing new here, model predictions are as follows:

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
      <th>Match</th>
      <th>Predicted</th>
      <th>xG</th>
      <th>Actual</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="wc-tbody-r16b"></tbody>
</table>

<script>
(function() {
  const m = [
    ["Jul 4","Canada","Morocco",0.68,1.72,"0-3"],
    ["Jul 4","Paraguay","France",0.18,2.80,"0-1"],
    ["Jul 5","Brazil","Norway",2.02,0.85,"1-2"],
    ["Jul 6","Mexico","England",1.00,1.27,"2-3"],
    ["Jul 6","Portugal","Spain",0.86,1.84,"0-1"],
    ["Jul 7","USA","Belgium",1.37,1.47,"1-4"],
    ["Jul 7","Argentina","Egypt",2.26,0.32,"3-2"],
    ["Jul 7","Switzerland","Colombia",0.92,1.43,"0-0"],
  ];

  const pred = {
    "Canada vs Morocco": "1-2",
    "Paraguay vs France": "0-3",
    "Brazil vs Norway": "2-1",
    "Mexico vs England": "1-1",
    "Portugal vs Spain": "1-2",
    "USA vs Belgium": "1-1",
    "Argentina vs Egypt": "2-0",
    "Switzerland vs Colombia": "1-1",
  };

  function outcome(p, actual) {
    if (!actual) return '<span class="wc-hit tbd">–</span>';
    if (p === actual) return '<span class="wc-hit exact">exact</span>';
    const pr = p.split('-'), ar = actual.split('-');
    const ps = Math.sign(pr[0]-pr[1]), as = Math.sign(ar[0]-ar[1]);
    return ps === as
      ? '<span class="wc-hit result">result</span>'
      : '<span class="wc-hit wrong">wrong</span>';
  }

  const tb = document.getElementById('wc-tbody-r16b');
  m.forEach(function(r) {
    const key = r[1]+' vs '+r[2];
    const p = pred[key] || (Math.round(r[3])+'-'+Math.round(r[4]));
    const actual = r[5];
    tb.innerHTML +=
      '<tr>' +
      '<td>'+r[1]+' <span style="color:#ccc">vs</span> '+r[2]+'</td>' +
      '<td class="wc-score">'+p+'</td>' +
      '<td class="wc-xg">'+r[3].toFixed(2)+' / '+r[4].toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(p, actual||'')+'</td>' +
      '</tr>';
  });
})();
</script>

### Quarter Finals

Updated July 9

No exact scores for the round of 16. The model had England and Belgium tying which personally I would've picked wins for both, but other than that some pretty unpredictable scorelines.

The most common scorelines are 1-0 with 14, 1-1 with 12, 2-1 with 11, 2-0 with 9, and 0-0 with 8. The model has the favorites winning 2-1 in every game of the quarterfinals. Although personally I would go with 1-0 for some games given it's late in the world cup and teams will probably tend to start favoring defense, I think picking the same score for every match gives me a good chance of getting exact scores, especially since 2-1 is common. Part of me wants to go 1-0 for each, but I will put some faith in the model and let it try and prove itself once again.

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
      <th>Match</th>
      <th>Predicted</th>
      <th>xG</th>
      <th>Actual</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="wc-tbody-qf"></tbody>
</table>

<script>
(function() {
  const m = [
    ["Jul 9","France","Morocco",2.06,0.51,"2-0"],
    ["Jul 10","Spain","Belgium",2.17,0.65,"2-1"],
    ["Jul 11","Norway","England",0.88,1.98,"1-2"],
    ["Jul 12","Argentina","Switzerland",1.83,0.56,"3-1"],
  ];

  const pred = {
    "France vs Morocco": "2-1",
    "Spain vs Belgium": "2-1",
    "Norway vs England": "1-2",
    "Argentina vs Switzerland": "2-1",
  };

  function outcome(p, actual) {
    if (!actual) return '<span class="wc-hit tbd">–</span>';
    if (p === actual) return '<span class="wc-hit exact">exact</span>';
    const pr = p.split('-'), ar = actual.split('-');
    const ps = Math.sign(pr[0]-pr[1]), as = Math.sign(ar[0]-ar[1]);
    return ps === as
      ? '<span class="wc-hit result">result</span>'
      : '<span class="wc-hit wrong">wrong</span>';
  }

  const tb = document.getElementById('wc-tbody-qf');
  m.forEach(function(r) {
    const key = r[1]+' vs '+r[2];
    const p = pred[key] || (Math.round(r[3])+'-'+Math.round(r[4]));
    const actual = r[5];
    tb.innerHTML +=
      '<tr>' +
      '<td>'+r[1]+' <span style="color:#ccc">vs</span> '+r[2]+'</td>' +
      '<td class="wc-score">'+p+'</td>' +
      '<td class="wc-xg">'+r[3].toFixed(2)+' / '+r[4].toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(p, actual||'')+'</td>' +
      '</tr>';
  });
})();
</script>

### Semi Finals

Updated July 12

Not bad quarter finals. 2 exact and 2 close. I am currently sitting around 30th out of 250 in the office league. Trusting the model paid off, I think picking the same score for every game at this point is a good strategy. However, I will still follow the model for the semis. Both are close games and I think the only way to really make up any serious ground is to get exact for both.

<table class="wc-table">
  <thead>
    <tr>
      <th>Match</th>
      <th>Predicted</th>
      <th>xG</th>
      <th>Actual</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="wc-tbody-sf"></tbody>
</table>

<script>
(function() {
  const m = [
    ["France","Spain",1.56,1.12,""],
    ["England","Argentina",1.26,1.07,""],
  ];

  const pred = {
    "France vs Spain": "2-1",
    "England vs Argentina": "1-1",
  };

  function outcome(p, actual) {
    if (!actual) return '<span class="wc-hit tbd">–</span>';
    if (p === actual) return '<span class="wc-hit exact">exact</span>';
    const pr = p.split('-'), ar = actual.split('-');
    const ps = Math.sign(pr[0]-pr[1]), as = Math.sign(ar[0]-ar[1]);
    return ps === as
      ? '<span class="wc-hit result">result</span>'
      : '<span class="wc-hit wrong">wrong</span>';
  }

  const tb = document.getElementById('wc-tbody-sf');
  m.forEach(function(r) {
    const key = r[0]+' vs '+r[1];
    const p = pred[key] || (Math.round(r[2])+'-'+Math.round(r[3]));
    const actual = r[4];
    tb.innerHTML +=
      '<tr>' +
      '<td>'+r[0]+' <span style="color:#ccc">vs</span> '+r[1]+'</td>' +
      '<td class="wc-score">'+p+'</td>' +
      '<td class="wc-xg">'+r[2].toFixed(2)+' / '+r[3].toFixed(2)+'</td>' +
      '<td class="wc-actual">'+(actual||'–')+'</td>' +
      '<td>'+outcome(p, actual||'')+'</td>' +
      '</tr>';
  });
})();
</script>

## Wrap Up

Updated June 12, 2026

I don't have much to say right now other than GO CANADA, and hopefully these predictions can serve me well. I will at the least update this post at the end of every groupstage round, but hopefully more often than that. Anyways I appreciate those who made it this far and best of luck to you and your team you will be cheering on this World Cup! 


[superbru]: https://www.superbru.com/
[historical]: https://www.footballhistory.org/world-cup/statistics.html
[historical2022]: https://www.thesoccerworldcups.com/statistics/wc_by_wc.php
[oddsapi]: https://the-odds-api.com/
[poisson]: https://www.scribbr.com/statistics/poisson-distribution/
[vig]: https://www.legalsportsreport.com/how-to-bet/vigorish/
[brent]: https://mathworld.wolfram.com/BrentsMethod.html
[1x2]: https://help.smarkets.com/hc/en-gb/articles/214108649-What-is-1X2-betting
[overunder]: https://www.cbssports.com/betting/news/over-under-betting/