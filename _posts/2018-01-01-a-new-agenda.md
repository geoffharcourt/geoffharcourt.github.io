---
layout: post
title: '2018 - A New Agenda'
---

Time to start over again. My team-building approach last year (building a
fantasy baseball WAR that was adaptable to conditions where I didn't have
empirical weekly results) was successful in that my team made the playoffs
despite some terrible luck from injuries.

My blogging approach was a complete failure. I let worrying about having my
ideas be "done" block me from putting my thoughts out there.

I'm going to endeavor to write more often and more consistently this year, even
if it's just to keep notes about my avenues of research and software learning.
Here's what I'm hoping to accomplish this winter before Opening Day:

- Generate "synthetic weeks" (simulate a player's performance many times over a
  weekly span using a set of annual projections) so that I can calculate the
  weekly team H2H mean and standard deviation for any plausible league
  configuration. I'll be using Python tools to do this, having had success with
  [SciPy](https://www.scipy.org/) and some of the distributions available in
  [PyMC3](http://docs.pymc.io/notebooks/getting_started.html). Ruby's existing tools
  just aren't fleshed out enough to do the things I want to do with statistical
  distributions.
- Improve my data crosswalk so that it's easier to link data from different
  sources.
- Start making headway on risk from injury or a collapse in performance into my
  player pricing model. I thought this
  [Saber Seminar talk](https://www.fangraphs.com/tht/applying-asset-pricing-theory-to-mlb/)
  might be an interesting place to start.
