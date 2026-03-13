# Sleep Coach App — Claude Code Update Instructions

## Context

You are editing a single `index.html` file in a GitHub repo called `sleep-coach` (owned by `glip303`). The file is a standalone HTML/CSS/JS app served via GitHub Pages at `https://lipington.com/sleep-coach/`. It connects to a BabyBuddy API to track an infant's sleep and feeding, and provides real-time guidance based on a sleep plan.

The current version has a dark theme and lacks several features. You need to make three changes described below. Do NOT add any external dependencies beyond Google Fonts. Everything stays in one HTML file.

---

## Change 1: Light, Friendly Theme

Replace the entire dark colour scheme with a warm, light design inspired by Jude's Ice Cream (cream backgrounds, pastel accents, vibrant colour pops) and Anthropic's website (clean, spacious, modern).

### Colour palette

```
--cream: #FDF6EE           /* page background */
--cream-dark: #F5EBE0       /* secondary background, input fields */
--warm-white: #FFFCF7       /* card backgrounds */
--peach: #FDDCBD            /* light accent */
--peach-deep: #F4A261       /* warning state, amber alerts */
--sky: #B8D8E8              /* light accent */
--sky-deep: #5B9BBF          /* links, info state */
--mint: #B8E0D2             /* light accent */
--mint-deep: #4CA882         /* positive/green state */
--lavender: #D4C5F0         /* sleep-related light */
--lavender-deep: #8B6FC0    /* sleep-related strong */
--rose: #F2C4CE             /* light urgent */
--rose-deep: #D4738A         /* urgent text */
--coral: #F28B82            /* critical/red state */
--text: #2D2926             /* primary text */
--text-mid: #6B5E57         /* secondary text */
--text-light: #A69A93       /* labels, hints */
--text-faint: #CFC5BC       /* placeholder text, timestamps */
--border: #EDE4DA           /* card borders */
```

### Typography
- Primary font: `'Nunito', sans-serif` (import weights 400, 500, 600, 700, 800)
- Monospace: `'JetBrains Mono', monospace` (import weights 400, 500)
- Import from Google Fonts

### Design rules
- All cards: `background: var(--warm-white); border: 2px solid var(--border); border-radius: 20px;`
- No box shadows except on the setup form
- Setup screen background: `linear-gradient(160deg, var(--cream) 0%, var(--peach) 50%, var(--sky) 100%)`
- Buttons: solid fills, no outlines, `border-radius: 14px`, `font-weight: 700`
- The wake window hero card has a coloured top bar (4px): mint-deep for green, peach-deep for amber, coral for red
- Guidance cards use `border-left: 4px solid [colour]` with a very subtle tinted background (e.g. `#F0FFF4` for positive, `#FFF8F0` for warning, `#FFF0EF` for urgent)
- Mobile-first: `max-width: 500px; margin: 0 auto;`
- Add `<meta name="theme-color" content="#FDF6EE">` and PWA meta tags for home screen install

### Temperament buttons
When selected, each gets a unique tint:
- Calm: `border-color: var(--mint-deep); background: #E8F5E9;`
- Drowsy: `border-color: var(--lavender-deep); background: #EDE7F6;`
- Fighting sleep: `border-color: var(--peach-deep); background: #FFF3E0;`
- Inconsolable: `border-color: var(--rose-deep); background: #FCE4EC;`

---

## Change 2: Sleep Timer Toggle

Add a card near the top of the dashboard (below the header, above the wake window) that allows the user to indicate the baby is currently asleep. This is needed because BabyBuddy only shows completed sleep records — if a sleep timer is running in BabyBuddy, the app doesn't know about it.

### UI
- When AWAKE: show a card with a ☀️ icon, text "He's awake", subtext "Tap when he falls asleep", and a button labelled "🌙 Asleep" (use `var(--lavender-deep)` background)
- When ASLEEP: replace the wake window hero with a sleep timer card:
  - Lavender gradient background: `linear-gradient(135deg, #EDE7F6, #F3E5F5)`
  - 🌙 moon icon
  - Large ticking timer showing elapsed nap time (JetBrains Mono, 36px, `var(--lavender-deep)`)
  - Text: "Nap started at HH:MM" or "Night sleep started at HH:MM" (based on time of day — nap if 6am-7pm, night otherwise)
  - An "ideal wake time" calculation (see below)
  - A button "☀️ He's awake now" (coral background) to end the manual sleep state

