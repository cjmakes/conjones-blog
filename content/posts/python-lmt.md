---
title: "Why there are 67 minutes between 8am and 9am"
date: 2024-12-26T14:04:52-08:00
draft: false
---

![xkcd-datetime](https://imgs.xkcd.com/comics/datetime.png)

I knew timezones were hard but I learned a new tripping hazard and I learned it
the fun way: in production. In this post I'll explain why there are 67 minutes
between 8am and 9am in the code below:


```python
>>> import datetime
>>> import pytz
>>> t1 = datetime.datetime(2024,12,25,8,tzinfo=pytz.timezone("US/Pacific"))
>>> t2 = datetime.datetime.fromtimestamp(1735146000, tz=pytz.timezone("US/Pacific"))
>>> t2 - t1
datetime.timedelta(seconds=4020)
>>> 4020 / 60
67.0
>>> t1 >= t2-datetime.timedelta(minutes=60)
False
>>> t1+datetime.timedelta(minutes=7) >= t2-datetime.timedelta(minutes=60)
True
```

My production scenario was fetching a list of events from an external service,
each of which has a start timestamp returned as an epoch. My logic was if t1
is less than 1 hour before event start time, do some action. Imagine my
surprise when the action didn't trigger until 7 minutes after I expected it to.

Looking closer at t1, we see its timezone is not Pacific Standard Time (PST) but Local Mean Time(LMT).


```python
>>> t1
datetime.datetime(2024, 12, 25, 8, 0, tzinfo=<DstTzInfo 'US/Pacific' LMT-1 day, 16:07:00 STD>)
>>> t2
datetime.datetime(2024, 12, 25, 9, 0, tzinfo=<DstTzInfo 'US/Pacific' PST-1 day, 16:00:00 STD>)
>>> 
```

[LMT](https://en.wikipedia.org/wiki/Local_mean_time) is a way of expressing
local time in a way that follows to the sun and has some neat history. Of
interest to us here, is that LMT predates UTC and adds around 4 minutes per
longitude from the equator.

So why does pytz set the timezone as LMT for one constructor and UTC for the
other? Because if I had read the pytz docs, I would have seen that this is not
the right way to build datetime objects:

> This library only supports two ways of building a localized time. The first is to use the localize() method provided by the pytz library. This is used to localize a naive datetime (datetime with no timezone information):
> 
> The second way of building a localized time is by converting an existing localized time using the standard astimezone() method:

From: [https://pypi.org/project/pytz/](https://pypi.org/project/pytz/)


So this is mostly on me for not reading the docs, but I found this to be
incredibly not intuitive. So here is a corrected version of the code:

```python
>>> t1_new = pytz.timezone("US/Pacific").localize(datetime.datetime(2024,12,25,8))
>>> t2_new = pytz.timezone("US/Pacific").localize(datetime.datetime.fromtimestamp(1735146000))
>>> t2_new - t1_new
datetime.timedelta(seconds=3600)
>>> 3600/60
60.0
>>> t1_new >= t2_new - datetime.timedelta(minutes=60)
True
```

Moral of the story, read the docs, and avoid working with time if possible. For
a more detailed answer of why pytz chooses LMT see
[this](https://groups.google.com/g/django-users/c/rXalwEztfr0/m/QAd5bIJubwAJ)
google group answer. Pretty interesting stuff, its fun to learn in production.
