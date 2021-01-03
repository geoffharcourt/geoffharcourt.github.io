---
layout: post
title: Visual Verification at CommonLit
---

Testing is a big part of our engineering and product process at
[CommonLit](https://www.commonlit.org). We have a small team and a lot of
application to cover, so we need a good test suite to allow us to merge and
deploy with confidence.

About a year ago, we started to notice that some issues would fall through the
cracks of our test suite and human-driven QA. These issues fell into two
buckets: issues that were only an issue in production (so dependent on some
production-specific setting or content only present in our production database)
and issues that were very hard to verify through
[Capybara](https://teamcapybara.github.io/capybara/) tests such as the
display of information in a chart or the specific layout of text elements on the
page. We started looking into potential solutions to work through these problems
that didn't just mean spending more time in QA.

We've since implemented a solution that has been a big success. (We'll define
success here as preventing these kinds of problems before they reach deployment
in production.) The solution gets used in two phases of our pipeline (our CI
suite and our staging & production deployment verifications) and leverages
[Percy](https://percy.io).

Percy is a visual testing tool that runs best as part of a browser-based
end-to-end test. It uses
[Puppeteer](https://developers.google.com/web/tools/puppeteer) mated with some
proprietary technology that's part of Percy's stack to take snapshots of what
your browser sees and then diffs those snapshots against a "baseline", which is
normally the same snapshot last taken against whatever your main branch is.
Where Percy really shines is in the ability to avoid false positives, which can
be a big problem with visual testing tools. Percy provides several features that
help avoid false positives on snapshot changes. Diffs of only 1 are considered
the irrelevant. Specific CSS can be ignored, which helps avoid things that you
know will change from page to page (such as the current date or a
randomly-generated user's specific username). Animations are frozen in a way
that avoids jitter from when the snapshot it taken.

We use Percy in our CI suite, taking desktop-width and in some cases
mobile-width snapshots of 70 scenarios. These snapshots help catch issues like
unintended CSS changes, problems with React plugins when we bump versions, and
places where we're scrubbing HTML content for formatting incorrectly. As a
result of these snapshots, we're able to make CSS and JavaScript changes with
more confidence and with less need to have our QA team re-test pages for visual
concerns on unrelated pages. When snapshots change for valid reasons, our team
gets a notification and we review someone on the Engineering or Product teams
approves the snapshot, making that snapshot the new baseline when its branch
gets merged. Taking snapshots of mobile pages has been particularly useful,
alerting us to possible CSS issues that engineers might not experience in
development or their own spot-testing of proposed changes.

This kind of visual testing isn't without drawbacks. For tests with lots of data
setup that take snapshots, we often need to set what would usually be random or
sequential data with specific names to prevent snapshots from having differences
solely due to the test's order in RSpec's random seeding. We've also had to take
some aggressive steps to avoid snapshot jitter with some third-party charting
code that renders results in a way that might be non-deterministic based on
widget loading order. Now that we've solved these issues in our codebase and
test suite, we're able to get high-signal feedback about the safety and impact
of a pull request before it gets merged or goes for human QA.

Earlier this year, we significantly tightened our content security
policy (CSP) to make the browser experience on CommonLit safer for our users.
(It also had the convenient effect of blocking several misbehaving browser
extensions and pieces of malware that occasionally created difficult-to-pinpoint
issues for students and teachers that stumped our team until the CSP
tightening.) The changes were a nice win, but we had some rollout hiccups where
some content on pages was blocked by the new CSP rules and we didn't see it in
development or testing.

We were at a loss as to how to improve our rollout of changes like this in the
future. In some cases the issues were related to third-party scripts that we
didn't want to load in test due to save time in our tests. Adding these scripts
in test would slow down all of our tests and likely not provide much benefit, as
the effects of the failures were subtle and not likely to be directly tested in
Capybara (our Ruby mechanism for running browser-based tests). In other cases,
the specific failures were related to assets that were controlled by content in
our database, so attempting to include them in test was a losing proposition, as
we'd need to remember to verify every new change and it would mean that our
Curriculum team would now have to verify every externally-visible content change
with Engineering before publication.

We managed to improve our results against both of these situations by adding
Percy to our post-deployment canary tests. Every time we deploy to our primary
staging environment and our production environment, we run a series of
end-to-end tests with Cypress to verify that key public pages, our library,
logins, sign-ups, school and district data dashboards, and teacher onboarding
are all in good working order. By adding Percy to these tests, we're now able to
verify that pages all look exactly the way we'd expect. The snapshots have
helped us confirm that pie charts and complex data visualizations look exactly
the same from deploy to deploy and helped us catch problems that didn't appear
in CI before we've promoted the relevant code to production.

Visual verification has become a critical tool in our development workflow at
CommonLit. It's improved our shipping speed and allowed us to make more
ambitious changes with confidence.
