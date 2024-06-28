---
layout: post
title:  "Test Time with Playwright!"
date:   2024-06-27 00:09:37 +0000
categories: playwright testing
---

Playwright released the new Clock API in v1.45 which allows us to mock the datetime object on our websites while testing. Having spent the last 3 months testing a bookings application via the API, and just starting to add UI E2E tests, this couldn't have come at a better time! In this short post I'll dive into how it's used in Python, how we can do arithmatic on dates with `dateutil`, and my initial thoughts on how I'll use it in my tests. 

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

Working with dates in Python can be slightly awkward for common testing scenarios, such as "one week from now". There's a great library called `dateutil` that has a type named `relativedelta` just for this usecase. 

Say we're testing a 14 day sale banner that starts today and ends in a fortnight. We want to be able to specify the start and end dates dynamically. 

```
today = datetime.now()
two_weeks_from_today = today + relativedelta(days=14)
```

`relativedelta` can take the following arguments: 

```
relativedelta(arg1=x,arg2=y,arg3=z...)

year, month, day, hour, minute, second, microsecond:
    Absolute information (argument is singular); adding or subtracting a
    relativedelta with absolute information does not perform an arithmetic
    operation, but rather REPLACES the corresponding value in the
    original datetime with the value(s) in relativedelta.

years, months, weeks, days, hours, minutes, seconds, microseconds:
    Relative information, may be negative (argument is plural); adding
    or subtracting a relativedelta with relative information performs
    the corresponding arithmetic operation on the original datetime value
    with the information in the relativedelta.

weekday:
    One of the weekday instances (MO, TU, etc) available in the
    relativedelta module. These instances may receive a parameter N,
    specifying the Nth weekday, which could be positive or negative
    (like MO(+1) or MO(-2)). Not specifying it is the same as specifying
    +1. You can also use an integer, where 0=MO. This argument is always
    relative e.g. if the calculated date is already Monday, using MO(1)
    or MO(-1) won't change the day. To effectively make it absolute, use
    it in combination with the day argument (e.g. day=1, MO(1) for first
    Monday of the month).

leapdays:
    Will add given days to the date found, if year is a leap
    year, and the date found is post 28 of february.

yearday, nlyearday:
    Set the yearday or the non-leap year day (jump leap days).
    These are converted to day/month/leapdays information.
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