# 10 — User Interface

## Security is not a separable concern

Every property built in `00`–`09` can be quietly undone at the UI layer.
A beautiful permission dialog nobody reads is not more secure than no
dialog at all — arguably less, since it creates the appearance of a
decision having been made. The UI here isn't a skin drawn on top of a
secure kernel; it's either where the capability model's guarantees reach
the user intact, or where they leak.

## Powerbox-style capability granting

The file-picker *is* the grant, not a separate permission system layered
on top of one. This isn't a new idea invented for this project — it's the
**powerbox** pattern from capability-security research (CapDesk, and its
better-known successor **Polaris**, built by Stiegler, Karp, Yee, Close,
and Miller specifically to run ordinary Windows programs under least
authority): an application asks for a file the way it always has, a
trusted picker (not the application itself) presents the dialog, and the
user's act of selecting a file *is* the act of authorizing access to
exactly that file — nothing more, and no separate "allow this app to
access your files: yes/no" wall to click through reflexively. Mapped onto
`08-STORAGE-AND-INTEGRITY.md`'s capability-scoped storage model directly:
the powerbox is the trusted process that actually holds broad filesystem
capability and hands out narrow ones, one file at a time, as a side
effect of normal use.

**This requires a secure input path, or it's theater.** A powerbox's
security depends entirely on the user being able to interact with the
trusted picker in a way the untrusted application cannot observe or
spoof. This is exactly where historical attempts broke: under the X
Window System, one client can inject or observe another client's input
events, so a malicious application could in principle fake the picker or
snoop what was selected — the powerbox model's core security assumption
simply didn't hold at the display-server layer. The compositor here has
to guarantee client isolation strongly enough that this can't happen —
this is a hard requirement on the compositor design, not a nice-to-have,
because every capability-grant UI in this system depends on it being true.

## Pluggable shell, same model as personality servers

The desktop shell — compositor, window manager, whatever presentation
layer a user actually sees — is itself unprivileged userspace, following
the exact pattern in `09-PERSONALITY-SERVERS.md`. Just as "what API does
this OS present" is late-bound per launch, "what does this OS look like"
is late-bound per shell choice, with no kernel involvement either way. A
minimal tiling shell, a full desktop metaphor, or a from-scratch
experiment can all run side by side, none of them privileged over the
others.

## No ambient rendering trust

The compositor composites; it does not parse or interpret arbitrary
application content. Each application owns and renders into its own
buffer; the compositor's job is placing those buffers on screen, not
decoding whatever's inside them. This keeps the same "no complex parsers
in a trusted position" discipline from `00-PHILOSOPHY.md` intact at the
display layer instead of quietly reintroducing a large, untrusted-input-
facing attack surface in the one process everything else has to trust to
draw the screen honestly.

## The capability graph inspector is the security dashboard, not just a debug tool

`05-FUN-SUBSYSTEMS.md`'s live capability graph inspector was framed there
as a debugging tool. It's also the answer to "where does an ordinary user
go to see who can access what" — instead of a conventional OS's buried,
static app-permissions settings page, it's a live, visual, glanceable
map of actual current authority, matching what's actually true right now
rather than a list of permissions granted once and forgotten.

## Anti-fingerprinting rendering discipline

If any networked rendering surface exists (a browser-equivalent, in
whatever personality hosts it), it needs to not become the thing that
undoes `07-PRIVACY-AND-IDENTITY.md`'s network-layer unlinkability work.
Tor Browser's approach is the relevant precedent: uniform window
dimensions (letterboxing rather than reporting the real screen
resolution), a deliberately limited and non-unique font-enumeration
surface, and generally making every instance look identical to every
other instance rather than expressive/customizable — customization at
this layer is a fingerprinting surface, not a feature, for anything that
touches the network.

## Performance discipline

Consistent with the "best performance," bare-metal goals: no browser-
engine-based desktop shell as a shortcut (a common move in hobby OS
projects, and a direct contradiction of the minimal-TCB, no-needless-
abstraction principles already adopted everywhere else). Direct GPU/
scanout submission, no unnecessary compositing layers, matching the same
discipline already applied to the kernel and drivers.

## Honest limits

A real usability study of Polaris (DeWitt & Kuljis, 2006) found that
even a system specifically designed to align security with usability
still ran into real problems: participants showed apathy toward security
decisions and knowingly traded security away to finish tasks faster. This
isn't a flaw unique to the powerbox idea — it's evidence that no UI
design fully solves the human element of security, and the honest
expectation here is the same: good design meaningfully reduces careless
grants, it doesn't eliminate them. This is a genuinely open, ongoing
design problem, consistent with the honest-limits sections in every other
pillar doc, not something solved by this document.

See ADR-0012 in `04-DECISIONS.md` for the decision record.
