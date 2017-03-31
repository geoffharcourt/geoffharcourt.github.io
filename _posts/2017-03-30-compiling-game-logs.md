Having determined the basic formula for offensive player fantasy WAR, I now
needed a way to determine weekly variation by teams. One of my league's members
has been compiling and publishing team weekly means and standard deviations for
each scoring category, which was useful. However, since our league is changing
teams, I didn't have a way to reliably adjust that for our 16-team league in
2017.

I decided that in the interest of being able to see the effect that team count
might have on weekly stat variation to design a system to simulate a league of
any size. In order to do that, I first needed daily statistics for every
viable fantasy player.

MLB has a JSON API that can return daily game logs for a given player. You can
request it from this endpoint:

```
http://m.mlb.com/lookup/json/named.sport_hitting_game_log_composed.bam
?game_type=%27R%27&league_list_id=%27mlb%27&player_id=#{player.id}&season=#{year}"
```

Where `&player_id=#{player_id}` should be the player's ID (for Mike Trout that
would be `&player_id=545361`), `#{year}` would be the year. You can even request
non-regular season games by changing the `game_type=` parameter, or request
minor league stats using the `league_list_id=` parameter.

(The string interpolations here are Ruby-style, so you omit the `#{}`
surrounding the variable.)

With a list of player IDs, you can use this endpoint to build your own database
of daily hitting statistics.

In the next post, we'll simulate a draft to create the correct number of teams
for each hypothetical league size we might want to analyze.
