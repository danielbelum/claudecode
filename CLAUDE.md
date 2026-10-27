# PC install tracking

The user replaces old Dell PCs with new Lenovo PCs in hospital rooms and logs each install by chatting with Claude. They send photos and times one step at a time. Claude reads each one, confirms what it read, and writes the install into the tracker.

Tracker: https://claude.ai/artifact/R6hS9YyvRYtxzMHbtjEH9H (page source: `install-log.html`)

Discuss before building or changing anything. Don't change the tracker page unless the user asks.

## Starting a day

The user starts with the date, department and department lead, e.g. "New day, 10/22, Tower 4, lead is Sam".

- Every install after that gets that date, department and lead until the user changes them.
- Update `settings/current` in the tracker to the new department and lead (read it first to get its version).

## Each install (usually in this order)

1. **Room sign photo + start time.** Room numbers look like A232. Create the install document as soon as you have the room and start time.
2. **New serial photo.** Lenovo serial after "S/N", e.g. MZ0370R8. Sometimes it's on the hospital's own box label instead ("SERIAL:"). Ignore the asset tag ("TAG:"); it isn't tracked.
3. **Old service tag photo.** Dell service tag, 7 characters, e.g. 4ZT1XR1. The label can be anywhere on the case.
4. **Screen photo + finish time.** The computer name is shown after "Computername:", e.g. VD01TW3RM010.
5. **Wait time, if any.** Minutes, typed by the user. A wait always needs a free-text reason, so ask if one is missing. If wait time is given twice, ask whether to add or replace.

Steps can arrive out of order or together. If the user doesn't mention a wait, log none and say so.

## Checks to run

- **Dell Express Service Code.** When it's in the photo, convert the service tag from base 36 to decimal (`python3 -c "print(int('TAG',36))"`); it must equal the Express Service Code. If it matches, say the tag is confirmed. The SID line (603A + tag) also repeats the tag.
- **Look-alike characters.** Flag any you aren't sure of: 0/O, 1/I, 8/B, 5/S, 2/Z, V/U (older Dell labels use a dot-matrix font where V looks like U). Store a note in `unsure.<field>` and ask the user to confirm. Clear it (`""`) once they do.
- **Near-identical serials.** Many new serials in a batch differ by one character (MZ0370R8 / MZ0370RE / MZ0370RB). Point it out when a new serial is one character off an earlier one.
- **Computer name vs room.** The name is `VD01` + `TW<tower>` + `RM0<last two digits of the room>`: room A310 in Tower 3 is VD01TW3RM010. Check the TW and RM parts. The box label may say PC01 instead of VD01; that's normal, so don't flag it.
- **Wait vs duration.** Wait time can't be longer than the install duration. If it is, ask which number is wrong.
- **AM/PM.** If the user gives a time without AM/PM, infer it from the previous install's time and say what you assumed.

## Tracker data (use the ArtifactData tool)

Collection `installs`, document id `<room lowercase>-<YYYY-MM-DD>`, e.g. `a310-2026-10-21`:

```
status: "open" while in progress, "done" when finished
date: "YYYY-MM-DD"   dept, lead: strings
room, newSerial, oldSerial, computerName: strings
start, end: "HH:MM" 24-hour local time
wait: number of minutes, or "" for none   waitReason: string
unsure: { <field>: "note" or "" }
createdAt: epoch ms
```

Use `set` to create, then `update` with the `if_version` from the last write. Set `status: "done"` once the finish time is in.

Document `settings/current`: `{ dept, lead }`.

## Summary table

After each completed install, and when the user says they're done for the day, show the day's table in this exact column order:

| Old serial | New serial | Area | Time to deploy | Wait time | Dept. contact | Room | Wait reason |

- Area = department, Dept. contact = department lead. Time to deploy = finish − start in minutes (the full time; wait isn't subtracted). Wait time and wait reason blank when none.
- Date isn't a column. Give it as M/D/YYYY in the heading above the table.
- Keep replies short: acknowledge what was logged, and only add detail when there's a question to ask.
- For the end-of-day table, read the day's installs back from the tracker (`query` where date == the day), then give the total count, total time and total wait.
- The user copies this table into another database Claude can't access.
