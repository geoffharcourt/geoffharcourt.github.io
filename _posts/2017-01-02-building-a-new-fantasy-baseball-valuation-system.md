Welcome to 2017! This is a new effort on my part to blog about my fantasy baseball research and tooling.

In a series of posts I'm going to walk through the process of how I built a
system for valuing the statistical contributions of fantasy baseball players
that can be customized for a wide variety of league parameters. I undertook this
project last year (and then reimplemented it after learning from using it over
the course of the season) after coming to the conclusion that my league was too
weird to solely rely on "off-the-shelf" fantasy advice and my own instincts were
too unreliable to just go with my gut.

First, here's a very brief description of my fantasy league's rules:

- We have 16 teams (expanded from 14 in 2016). Each team gets the MLB luxury tax
  threshold worth of cash to spend (~$200m) on keepers and free agents.
- Players picked up mid-season have to be paid the minimum salary, prorated for
  the portion of the season remaining. In short, you can't stream pitchers.
- Up to five players can be kept every year at their existing salary. Players
  can only be kept for three years.
- Players who have not exhausted their rookie eligibility are drafted in an "AA"
  draft. These players, once they are promoted are underpaid (just like in real
  life!), and can be retained for four years before they go back into the free
  agent pool. They do not count against your team's keeper total.
- Scoring is head-to-head over 20 weeks followed by an NFL-style playoff format
  (wildcards, first-round byes for league winners).
- The scoring categories for offense are OBP, OPS, R, HR, RBI - GIDP (aRBI),
  SB - 0.5 CS (aSB).
- The scoring caetegories for pitching are ERA, WHIP (but we include HBP, so
  aWHIP), K, HR allowed (HRa), a composite relief stat which is basically SV +
  HLD - a big penalty for blown saves or relief losses with some small credit
  for innings pitched (VIJAY), and 2 QS + W - L - 2 non-QS starts (nQW). Teams
  must throw a minimum of 44 innings each week or they forfeit four pitching
  categories and incur some budget/draft order penalties.
- Teams roster 30 non-"AA" players, plus 3 DL slots, and daily lineups have the
  following active spots: C, 1B, 2B, SS, 3B, OF, CF, RF, with two Util players.
  5 SPs and 6 RPs can be active on any given day.

Our scoring and roster rules (particularly for pitching) are just unusual enough
that I felt like most public fantasy roster construction advice and publicly
available rankings (including the sort from Baseball Prospectus' [PFM auction
calculator](http://baseballprospectus.com/pfm/) and FanGraphs' [auction
calculator](http://www.fangraphs.com/auctiontool.aspx)) were only useful as
guideposts rather than something authoritative.

After doing some research and re-reading Dave Cameron's [extensive
series](http://www.fangraphs.com/library/war/war-position-players/) on how Wins
Above Replacement (WAR) is calculated, I wanted to set out to build my own
version of fantasy wins above replacement. I've seen several versions of this
built before, but I specifically wanted to accomodate the following
requirements:

- This is not a prediction system, it's a valuation system. I'll be using
  someone else's projections, either FanGraphs' [Depth
  Charts](http://fangraphs.com) (based on [Steamer
  Projections](http://steamerprojections.com/blog/)) or Baseball Prospectus'
  [PECOTA](http://www.baseballprospectus.com/) are my likely candidates for
  player MLB production data.
- Category scores should be equally weighted. In an environment where home runs
  are way up (but also more evenly distributed) and stolen bases are down but
  increasingly concentrated among fewer players, I want to better understand how
  those categories contribute to player value. I suspect that many fantasy
  players are ignoring the downside of sluggers who never run, or assuming they
  can cheaply snag a couple burners to fill in that category at the back-end of
  their rosters. In 2016, a single steal was much more valuable than a home run,
  but how much more valuable?
- Player value must be calculated on a per-game basis rather than a season-long
  basis. If a player was very good in short service due to injury or a delayed
  promotion, that player while he was on the field was better than some mediocre
  player who accumulated over a full season. This aspect of my WAR will be
  useful for building "platoons" of two left-handed hitters who play the same
  position that I will only start when they have the platoon advantage (against
  RHP).
- We should be able to adjust for hypothetical changes in the league's rules
  (like this year's expansion to 16 teams, or scoring category changes). The
  baselines for average and replacement levels must be calculable from the
  universe of MLB stats, and not reliant on results from the league's actual
  performance.
- The true value of a position's adjustment value should be quantifiable. After
  reading Corinne Landrey's [excellent article in the 2017 Hardball Times
  Annual](http://www.hardballtimes.com/bookstore/) my suspicion that non-catcher
  positions were converging was confirmed. However, since our league only has 16
  teams, I wanted to understand how much that effect would apply to our player
  universe and position eligibility rules. Having custom-calculated position
  adjustments will provide knowledge in auction about a player if I will end up
  playing him outside of his most-scarce roster spot.
- The system must be able to respond to pricing and scarcity mid-auction. In
  prior years I built auction valuations with Excel and toward the end of the
  auction when the amount of money and talent was low some calculations returned
  completely unrealistic values. I should be able to field-test the system
  before our auction in March with confidence.
- Aggregating player values should give a reasonable representation of team
  strength. I can test this by running the valuation system against 2015 and
  2016 rosters to see if the best teams scored well.
- The system must be as reliable as possible. If I get new information, whether
  that's in the form of updated projections or new player ownership mid-auction,
  the current situation should always be accurately reflected.

In the next post I'm going to assemble my player data and projections in a
database so I can begin assigning values to those statistical contributions from
the players.