### Sleep state logic
- Store `manualSleep = { start: Date }` in app state when the user taps "Asleep"
- When `manualSleep` is active:
  - Wake window shows 0 (baby is sleeping)
  - Guidance changes to say "He's sleeping. Don't disturb." plus the ideal wake info
  - The temperament selector is hidden (not relevant while asleep)
- When the user taps "He's awake now", clear `manualSleep` and resume normal dashboard
- Also auto-clear `manualSleep` if the API returns a new completed sleep record that started after the manual sleep start time (meaning BabyBuddy has caught up)

### Ideal wake time calculation
When asleep, calculate and display when to ideally wake him to stay on the plan:

```javascript
function calcIdealWake(sleepStart, now) {
  const hr = now.getHours();
  let targetDuration; // in minutes

  if (hr >= 14 && hr < 17) {
    // Critical afternoon nap: let him sleep longer
    targetDuration = 60;
  } else if (hr < 12) {
    // Morning nap
    targetDuration = 50;
  } else if (hr >= 17) {
    // Late catnap: keep it short
    targetDuration = 30;
  } else {
    // Midday
    targetDuration = 45;
  }

  // Evening constraint: nap must end by 18:30 so bedtime routine
  // can start within 90 mins (targeting bedtime before 20:00)
  if (hr >= 15) {
    const latestEnd = new Date(now);
    latestEnd.setHours(18, 30, 0, 0);
    const maxRemaining = Math.max(0, (latestEnd - now) / 60000);
    const elapsed = (now - sleepStart) / 60000;
    targetDuration = Math.min(targetDuration, elapsed + maxRemaining);
  }

  const idealWake = new Date(sleepStart.getTime() + targetDuration * 60000);
  return idealWake > now ? idealWake : null; // null if already past
}
```

Display as: "Ideal wake: HH:MM — X mins from now" in a tinted box within the sleep card.

---

## Change 3: Forecast Schedule

Add a new section below the guidance cards (above the daily stats) that shows a **forward-looking schedule** for the rest of the day. This tells the parents what times to aim for.

### Logic

```javascript
function buildForecast(cs) {
  // cs = current state from analyze()
  const events = [];
  const now = cs.now;
  let cursor; // tracks the "current time" as we project forward

  if (cs.sleeping && cs.sleepStart) {
    // Baby is asleep — forecast from ideal wake time
    const idealWake = calcIdealWake(cs.sleepStart, now);
    if (idealWake) {
      events.push({ time: idealWake, label: 'Wake from nap', type: 'wake' });
      cursor = idealWake;
    } else {
      cursor = now;
    }
  } else {
    // Baby is awake — forecast from now
    cursor = now;
  }

  // Determine which wake window applies
  const getWW = (d) => {
    const h = d.getHours();
    // Count naps so far + projected naps
    if (cs.napCt === 0 && h < 12) return 70; // first nap
    if (h >= 16) return 90; // evening
    return 85; // daytime
  };

  // Project forward: alternate between nap and wake periods
  // until we hit bedtime (target 19:00-20:00)
  let napIndex = cs.napCt;
  const maxEvents = 8;
  let isAwake = !cs.sleeping; // current state

  if (cs.sleeping) {
    // After projected wake, next is a wake window
    isAwake = true;
  }

  for (let i = events.length; i < maxEvents; i++) {
    if (cursor.getHours() >= 20) break; // past bedtime, stop

    if (isAwake) {
      // Project next nap start
      const ww = getWW(cursor);
      const napStart = new Date(cursor.getTime() + ww * 60000);
      if (napStart.getHours() >= 19) {
        // This would be bedtime, not a nap
        events.push({ time: napStart, label: 'Bedtime routine', type: 'bed' });
        break;
      }
      events.push({ time: napStart, label: `Nap ${napIndex + 1} start`, type: 'sleep' });

      // Project nap duration
      const napH = napStart.getHours();
      let napDur = 45; // default
      if (napH >= 14 && napH < 17) napDur = 60; // afternoon
      else if (napH < 12) napDur = 50; // morning
      else if (napH >= 17) napDur = 25; // catnap

      // Evening constraint
      if (napH >= 15) {
        const latestEnd = new Date(napStart);
        latestEnd.setHours(18, 30, 0, 0);
        napDur = Math.min(napDur, (latestEnd - napStart) / 60000);
      }

      if (napDur > 5) {
        const napEnd = new Date(napStart.getTime() + napDur * 60000);
        events.push({ time: napEnd, label: `Nap ${napIndex + 1} end`, type: 'wake' });
        cursor = napEnd;
        napIndex++;
      } else {
        cursor = napStart;
      }
      isAwake = false;
    } else {
      // Just woke — this starts a wake window
      isAwake = true;
    }
  }

  // Add bedtime if we haven't already
  if (!events.some(e => e.type === 'bed') && cursor.getHours() < 20) {
    const ww = 90;
    const bedtime = new Date(cursor.getTime() + ww * 60000);
    if (bedtime.getHours() >= 17 && bedtime.getHours() <= 21) {
      events.push({ time: bedtime, label: 'Bedtime routine', type: 'bed' });
    }
  }

  // Add formula reminder if not yet given
  if (cs.formulaToday.length === 0) {
    const bedEvent = events.find(e => e.type === 'bed');
    if (bedEvent) {
      events.push({
        time: new Date(bedEvent.time.getTime() - 5 * 60000),
        label: 'Formula top-up (150-180ml)',
        type: 'feed'
      });
    }
  }

  // Sort by time and filter to future only
  return events
    .filter(e => e.time > now)
    .sort((a, b) => a.time - b.time);
}
```

