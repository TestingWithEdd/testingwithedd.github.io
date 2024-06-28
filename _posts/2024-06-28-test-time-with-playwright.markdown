---
layout: post
title:  "Test Time with Playwright!"
date:   2024-06-28
categories: testing playwright
---

_Note this code is accurate as per the documentation and the JS implementation, but Python is still pending 1.45 to be tagged. I'll update if anything changes post-release._


Playwright released the new Clock API in v1.45 which allows us to mock the datetime object on our websites while testing. Having spent the last 3 months testing a bookings application via the API, and just starting to add UI E2E tests, this couldn't have come at a better time! 

In this short post I'll dive into how it's used in Python, how we can do arithmetic on dates with `dateutil`, and my initial thoughts on how I'll use it in my tests. 

First we need to initialize the clock object with a time. Either `datetime.now()` to get the current time, or a date object for some time in the future. 

```
from datetime import datetime

...
# datetime.datetime(year, month, day, hour, minute, second)
page.clock.install(time=datetime.datetime(2024, 6, 28, 9, 0, 0))
```

The clock will now start from that time. Say we're testing a 5 minute sale offer; users open the site with a pop up saying "there's a sale" and after 5 minutes the pop up will show a string saying the "offer has expired".

```
from datetime import datetime

...
page.clock.install(time=datetime.now())

expect(page.get_by_text("there's a sale")).to_be_visible()

# fast forward 5 minutes and 1 second
page.clock.fast_forward("05:01")

expect(page.get_by_text("offer has expired")).to_be_visible()
```

When we're testing times we may want to be specific that something happens before, on the second, and immediately after. This is a little hard if time is rolling but the Clock API does let us pause at times too. 

```
page.clock.pause_at(datetime.datetime(2024, 12, 31, 23, 59, 59))
# do something

page.clock.pause_at(datetime.datetime(2025, 01, 01, 00, 00, 00))
# do something else

page.clock.pause_at(datetime.datetime(2025, 01, 01, 00, 00, 01))
# do something else again
```

If we want to start time from that point it's a simple `clock.resume()`.

### Date Arithmetic 

Working with dates in Python can be slightly awkward for common testing scenarios, such as "one week from now". There's a great library called `dateutil` that has a type named `relativedelta` just for this use case. 

Say we're testing a 14 day sale banner that starts today and ends in a fortnight. We want to be able to specify the start and end dates dynamically. 

```
today = datetime.now()
two_weeks_from_today = today + relativedelta(days=14)
```

`relativedelta` can take the following arguments: 

```
relativedelta(arg1=x,arg2=y,arg3=z...)

years, months, weeks, days, hours, minutes, seconds
```

positive or negative deltas

```
datetime.now() + relativedelta(months=+1)
datetime.now() + relativedelta(months=-1)
```

and how long between two dates

```
relativedelta(datetime.now(), date(2000, 01, 01))
```

If you try stick to datetime objects in the standard library you'll probably hit a wall, so try `dateutil` out.

Good luck testing time! :)