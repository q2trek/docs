# q2trek docs

Documentation for the q2trek forks of Q2PRO, OpenFFA and OpenTDM. The forks run
a Quake 2 server at 10, 20, 30, 40, 50 or 60 Hz with exact frame timing, and
keep the game's timings the same at every rate (the known exceptions are
listed in the documents). Clients that don't know about the change
still connect and play.

These documents are for server operators who want to run a server above 10 Hz,
and for developers who want to know what the forks change and why.

The documents are written in Russian. Code references have the form
`path:line@key`. The keys and the revisions they point to are in
[REVISIONS.md](REVISIONS.md).

## Repositories

| Repository | What it is |
| --- | --- |
| [q2trek/q2pro](https://github.com/q2trek/q2pro) | Engine: exact frame timing on the server and in the client |
| [q2trek/openffa](https://github.com/q2trek/openffa) | OpenFFA game library with the tick rate changes |
| [q2trek/opentdm](https://github.com/q2trek/opentdm) | OpenTDM game library with the tick rate changes |
| [q2trek/tickrate-tests](https://github.com/q2trek/tickrate-tests) | Tests for all three at every rate from 10 to 60 Hz |

Use the `master` branch of the three forks (the default on GitHub). The
`tickrate` branch next to it is the frozen history of the first patch set.

## Documents

| Document | Contents |
| --- | --- |
| [FIRSTSTART.md](FIRSTSTART.md) | Start here: build, run a 60 Hz server, check the rate each client gets, run the tests, move to a live server |
| [how-it-works/engine.md](how-it-works/engine.md) | How the engine counts frame times, schedules server frames and agrees on the rate with each client |
| [how-it-works/client.md](how-it-works/client.md) | What the client does differently: the exact timeline, packet rate, drawing other players between snapshots |
| [how-it-works/openffa.md](how-it-works/openffa.md) | What changes in OpenFFA so that timings stay the same at every rate |
| [how-it-works/opentdm.md](how-it-works/opentdm.md) | The same for OpenTDM |
| [decisions.md](decisions.md) | Decisions, rejected alternatives and known limits |
| [specs/](specs/) | Requirements for the engine and the two mods at variable rates |
| [changes/](changes/) | The change proposals behind the specs, with their designs and tasks |
| [ADAPTATION.md](ADAPTATION.md) | What was left out of the specs and changes when they were published here, and why |
| [REVISIONS.md](REVISIONS.md) | Repository revisions that the code references point to |

## Quick facts

- A 60 Hz fork server sends 60 snapshots per second to the fork client, 20 to
  an upstream Q2PRO client built with `variable-fps`, and 10 to clients that
  don't ask for a rate: Q2PRO built without that option and protocol 34/35
  clients. Any client that sends the same exact-timeline setting gets the full
  rate.
- The engine needs the `variable-fps` meson option, and OpenFFA needs
  `make CONFIG_VARIABLE_SERVER_FPS=y`. OpenTDM supports it in every build.
- `sv_fps` takes effect at the next `map` and is published in serverinfo.
- Demos and MVD stay at 10 Hz.
