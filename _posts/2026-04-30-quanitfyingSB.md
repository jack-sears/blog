---
layout: post
title: "Quantifying Second Ball Wins"
date: 2026-04-30
categories: [notes]
---

The evaluation of the in-game actions of players to assess their impact on the outcome
of a game is an important sub-section of soccer analytics [2]. Unlike traditional
statistics that focus on goals and assists, evaluating other actions that players perform
delves deeper into the nuances of player behaviour and decision-making.

An action that often goes unnoticed is the ability to win second balls [3]. A second
ball is a possession gain following an intervention from an attempted long pass. Simply
put, a second ball occurs after the ball is kicked long and neither team clearly controls
it. Instead, the ball bounces, gets deflected, or is contested, and then the players rush
to try and take control of the ball. A quote from Pep Guardiola,
one of the most successful soccer managers, states the importance of winning second
balls, ”The main thing in English football is to control the second ball. Without that,
you cannot survive.” [4].

Despite the importance of winning second balls, there is a lack of research quantifying
and analyzing the value of winning second balls and how to do so successfully. Therefore this post aims to extend some of the research done and pose future direction for further work/extensions.

## Definition and Example

Let's begin by establishing a working definition for second balls. The following definitions have been adapted from a [blog][chun-hang] post by Chun Hang, but remain largely unchanged since I think it's a great definition.

SECOND BALL DEFINITION

And to further build intuition for those who are less familiar with second balls, I think seeing a real example does a great job.

SECOND BALL VID

It is clear from the example that second balls can lead to chaotic and quick attacking chances in favor of the winner. Before beginning to quantify them, let's investigate their importance a little bit more.


## Importance of Second Balls

The following brief analysis examines the relationship between second ball success, goal difference, and overall team performance. It should not be taken as fact but rather a thought provoking analysis to give reason why we should investigate second balls deeper.

### League Performance

Using data from the top 5 European leagues in 2015/2016, a linear regression is performed on goal difference (GD) vs total points earned in a season. 

GD VS POINT PICTURE

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

LEAGUE TABLE 2015/2016.

## Other Second Ball Research

To date, only one research [paper][sunjic] has directly analyzed second balls, while few online blogs/articles offer exploratory analyses ([here][chun-hang] and [here][nyt]).

[Sunjic et al.][sunjic] analyzed how second balls affect technical performance. 

## Second Ball Chains

## Second Ball Possessions

## Mathematical Definition of Second Balls

## Results

### Second Ball Rankings

### Second Ball Heatmaps

## Conclusions


[chun-hang]: https://lchunhang.medium.com/quantifying-second-ball-wins-d626ac56f108

[sunjic]: https://www.tandfonline.com/doi/full/10.1080/24748668.2025.2462399

[nyt]: https://www.nytimes.com/athletic/6193329/2025/03/13/measuring-second-balls-premier-league-analysis/