---
layout: post
title: Stubbing Time in Capybara Specs
---

I hate flaky tests. We work really hard to knock them out on our team. My
thinking on this (beyond just being annoyed) was driven by this
[excellent Sam Saffron blog post](https://samsaffron.com/archive/2019/05/15/tests-that-sometimes-fail)
about the pernicious and compounding costs of intermittent test failures over
time. One thing we do to test robustly is to aggressively use the
[Timecop](https://github.com/travisjeffery/timecop) gem. Rails added some
helpers to do time travel, but Timecop's methods continue to have more
flexibility around freezing, travelling to another time and restarting the
clock, etc.

Some of the most annoying spec failures here are "time-of-day" test failures.
These are tests that fail during certain windows of the day, usually when the
calendar day is different in UTC (or wherever your CI machine is located) and
your home time zone. We often use Timecop to freeze unit test time in one of
these seams to intentionally check behavior at the edges.

Timecop can also be useful for avoiding rounding precision issues when you
compare "now" to the saved time in the database, which might be less precise in
some operating systems. For specs that test an exact timestamp that comes from
tracking the exact time of an event, we often freeze time at a "round" interval
with no milliseconds:

```ruby
it "marks the current time" do
  Timecop.freeze(Time.utc(2020)) do
    # example here
  end
end
```

We have a few specs in our system test suite where we have sensitive logic based
on the current date as determined by the browser. We noticed that these specs
would fail in CI runs where UTC time was on a new calendar day and our US time
zones were still within the prior calendar day. Timecop is only able to affect
the time in Ruby-land on the server, so using Timecop in a system spec will lead
to odd issues in system tests where the browser and the server have diverging
current times.

For tests that absolutely must mock the time in JavaScript and Ruby, here's what
we do. We have a partial that only gets loaded in system tests:

```erb
<% if Timecop.top_stack_item %>
  <%= javascript_include_tag "/fakeTimers.js" %>

  <script type="text/javascript">
    // [NOTE] Faking time *must* be in the calendar future
    // or authenticated AJAX requests will fail
    FakeTimers.install({
      now: <%= (Time.now.to_i * 1000).to_json %>,
      shouldAdvanceTime: true,
    });
  </script>
<% end %>
```

Here's what this does:

- `Timecop.top_stack_item` indicates that Timecop is currently in use and
  manipulating the system time.
- `fakeTimers.js` loads the [`@sinon/fake-timers`](https://github.com/sinonjs/fake-timers)
  library. We then pass it the mocked current time from Ruby, which is
  in turn controlled by Timecop. This allows us to create a scenario where
  server and browser time are mocked and in sync with one another.

We use this very sparingly and only load it if the spec requires this
functionality. We've noticed that this doesn't work very well unless the fake
time is in the future, probably because Rails has some mechanism to refuse
requests that are too far in the past. While this is a bit hacky, it does allow
us to do full system tests in scenarios where we have to set the exact date such
as scheduling scenarios that use a datepicker, data visualizations that we test
with a visual snapshot, etc.