### UI

Render the forecast as a vertical timeline with coloured dots:
- `sleep` events: lavender dot
- `wake` events: amber/peach dot  
- `bed` events: deep lavender dot
- `feed` events: sky dot

```html
<div class="forecast-section">
  <div class="sec-title">Schedule forecast</div>
  <div class="forecast-list">
    <!-- For each event: -->
    <div class="forecast-item">
      <span class="forecast-time">14:30</span>
      <span class="forecast-dot forecast-dot-sleep"></span>
      <span class="forecast-label">Nap 3 start</span>
    </div>
  </div>
  <div class="forecast-note">
    Times are estimates based on the plan. Follow his cues.
  </div>
</div>
```

Style the forecast similarly to the timeline section but with a slightly different appearance to distinguish "projected" from "actual":
- Use dashed left borders or slightly muted colours
- Add a small italic note at the bottom: "Times are estimates based on the plan. Follow his cues."
- forecast-time: JetBrains Mono, 13px, `var(--text-mid)`
- forecast-dot: 8px circles, same colours as guidance states
- forecast-label: 14px, `var(--text)`, font-weight 600

---

## Important implementation notes

1. **Efficient DOM updates**: Do NOT replace innerHTML of the entire dashboard every render cycle. Instead:
   - Build the full HTML string
   - Compare against a cached `lastHTML` variable
   - Only update the DOM if the HTML has changed
   - For ticking timers (wake window, nap timer), update ONLY those specific text nodes using `querySelector` when the rest of the page hasn't changed
   - Render every 5 seconds, fetch new API data every 5 minutes

2. **The app currently stores BabyBuddy URL, API token, and Gemini API key in localStorage.** Keep this behaviour. The setup screen has three fields: URL, Token, Gemini key (optional). The Gemini chat functionality should be preserved as-is if present, or omitted if not yet implemented — focus on the three changes above.

3. **All times should use `en-GB` locale** (24-hour format): `toLocaleTimeString('en-GB', {hour:'2-digit', minute:'2-digit'})`

4. **The guidance engine** (the function that generates contextual advice cards) should be aware of:
   - The manual sleep state (don't show wake window alerts while sleeping)
   - The forecast (reference upcoming targets in guidance text where helpful)
   - The temperament selector (hide it while baby is sleeping, show it when awake)

5. **Auto-clear manual sleep**: On each data refresh, check if the most recent completed sleep record from the API has a start time that is close to (within 5 minutes of) the manual sleep start time. If so, BabyBuddy has recorded the sleep, so clear the manual state.

6. **Mobile-first**: This is used on phones at 3am. Large tap targets (minimum 44px), readable fonts, no tiny text. Bottom bar with refresh button fixed to viewport bottom with safe area inset padding.

---

## Files to edit

Only edit `index.html` in the root of the repo. The entire app (HTML, CSS, JS) is in this single file.

## Testing

After editing, the app should:
1. Load the setup screen with the gradient background
2. Connect to BabyBuddy and show the dashboard
3. Show the sleep toggle card at the top
4. Show the wake window timer (green/amber/red) when awake
5. Show the sleep timer with ideal wake time when "Asleep" is tapped
6. Show the forecast schedule section
7. Not jitter or flicker during the 5-second render cycle
8. Work on mobile Safari and Chrome (iOS and Android)
