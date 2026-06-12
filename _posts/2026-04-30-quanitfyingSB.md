---
layout: post
title: "Quantifying Second Ball Wins"
date: 2026-04-30
categories: [notes]
---

<script type="text/javascript" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>

The evaluation of how in-game actions of soccer players affect the outcome
of a game is an important part of soccer analytics [2]. Unlike traditional
statistics that focus on goals and assists, evaluating other actions that players perform
looks deeper into the nuances of player behaviour and decision-making.

An action that often goes unnoticed is the ability to win second balls [3]. A second
ball is a possession gain following an intervention from an attempted long pass. Simply
put, a second ball occurs after the ball is kicked long and neither team clearly controls
it. Instead, the ball bounces, gets deflected, or is contested, and then players rush
to try and take control of the ball. A quote from Pep Guardiola,
one of the most successful soccer managers of all time, states the importance of winning second
balls, ”The main thing in English football is to control the second ball. Without that,
you cannot survive.” [4].

Despite the importance of winning second balls, there is a lack of research quantifying
or analyzing the value of winning second balls and how to do so successfully. Therefore this post aims to extend the research done and pose future direction for further work/extensions. 

## Definition and Example

Let's begin by establishing a working definition for second balls. The following definitions have been adapted from a [blog][chun-hang] post by Chun Hang, but remain largely unchanged since I think they are a great definition.

<div class="definition">
  <div class="box-title">Definition 1 — Second Ball</div>
  The loose ball that results from an intervention following a long pass or clearance.
  <ol>
    <li>Long Pass (long ball): A pass with a length of 20 metres or greater that any player meets, successful or not. </li>
    <li>Interventions: The first point of contact with the long ball must be an intervention. This intervention can be in the form of an aerial duel between players, a clearance attempt, or miscontrol. The key part of the interaction is that it leads to a loose ball.</li>
    
  </ol>
  
</div>

<div class="definition">
  <div class="box-title">Definition 2 - Second Ball Win</div>
  A possession gain or regain immediately following a second ball.
  <ol>
   <li>Possession Gain/Regain: A player controls the second ball gaining or regaining possession for their team. Including the player wit the ball, their team must complete two successful actions to be considerd a second ball win.</li>
  </ol>
</div>

And to further build intuition for those who are less familiar with second balls, I think seeing a real example does a great job.

<video width="560" controls playsinline preload="auto">
  <source src="/assets/ABA_example.mp4" type="video/mp4; codecs=avc1.42E01E,mp4a.40.2">
</video>

It is clear from the example that second balls can lead to chaotic and quick attacking chances in favor of the winner. Before beginning to quantify them, let's investigate their importance a little bit more.


## Importance of Second Balls

The following brief analysis examines the relationship between second ball success, goal difference, and overall team performance. It should not be taken as fact but rather a thought provoking analysis to give reason why we should investigate second balls deeper.

### League Performance

Using data from the top 5 European leagues in 2015/2016, a linear regression is performed on goal difference (GD) vs total points earned in a season. 

![Alt text]({{ site.baseurl }}/assets/images/gd_vs_pts.png)

An R^2 value of 0.95 demonstrates a strong linear relationship between GD and points. Based on the regression an increase in GD by 1 corresponds to an expected points increase of 0.65 points. Potentially even marginal improvements in GD can affect a team's league standing position.

### Shot Creation

Now, let's look at the role second balls play in generating goal-scoring opportunities. The following information is extracted from the dataset:
- 19.38% of second ball wins result in a shot within the following possession.
- The average expected goal (xG) value per shot is 10.1%.
- On average teams contest approximately 24 second balls per game.

The above list suggest increasing second ball wins could lead to a higher number of shots for a team. Also note, expected goals is a metric that predicts the probability that a shot results in a goal by using various features such as distance to goal, angle to goal, etc.

### Impact of Winning More Second Balls

Now let's look at the potential impact on a team that wins an additional 5 second balls per match. Based on the dataset:

- Approximately 0.97 additional shots per game, or 36.81 additional shots per season (for a standard 38 game season).
- Given an average xG per shot of 0.101, this translates to 3.72 expected goals gined per season.
- Conversely, by preventing the opponent from winning these second balls, a team can reduce shots conceded, preventing 3.72 expected goals against per season.
- Thus, a team could see a net gain as large as 7.44 GD per season.

### Translating Second Ball Success to League Points

Finally, by using the regression model, a 7.44 GD corresponds to an additional 4.84 points over a season. It may not seem like much, but minor differences in points can define team's seasons as success or failure. Below is the league table from the 2015/2016 premier league, where the data is taken from.

