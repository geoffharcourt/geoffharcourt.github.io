Our league had our auction last week, and having deployed the new valuation
system and been fairly happy with the result, I think the mechanics of what I
did are worth sharing.

The key difference in what I built in 2017 and 2016 was the use of team-level
means and standard deviations to calculate z-scores rather than player-level
ones. I wanted to use team-level calculations for z-scores this year so that I
could more easily track team week-to-week variability and also so that I could
see a player's impact and more intuitively link it to the change in a team's
expected fortunes.

For real baseball the [formula for WAR] for a position-player is:

```
mlbWAR = (Batting Runs + Base Running Runs + Fielding Runs + Positional Adjustment + League Adjustment + Replacement Runs) / (Runs Per Win)
```

For our fantasy WAR the formula will be similar:

```
fantasyWAR = (z-scores + Positional Adjustment + League Adjustment + Replacement
z-score) / (z Per Win)
```

The formula's nearly the same, which makes comparing contributing parts of the
equations straightforward to understand.

The first task is figuring out player z-scores, which like batting runs is the
largest and most important contributor to player value. A [z-score] is how many
standard deviations a given measurement is above or below the mean for the
population. It's a statistical method for standardizing measurements that have
different scales and averages. Z-scores will allow us to do an apples-to-apples
comparison of players' contributions across all the scoring categories. An easy
way to think about this is that a single home run is more valuable than an RBI,
because the range of RBIs that players generate is much larger than the range of
home runs.

Many fantasy valuations systems use z-scores to calculate player value. The ones
that do typically determine the population of likely fantasy contributors (so
that the mean for each category is the fantasy league mean and not the major
league mean, which would be significantly lower depending on roster size and
team count) use those players to calculate the mean and standard deviation for
the population, then compare each player's contributions against the mean. For
leagues with weekly roster settings or season-long rotisserie scoring this
technique may be valid, but I found that it resulted in players who were
mediocre but played the whole season might record the same z-score sum as a
player who was very good but only played half of a season. These players aren't
actually equally valuable, because an owner could replace the player who played
less with someone from the bench and generate extra production. By only looking
at season-long production, we're losing out on the value of players who are
great but often hurt or missing the opportunity to understand the value of
part-time players who might play in a platoon.

My league has daily lineups, so the day is the most granular playing time
decision we can make. Very few doubleheaders are played in MLB, so we can cheat
and use production per game as a proxy for production per day. After downloading
projections from our preferred projections provider (I did separate versions in
2016 and 2017 using Fangraphs Depth Charts which are built from Steamer and ZiPS
and Baseball Prospectus' PECOTA), we want to create a table that looks like
this:

```
player_id year stat_system games obp ops r_per_g arbi_per_g hr_per_g asb_per_g
```

Where `arbi` is our league's RBI - GIDP stat and `asb` is our steals minus
caught stealing stat.

At this point, we could go ahead and figure out the z-score for each of the six
scoring categories, which is what I did to determine player valuations in 2016.
However, I decided that using the players alone to determine the standard
deviations wasn't correctly capturing the variability of weekly head-to-head
scoring (season-long results for players smoothed out volatile categories) and
was also resulting in incorrect valuation of players with extreme outlier
results in a single category. For someone like Mike Trout in OBP or Billy
Hamilton in steals, it was making their impacts on a team's weekly result look
overstated.

How can we calculate team-level means and standard deviations? If you have the
prior season's results that can be a start, but that doesn't work if you don't
have detailed history of the current or prior season or if in my case, the
number of teams in the league changed. In the next post, I'll walk through how
to build simulated team weekly scoring for any possible league configuration.

[formula for WAR]: http://www.fangraphs.com/library/war/war-position-players/ [z-score]:
https://en.wikipedia.org/wiki/Standard_score#Calculation_from_raw_score
