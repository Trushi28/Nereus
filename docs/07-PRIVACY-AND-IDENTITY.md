# 07 — Privacy & Network Identity

## Naming the actual property before designing for it

Privacy/anonymity research (Pfitzmann & Hansen's terminology, the reference
vocabulary most anonymity-system design — including Tor's — is built on)
distinguishes several properties that are easy to conflate:

- **Anonymity** — a subject can't be identified within a set of subjects.
- **Unlinkability** — two observed items (e.g. two sessions, two
  connections) are no more and no less related, after observation, than
  they were before it. The channel is visible; which real-world identity
  is behind it, across time, is not.
- **Undetectability** — an observer can't tell an item of interest (a
  message, a party, a channel) exists *at all*, as opposed to not existing.
- **Unobservability** — undetectability for uninvolved parties, plus
  anonymity even among involved parties, in the strongest definitions.

**"The channel exists but the user is different" is an unlinkability
target, not an undetectability target** — and that's the right, achievable
one to design for. A third party can see this machine is on the network,
sending packets, behaving like *something*. What they shouldn't be able to
do is correlate "the device I saw yesterday" with "the device I'm seeing
today." Being precise about this matters: designing for unlinkability and
believing you have undetectability is exactly the kind of overconfidence
that gets someone deanonymized by an adversary using a technique the
design never actually addressed.

## Mechanisms

### MAC address randomization
Generate a locally-administered, random unicast MAC (set the
locally-administered bit, clear the multicast bit, CSPRNG the rest) per
network attach. This is already standard practice on iOS/Android/Windows/
GrapheneOS for Wi-Fi probe requests and associations — not novel, but
table stakes, and worth getting the granularity right: per-SSID rotation
avoids captive-portal/DHCP-reservation breakage; per-connection rotation
is more private but breaks networks that pin a lease to a MAC. Make this a
user-facing policy choice per network, not a single global setting.

### IPv6 privacy addresses (RFC 4941)
The historical, notorious IPv6 privacy hole: a stateless-autoconfigured
address derived via EUI-64 embeds the interface's MAC address directly
into every address the machine ever uses — the exact opposite of what you
want. RFC 4941 exists specifically to fix this: generate temporary
interface identifiers via CSPRNG, rotate them on a schedule (session-based
or time-based), and prefer the temporary address for all outbound
connections, keeping the stable address (if any) unadvertised.

### Egress rotation — the honest version of "randomize the IP"
A single host cannot simply pick an arbitrary public IP address on demand
— that's a function of routing, not something an end host controls. The
achievable version of this idea: route outbound traffic by default through
a rotating anonymity overlay (an onion-routing layer, Tor-protocol-
compatible or purpose-built) so the address a remote party *sees* is a
frequently-rotating relay/exit address, never this machine's actual
attachment point. This is where "IP randomization" actually lives
architecturally — at the overlay-routing layer, not as a raw host-OS
IP-address-picker, which doesn't correspond to how IP routing works.

### TCP/IP stack fingerprint normalization
Tools like `p0f` and `nmap -O` fingerprint an OS remotely from passive
stack quirks: initial TTL, TCP window size and scaling, options presence
and ordering, IP ID sequence behavior, ICMP quirks. The design goal here
is counterintuitive but important: **do not build a unique-looking
"private" stack — blend into the largest existing population instead.**
A stack with a distinctive, novel fingerprint is *more* identifiable, not
less, the moment two devices running it are ever compared. Match a
common, high-population profile (a widely-deployed Linux kernel's default
behavior is a reasonable target) rather than inventing a new one.

### Per-application network compartmentalization
Different applications get different network identities and, where the
threat model calls for it, different egress paths — the Qubes/Whonix
precedent (separate VMs/namespaces per trust domain so one leaky
application can't be correlated against another). This is a strong,
practical mitigation against the common real-world failure mode: one app
leaks an identifying detail that retroactively deanonymizes everything
else that shared its network identity.

### DNS privacy
DNS-over-HTTPS/TLS by default, with oblivious DNS or DNS-over-onion-overlay
as a stronger option. Every mechanism above is undone if plaintext DNS
queries are sitting on the wire revealing exactly what the encrypted,
rotated, fingerprint-normalized connection was actually for.

## Honest limits — what this does not defend against

- **A global passive adversary correlating traffic timing/volume at both
  ends of the overlay** can still deanonymize in principle, regardless of
  IP/MAC rotation. This is Tor's own stated threat-model exclusion, not a
  gap unique to this design — no unlinkability scheme built on relays
  defeats an adversary who can watch both the entry and the exit.
- **Application-layer fingerprinting** (installed-font lists, screen
  resolution, timing side channels, canvas fingerprinting if a browser
  is involved) is a separate problem from network-layer identity and
  needs its own hardening — network unlinkability doesn't help if the
  browser leaks a unique fingerprint at the content layer.
- **Endpoint compromise defeats all of this.** If malicious code is
  running with this machine's network privileges, rotating MAC/IP/DNS
  doesn't stop it from exfiltrating data through whatever channel it
  already has — this is a storage/execution problem, addressed in
  `08-STORAGE-AND-INTEGRITY.md`, not a network-identity problem.

See ADR-0009 in `04-DECISIONS.md` for the decision record.
