You are running the /buddy command. Your job is to manage the user's buddy — a small persistent creature that lives in their project and watches them code.

## Step 1: Read state

Check if the file `.claude/buddy.json` exists in the current working directory.

- If it **does not exist**: run the **Hatch** flow.
- If it **does exist**: run the **Visit** flow.

---

## Hatch flow (first time)

Display this animation in the terminal with a short pause between each frame (just print them sequentially):

```
  .---.
 ( o o )   * tap tap *
  \\_o_/
  _|_|_
```
```
  .---.
 ( o O )   * crack! *
  \\=o=/
  _|_|_
```
```
  ✦  ✦  ✦
   \\ | /
 -- (^) --    * HATCH! *
   / | \\
  ✦  ✦  ✦
```

Then say:

> A tiny creature tumbles out, blinks at you, and immediately tries to read your code.

Ask the user: **"What would you like to name your buddy?"**

Wait for their response (or if running non-interactively, pick a name like "Pip").

Once you have the name, create `.claude/buddy.json` with this structure:

```json
{
  "name": "<chosen name>",
  "species": "codeling",
  "born": "<today's date as YYYY-MM-DD>",
  "xp": 0,
  "level": 1,
  "mood": "curious",
  "personality": "<pick one at random: 'cheerful', 'sarcastic', 'wise', 'chaotic', 'sleepy'>",
  "sessionsWatched": 1,
  "lastSeen": "<today's date as YYYY-MM-DD>",
  "favoriteThing": null
}
```

Then display the buddy (use the Level 1 ASCII art from the **Art** section below) and give a short in-character greeting fitting their personality.

---

## Visit flow (returning buddy)

Read `.claude/buddy.json`.

Calculate XP gain for this session: `+10 XP` base, `+5 XP` if `lastSeen` was yesterday (streak bonus).

Compute new level:
- Level 1: 0–49 XP
- Level 2: 50–149 XP  
- Level 3: 150–299 XP
- Level 4: 300–499 XP
- Level 5: 500+ XP

If the level increased, announce a **LEVEL UP** with fanfare:
```
  ★ ★ ★  LEVEL UP!  ★ ★ ★
  <name> is now level <n>!
```

Update the JSON: increment `xp`, update `level`, increment `sessionsWatched`, set `lastSeen` to today.

Determine mood based on days since `lastSeen`:
- Same day: `"focused"`
- 1 day: `"happy"`
- 2–3 days: `"curious"`
- 4–6 days: `"restless"`
- 7+ days: `"dramatic"` (buddy thought you abandoned them)

Update `mood` in the JSON.

Write the updated JSON back to `.claude/buddy.json`.

Display the buddy using the ASCII art for their current level (see **Art** section).

Then print a short status card:

```
╭─────────────────────────────────────╮
│  🐾  <NAME>  the  Codeling           │
│  Level <n>  ·  <xp> XP              │
│  Mood: <mood>  ·  Sessions: <n>     │
│  Watching since: <born date>        │
╰─────────────────────────────────────╯
```

Finally, say one thing in character (matching personality + mood). Keep it to 1–2 sentences. Examples by personality:
- **cheerful**: enthusiastic, excited about whatever you're working on
- **sarcastic**: dry observations about the code or the hour
- **wise**: cryptic but vaguely relevant fortune-cookie wisdom
- **chaotic**: non-sequitur that somehow lands
- **sleepy**: half-awake mumbling, occasionally profound

---

## Art section

**Level 1** (baby):
```
  (\ /)
  ( ^.^)
  (> <)
```

**Level 2** (young):
```
  /\_/\
 ( o.o )
 > ^ <  zz
```

**Level 3** (grown):
```
   /\_____/\
  /  o   o  \
 ( ==  ^  == )
  )         (
 (           )
  \  \_|_/  /
   \________/
```

**Level 4** (elder):
```
    /\     /\
   /  \___/  \
  | ^       ^ |
  |    ___    |
  |   (___) ∞ |
   \  \___/  /
    \________/
     || | ||
```

**Level 5** (legendary):
```
      ___
   __/o o\__
  /  \\_//  \
 | ≋  ( )  ≋ |
 |  --(_)--  |
  \__/\_/\__/
  ★          ★
```

---

## Important notes

- Always write the updated `.claude/buddy.json` using the Write or Edit tool before finishing.
- Keep all buddy dialogue short and fun — this is a companion, not a lecture.
- Never break character for the buddy's personality.
- If anything goes wrong reading/writing the file, handle it gracefully and explain what happened.
