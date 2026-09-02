# A home server that hosts personal data and public demos, without letting one reach the other

This is the design of a single small home server that does two jobs people usually keep apart: it
stores a family's photos, documents and media, **and** it serves things publicly on the internet.
Those two jobs have opposite security postures, and most of the interesting decisions here come
from refusing to compromise between them.

It runs on one small desktop machine with six cores and 16 GB of RAM.

**This document is the reasoning, not a script.** Commands rot within a release or two; the reasons
do not. Where a specific number appears it is there because the number itself taught something.

---

## The shape

One hypervisor. Every service in its own unprivileged container, and full virtual machines where a
container is not a hard enough boundary, or the software only ships as a whole appliance. Nothing listens on the public internet.
Anything that needs to be reachable from outside is reached through an **outbound-only tunnel**, so
the router has **no forwarded ports at all** and the home IP address is never published.

```
        internet
            │
    ┌───────┴────────┐
    │  CDN / tunnel  │   outbound connection, initiated from inside
    └───────┬────────┘   there is no other way in
            │
    ┌───────┴────────────────────────────────┐
    │  reverse proxy container               │
    │  routing, rate limiting, access rules  │
    └───┬────────────────────────────┬───────┘
        │                            │
   ┌────┴─────┐                ┌─────┴──────────────┐
   │ PRIVATE  │                │ PUBLIC             │
   │ photos   │                │ demos              │
   │ media    │                │                    │
   │ documents│                │ own guest, own net │
   │          │                │ egress blocked     │
   └──────────┘                └────────────────────┘
```

**The demos live on their own guest with no route to the private ones.** Not a firewall rule inside
a shared container. A separate guest, whose network cannot reach the rest of the LAN. I tested that after a reboot.

⚠️ **And the honest version of that is worth more than the tidy one: not everything public sits
behind that boundary.** A static site with no data of its own is a much smaller prize than a photo
library, so it ended up where it was convenient, not on the isolated guest. **The tiers are
drawn around what a compromise would reach, not around what faces the internet.** If you draw a
diagram like this for your own setup, draw what it does. Not what you meant.

---

## The decisions worth copying

### Every container is unprivileged

Root inside the container is not root on the host. This is the default in modern hypervisors and
should stay that way even when it is inconvenient, which it will be, because file ownership stops
being obvious.

⚠️ **The tax you will pay:** a directory created by the *host's* root is not writable from inside a
container, and the failure mode is horrible. Every operation returns "permission denied" while the
process otherwise looks like it is running normally. A batch job can fail on all of its work and
still exit cleanly.

**The fix is to match a directory that already works.** Do not try to reason out the id mapping
from first principles. Look at how an existing shared folder is owned and copy it.

### No inbound ports, ever

The tunnel connects outward from the server to the edge. There is no port forward, no dynamic DNS
pointing at the house, and no listening service exposed to the internet.

This costs almost nothing and removes an entire category of attack. It also means a misconfigured
service cannot accidentally become world-reachable through the router.

⚠️ **But it introduces a subtler risk that caught me out:** a container with **no
published port at all can still be world-reachable**, because the tunnel does not care about the
container's port mapping. It reaches the service by name on the internal network. If you audit
exposure by looking at published ports, you will conclude something is private when it is not.
**Audit the tunnel's routing table, not the container's port list.**

### The public workloads cannot reach the private network

A game or a demo has no legitimate reason to talk to the photo library, so it is not merely
discouraged from doing so, it is prevented at the network layer, and the rule persists across a
reboot.

📌 **A constraint like this will look, later, like an oversight.** Someone will find that the public
guest cannot download its own dependencies and will be tempted to "fix" the firewall. It is not
broken. Images are built elsewhere and shipped in as artifacts. **Before relaxing any constraint,
find out what it protects.**

### Storage is split by what the loss would mean

| Tier | Example | If it is lost |
|---|---|---|
| Irreplaceable | photos, documents | gone forever, no substitute |
| Expensive | a media library | re-obtainable, but slowly |
| Regenerable | container images, caches | rebuild in minutes |

Only the first tier justifies real backup effort. Backing up the second tier at multi-terabyte
scale is usually not worth it, **but be honest about what goes with it**: if you have invested work
*into* those files (transcoding, embedded subtitles, metadata), that work is in the irreplaceable
tier even though the files are not.

---

## Backups, and the part everyone gets wrong

Scheduled backups are the easy half. The hard half is these three questions.

### 1. Is the job still running?

**A backup job that silently stopped looks exactly like one that works**, because the destination is
still full of last month's files. Nothing errors. The folder is the right size.

**Check freshness, not existence.** Walk each destination, find the newest file, and compare its
age against the schedule. Do it on a timer, and make it complain.

### 2. Does the backup contain what you think?

Hypervisor-level guest backups capture the guest's own filesystem. They typically **do not** capture
data that is bind-mounted in from the host, which is exactly where the large and important data
usually lives.

**So the guest backup for a photo service can contain zero photos**, and it will restore perfectly
and look fine until someone opens the app. Know which of your data is inside the guest image and
which is covered by a separate job.

### 3. 🔴 Where do the copies physically live?

The question people skip.

Redundancy inside one machine is not the same as a second copy. Mirrored disks, a RAID array, a
scheduled dump to a second drive in the same chassis: every one of those is a plan for a dead disk.
A fire, a flood, a theft or a power event takes the machine, and everything inside it goes at the
same moment.

**So ask it of your own setup: how many separate buildings hold this data?** If the answer is
one, the backup strategy protects against exactly one failure mode, disk failure, and nothing else.
That can be a perfectly reasonable decision. It should be a decision rather than a discovery.

⚠️ **And check the whole path before you answer, including the parts that are not the server.**
People routinely conclude a data set is unprotected because the server does not back it up, while a
phone has been quietly syncing most of it to a cloud service for years. The reverse trap is worse:
assuming that sync covers everything, when it only ever covered what the phone itself produced.
**Enumerate what is in the set and where each part came from**, then answer.

---

## Traps, each of which cost me a real evening

- **A green exit code proves the command returned, and nothing else.** A file transfer here reported
  success while moving zero bytes. A container build cached a broken layer, exited 0, and left empty
  dependency directories that a file count walked straight past. Check the far end, not the status.
- **A ledger is a resume mechanism.** It records that a step ran. It does not record that the
  result was right. Feed it the wrong input and it will report, accurately, that the wrong thing
  was completed. Go and re-derive the current state.
- **Zero problems found and a crash on the first line print the same thing: nothing.** Store those as
  two different values rather than one blank, and put the zero on the screen where someone sees it.
- **A review tool wired to one kind of output will never examine any other kind, and will not say
  so.** Confirm it opened the artifact. Configured for it is not enough. Silence from a
  checker is the easiest thing in this whole document to mistake for approval.
- **There may be two firewalls.** If a guest cannot reach the network, the hypervisor may be
  dropping the packets before the guest ever sees them, so the guest's own counters read zero drops
  and look innocent.
- 🔴 **Treat downloaded media as hostile input.** You did not write those filenames, and admin scripts
  often run as root from a scheduler. **Pass arguments as a list, never build a shell string.** A
  filename containing `$(...)` executes inside double quotes.

---

## What this is not

It is not high availability. One machine, one power supply, one location. Services go down when it
reboots, and that is an accepted trade for simplicity and cost.

It is not a recommendation to run someone else's stack. The value here is the set of questions:
what would losing this mean, what can reach what, who can open the door, and how would you find out
if a safeguard had quietly stopped working.

**That last one is the theme.** Almost every real problem in this setup was not a component
failing. It was a safeguard that had stopped working while continuing to report success.

