# Notification Center

An [Omarchy](https://omarchy.org) bar widget that keeps the notifications you
were sent. A bell on the right of the bar, a dot on it when something has come
in, and a panel of everything you were told, in the order you were told it,
still there tomorrow.

Omarchy already shows you a notification once. This is the answer to the other
question, the one that comes ten minutes later while you are in the middle of
something else: *what did that say?*

## Where the notifications come from

Omarchy's notification service writes every notification to disk on its way
past: one JSON file per popup under
`~/.local/state/omarchy/notifications/`, moved into `history/` when it leaves
the screen. That is the source here, and nothing in this plugin writes to those
directories — it only reads them.

What it is not is a history you can read. It keeps ten files, deletes the
eleventh, and deletes the icon it was keeping for it at the same time. Ten is
the right number for a service whose job is replaying the toasts you just
missed, and far too few for the question this panel exists to answer.

So `bin/notification-center` copies each file out of there the moment it
lands, into an archive of its own with the icon copied beside it. It follows
the directory with inotify rather than polling it, so a notification is
archived before its toast has finished appearing. The archive keeps 30 days or
1000 notifications, whichever runs out first, and both are settings.

That also means it only catches what arrives while the shell is running, which
is every notification you were actually shown.

## Install

```bash
omarchy plugin add https://github.com/jankeesvw/omarchy-notification-center.git --enable
```

That is the whole of the setup. It needs `jq` and `inotifywait`
(`inotify-tools`), both of which Omarchy already has.

The bell lands on the right of the bar. Leave it at the far right: the panel is
pinned to the right edge of the screen whatever happens, so a bell in the
middle of the bar is a bell whose panel opens somewhere else.

## What it shows

**In the bar**, a bell, and a dot on it when something has arrived since you
last opened the center. Not a number, by default: how many is a question you
ask once you are already interested, and a bar you have to read is a bar you
stop reading. Set **Mark what you have not read** to `Count` for the number, or
`None` for a bell that never changes.

Right-clicking the bell silences notifications without opening anything —
deciding you want quiet and wanting to read the backlog are opposite impulses.

**In the panel**, one card per notification, newest first, under the day it
arrived on: *Today*, *Yesterday*, then the weekday for the rest of the week and
the date beyond it. Each card carries the app's own icon, what it said, and how
long ago — minutes while that is still the useful answer, then the clock.

- **Clicking a card** does what the notification itself asked for. A screenshot
  toast still opens its screenshot a week later; a chat notification, which
  almost never registers an action, focuses the app that sent it. That is the
  same fallback Omarchy's own toasts use, so a click here lands where a click
  on the toast would have.
- **The × in the corner**, or a right-click anywhere on the card, removes one.
- **Clear** empties the archive, and asks a second time before it does.
- **The bell in the header** is Do Not Disturb, the same switch as the bar's
  DND indicator and the menu's. It is here because silencing notifications and
  catching up on them are the same conversation.
- **The magnifier**, or `/`, searches everything kept — app, subject and
  message. Escape leaves the search, Escape again closes the panel.

Notifications that arrived while you were away are marked with a dot in the
accent colour, and the marks survive the panel being drawn: opening the center
makes everything read, but the list you are looking at still shows you which
ones were new when you opened it.

## Settings

| Setting | Default | What it does |
| --- | --- | --- |
| Mark what you have not read | Dot | `Dot`, `Count`, or `None` on the bell. |
| Keep notifications for | 30 days | Older than this is deleted, icon and all. |
| Keep at most | 1000 | A ceiling regardless of age. Whichever limit is hit first wins. |
| Clicking a notification | Auto | `Auto` runs what the notification asked for and falls back to focusing the app; `Focus the app` never runs a stored command; `Nothing` makes the list read-only. |
| Show the message text | on | Off leaves the sender and subject only — the version to run on a screen other people can see. |
| Panel width | 420 | In the shell's spacing units. |
| List height | 480 | How tall the list grows before it scrolls. |

## Where things are kept

```
~/.local/state/omarchy-notification-center/
  archive.jsonl      one notification per line, oldest first
  images/            a copy of each icon, named after its notification
  seen               when the center was last opened
```

Which is worth knowing for one reason: **that file is every notification you
have been sent.** Chat messages, two-factor codes, whatever an app decided to
put in a popup. It is readable only by you and it never leaves the machine, but
it is not something to sync, back up carelessly, or hand to anything else.

`Keep notifications for` is the setting that limits the damage, and one day is
a perfectly reasonable answer to it.

To take the whole thing out:

```bash
omarchy plugin remove jankeesvw.notification-center
rm -rf ~/.local/state/omarchy-notification-center
```

The second line is deliberately not part of the first: removing a plugin by
accident should not cost you the archive.

## The command line

`bin/notification-center` is the whole of the storage side and is useful on its
own — it is how you search further back than the panel loads:

```
watch              follow the notification service and archive what it receives
sync               catch up on anything not archived yet
list [LIMIT]       the archive as JSON, newest first (default 200)
remove KEY         drop one notification
clear              drop all of them
seen [MS]          read or set when the center was last opened
unread             how many arrived since then
seed [N]           fill the archive with test traffic
prune              apply the retention limits now
```

Everything prints JSON, so nothing that goes wrong reaches the panel as a parse
error.

```bash
# everything Slack sent you last week, as text
notification-center list 2000 | jq -r '.[] | select(.app == "Slack") | "\(.timestamp) \(.summary): \(.body)"'
```

## Testing it without waiting for a week of notifications

Synthetic input does not reach the shell, so there is an IPC hook instead:

```bash
omarchy-shell jankeesvw.notification-center.test seed 25   # fill it with plausible traffic
omarchy-shell jankeesvw.notification-center.test clear     # empty it again
omarchy-shell jankeesvw.notification-center.test reload    # re-read the archive
```

Seeded entries all have keys beginning `seed-`, so they can be taken back out
of a real archive by hand:

```bash
A=~/.local/state/omarchy-notification-center/archive.jsonl
grep -v '"key":"seed-' "$A" > "$A.tmp" && mv "$A.tmp" "$A"
```

The panel itself is opened and closed over IPC, which is also how you bind it
to a key:

```bash
omarchy-shell jankeesvw.notification-center toggle
```

## Licence

MIT.