<table>
  <thead>
    <tr>
      <th>Rank</th>
      <th>Team</th>
      <th>Points</th>
    </tr>
  </thead>
  <tbody>
    <tr style="background-color: rgba(46, 204, 113, 0.5);">
      <td>1</td><td>Leicester City</td><td>81</td>
    </tr>
    <tr style="background-color: rgba(241, 196, 15, 0.5);">
      <td>2</td><td>Arsenal</td><td>71</td>
    </tr>
    <tr style="background-color: rgba(241, 196, 15, 0.5);">
      <td>3</td><td>Tottenham</td><td>70</td>
    </tr>
    <tr style="background-color: rgba(241, 196, 15, 0.5);">
      <td>4</td><td>Manchester City</td><td>66</td>
    </tr>
    <tr style="background-color: rgba(241, 196, 15, 0.5);">
      <td>5</td><td>Manchester United</td><td>66</td>
    </tr>
    <tr><td>6</td><td>Southampton</td><td>63</td></tr>
    <tr><td>7</td><td>West Ham</td><td>62</td></tr>
    <tr><td>8</td><td>Liverpool</td><td>60</td></tr>
    <tr><td>9</td><td>Stoke City</td><td>51</td></tr>
    <tr><td>10</td><td>Chelsea</td><td>50</td></tr>
    <tr><td>11</td><td>Everton</td><td>47</td></tr>
    <tr><td>12</td><td>Swansea City</td><td>47</td></tr>
    <tr><td>13</td><td>Watford</td><td>45</td></tr>
    <tr><td>14</td><td>West Brom</td><td>43</td></tr>
    <tr><td>15</td><td>Crystal Palace</td><td>42</td></tr>
    <tr><td>16</td><td>Bournemouth</td><td>42</td></tr>
    <tr><td>17</td><td>Sunderland</td><td>39</td></tr>
    <tr style="background-color: rgba(231, 76, 60, 0.5);">
      <td>18</td><td>Newcastle</td><td>37</td>
    </tr>
    <tr style="background-color: rgba(231, 76, 60, 0.5);">
      <td>19</td><td>Norwich City</td><td>34</td>
    </tr>
    <tr style="background-color: rgba(231, 76, 60, 0.5);">
      <td>20</td><td>Aston Villa</td><td>17</td>
    </tr>
  </tbody>
</table>

<div style="display: flex; gap: 20px; margin-top: 10px; margin-bottom: 30px;">
  <div style="display: flex; align-items: center; gap: 6px;">
    <div style="width: 16px; height: 16px; background-color: rgba(46, 204, 113, 0.5); border: 1px solid rgba(46, 204, 113, 0.8);"></div>
    <span>Champions</span>
  </div>
  <div style="display: flex; align-items: center; gap: 6px;">
    <div style="width: 16px; height: 16px; background-color: rgba(241, 196, 15, 0.5); border: 1px solid rgba(241, 196, 15, 0.8);"></div>
    <span>European Qualification</span>
  </div>
  <div style="display: flex; align-items: center; gap: 6px;">
    <div style="width: 16px; height: 16px; background-color: rgba(231, 76, 60, 0.5); border: 1px solid rgba(231, 76, 60, 0.8);"></div>
    <span>Relegation</span>
  </div>
</div>

## Other Second Ball Research

To date, only one research [paper][sunjic] has directly analyzed second balls, while few online blogs/articles offer exploratory analyses ([here][chun-hang] and [here][nyt]).

[Sunjic et al.][sunjic] analyze how second ball wins correlate with technical performance such as shots, passes, and dribbles. [Carey and Walid][nyt] provide really cool graphics about premier league teams abilities from winning second balls to even looking at the number of defensive vs offensive second balls won for each team. While both very interesting, neither provide a clear mathematical definition that allows for algorithmic extraction. 

[Hang][chun-hang] addresses this problem by creating an idea called "Second Ball Chains", a chain of events that denote the actions leading up to a second ball. An example can be found below.

ABABA SECOND BALL EXAMPLE.

The two problems I am trying to address in this post are:
 - Naming Conventions
 - Higher-order balls (third balls, fourth balls, and so on)

## Second Ball Chains

The original second ball chains naming convention can be confusing. For example, ABABA means Team A long ball, followed by an aerial duel in which Team B wins, followed by Team A gaining the ball from the duel. I argue this is not clear from the above example. Instead it should look something like this:

ADBA EXAMPLE CHAIN

## Higher Order Balls



## Second Ball Possessions

Now that we have updated second ball chains, we now define what happens after a second ball win: a second ball possession. The following defintion sets up future analysis (my [next][blogpost2] post).

SECOND BALL POSSESSION DEF

## Mathematical Definition of Second Balls

Now we define them mathematically to allow for automated extraction from a dataset. 

FORMAL DEFINTION

## Results

Finally, we have a few results. Similiarly to [Hang][chun-hang], we can create a rankings for players and teams of second ball wins. Furthermore, we can create visuals that show the locations of every second ball win on a player and team level.

### Second Ball Rankings

QUANTITY TABLE HERE

Again this result is not new, first being shown by [Hang][chun-hang], it has just been updated to incorporate our new framework including the updated second ball chains and higher order balls.

### Second Ball Heatmaps

By using the second ball algorithms we can create visuals that show where on the field second balls occur and which result in shots created in the subsequent second ball possession.

DECLAN RICE VISUAL HERE

It can also be done for a team, including when they lose a second ball. We can use these maps to visualize players second ball winning habits and which spaces on the pitch they occupy often.

ENGLAND VISUAL HERE

Although England won more second balls than they lost, there were more shots conceded than created. A likely explanation is that since England often dominates possession, opposition may focus on counterattacks through a direct, second ball inducing style of play, with the intention of creating attacking chances quickly after possession regain.

## Conclusions

In this post we investigated quantifying second balls in soccer. [Hang's][chun-hang] framework was built upon, clearing up naming convention confusion as well as introducing higher order balls. Then a mathematical framework was created to allow for conistent extraction of second ball scenarios. Finally player and team statisitics and heatmaps were shown to visualize the quantification of second balls. What has yet to be answered is how second ball wins influence the resulting second ball possesion. Stay tuned for the next post to find out more! Thanks for reading and please feel free to share or leave comments or feedback on Twitter. I am always looking for ways to improve my writing and research skills. Cheers.



[chun-hang]: https://lchunhang.medium.com/quantifying-second-ball-wins-d626ac56f108

[sunjic]: https://www.tandfonline.com/doi/full/10.1080/24748668.2025.2462399

[nyt]: https://www.nytimes.com/athletic/6193329/2025/03/13/measuring-second-balls-premier-league-analysis/