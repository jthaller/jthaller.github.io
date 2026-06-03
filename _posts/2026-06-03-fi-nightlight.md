---
title: "Fi Nightlight: Automating My Dog's Collar LED with GCP"
date: '2026-06-03'
categories:
  - miscellaneous
tags: 
- gcp
- iot
- python
- infrastructure
classes: 
- wide
- text-justify
---

My dog Baguette wears a [Fi collar](https://tryfi.com/) — a GPS smart collar that tracks her location and defines safe zones around home. The collar has an LED light you can toggle from the app, which is great for visibility on night walks. But there's no way to automate it. If Baguette slips out the door after dark, I'd have to manually notice and pull up the app to turn the light on. That's exactly the kind of thing a computer should be doing for me.

So I built [fi-nightlight](https://github.com/jthaller/fi-nightlight) — a serverless automation that turns on the collar LED when Baguette leaves the safe zone after sunset, and turns it off when she comes back. It runs on Google Cloud's free tier and costs $0/month.

## Why this feature should exist

Fi already knows where your dog is. It already knows where your safe zones are. It already controls the LED. It even sends you push notifications when your dog leaves a zone. But it can't connect these dots: "it's dark and my dog is outside — turn the light on." That's the gap this project fills.

The LED is a safety feature. A dog running around at night without a visible light is harder for drivers, cyclists, and other people to see. Automating this means it works even when I'm not paying attention to my phone.

## The infrastructure

The whole system is three GCP services:

```
Cloud Scheduler (cron: */5 * * * *)
        │
        │  HTTPS GET every 5 min, 24/7
        │  (authenticated via OIDC service account)
        │
        ▼
Cloud Function (fi-nightlight)
        │
        ├── Reads state from Firestore
        ├── Calculates sunset (astral library, pure math)
        ├── Before sunset or after 1am → exits early
        ├── In active window → polls Fi API for zone status
        ├── Toggles LED based on zone
        └── Writes state back to Firestore
```

**Cloud Scheduler** fires an HTTPS request at the function every 5 minutes. It doesn't know anything about dogs or sunsets — it just pokes the function on a timer.

**Cloud Function** is a single Python function that spins up, runs for 2-4 seconds, and shuts down. It's stateless, which is why we need the third piece.

**Firestore** holds one document: `{"led_on": false, "last_checked": "..."}`. This is how the function knows whether it already sent an on/off command. Without it, every invocation would start from scratch and might redundantly hit the Fi API.

## Design decisions

### Why not poll only during the active window?

The original design had two Cloud Functions — one to calculate sunset and enable a scheduler job, and another to do the actual checks. The sunset calculator would pause and resume the zone checker dynamically.

This was over-engineered. A single function that runs every 5 minutes and exits early during the day is simpler, cheaper to maintain, and has the same API cost (the early exits are essentially free — they don't touch the Fi API at all). At 288 invocations/day, we're nowhere near any free tier limit.

### Why not webhooks?

Fi doesn't expose webhooks or push notifications through their API. The [pytryfi](https://pypi.org/project/pytryfi/) library is a GraphQL client that pulls data on demand. Polling is the only option.

### How does sunset calculation work?

No external API calls. The [astral](https://pypi.org/project/astral/) library computes sunset and sunrise from your latitude/longitude using astronomical formulas. It's deterministic, works offline, and handles DST transitions correctly. The function recalculates it on every invocation from the current date — no need to store it.

### How does zone detection work?

This was the trickiest part. The Fi API exposes several location-related fields on a pet object, and it wasn't obvious which one to use.

- `areaName` — was always `None` in my testing
- `currPlaceName` — reliably returns `"Home"` when the dog is in the safe zone

So the detection is simple: if `currPlaceName == "Home"`, the dog is inside. Anything else means outside. This works even with the base station offline — Fi determines zone status server-side from GPS, not from the base's Wi-Fi.

### Battery considerations

This came up early in the design process. Three things to know:

1. **Polling the Fi API does not drain the collar battery.** The API reads from Fi's cloud servers. The collar reports its GPS position on its own schedule regardless of how often you poll.

2. **The LED uses minimal extra battery.** It's designed to stay on during walks. The impact is small and only applies during the active window (sunset to 1am at most).

3. **Lost Dog Mode is never used.** That's the feature that significantly drains battery by increasing GPS reporting frequency. This project doesn't touch it.

### The cutoff problem

The initial implementation had a bug in the cutoff logic:

```python
# Broken: can never be >= 22 AND < 12 at the same time
if now.hour >= config.cutoff_hour and now.hour < 12:
```

The fix handles day rollover properly:

```python
def is_active_window(sunset, cutoff_hour, now):
    cutoff = now.replace(hour=cutoff_hour, minute=0, second=0)
    if cutoff_hour < 12 and now.hour >= 12:
        cutoff += timedelta(days=1)
    return sunset <= now < cutoff
```

A cutoff of 1 means 1am the next day. A cutoff of 22 means 10pm the same day. This is a pure function with no external dependencies, which makes it easy to test exhaustively — there are 10 unit tests covering every edge case.

### Why keep checking after the dog returns?

Another early design flaw was disabling the scheduler when the dog came back inside the zone. But what if the dog goes out, comes back, and goes out again? The automation would be dead after the first return.

The fix: keep checking every 5 minutes for the entire active window (sunset to 1am), regardless of what the dog does. The LED toggles reactively on every check. At 1am, a final check ensures the LED is off.

## The cost

| Resource | Usage/day | Free tier | Cost |
|----------|-----------|-----------|------|
| Cloud Scheduler | 288 invocations | 3 jobs free | $0 |
| Cloud Function | 288 runs × ~3s | 2M invocations/mo | $0 |
| Firestore | 288 reads + writes | 50K reads + 20K writes/day | $0 |

Firestore stores a single 60-byte document that gets overwritten (not appended) on every check. It will never grow. No cleanup needed.

## Running it yourself

The project supports three deployment modes:

- **Local**: `python run.py` with APScheduler (good for Raspberry Pi)
- **Docker**: `docker compose up` with persistent volume for state
- **GCP**: Single deploy script sets up everything

The [repo](https://github.com/jthaller/fi-nightlight) has a test script that lets you verify your Fi connection, check zone detection, and toggle the LED without deploying anything:

```bash
python test_connection.py              # verify credentials + zone status
python test_connection.py --led-on     # turn LED on
python test_connection.py --led-off    # turn LED off
python test_connection.py --simulate   # 3-minute check cycle
```

If you have a Fi collar and want your dog to be automatically visible at night, check it out.
