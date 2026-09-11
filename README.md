# actic-booking-cli

Command line tool for viewing, booking and cancelling group fitness classes at
[Actic](https://www.actic.com) — the same API the `minesider.actic.no` / `minasidor.actic.se`
web app uses. Built to avoid opening the web app just to grab a CrossFit class, and to make
automated booking possible.

Unofficial. Not affiliated with Actic.

## Installation

```bash
git clone git@github.com:essoen/actic-booking-cli.git
cd actic-booking-cli
pip install -r requirements.txt        # just `requests`
ln -s "$PWD/actic" ~/bin/actic         # optional
```

Requires Python 3.8+.

## Configuration

Credentials are read from environment variables, or from the first env file that exists:

1. `$ACTIC_ENV_FILE`
2. `~/.config/actic/env`
3. `./.env`

```bash
mkdir -p ~/.config/actic
cp .env.example ~/.config/actic/env
chmod 600 ~/.config/actic/env
$EDITOR ~/.config/actic/env
```

| Variable | Default | Purpose |
|---|---|---|
| `ACTIC_USERNAME` | — | Login for My Pages |
| `ACTIC_PASSWORD` | — | Password |
| `ACTIC_CENTER` | your home center | Center ID used when `--center` is omitted |
| `ACTIC_COUNTRY` | `NO` | Country for `actic centers` |
| `ACTIC_TOKEN_FILE` | `~/.config/actic/token.json` | Where the session token is cached |
| `ACTIC_ENV_FILE` | — | Explicit path to an env file |

The token is cached in `~/.config/actic/token.json` (chmod 600) and refreshed automatically.
No password or token ever enters the repository — `.gitignore` covers `.env*` and `token.json`.

## Usage

```bash
actic classes --days 7                     # schedule
actic classes --name WOD --bookable        # only WOD classes with spots left
actic classes --date 2026-09-14
actic bookings                             # your own bookings (alias: `actic mine`)
actic book "WOD 2026-09-14 06:30"          # or a booking ID: 544p144115
actic book 544p144115 --waitlist           # accept a waiting list spot
actic cancel "WOD 2026-09-14"
actic centers
```

`--json` works on every command, before or after the subcommand, and is meant for scripts
and agents. `--center` can be repeated to merge several centers into one schedule.

### Waiting lists

The API has no separate waiting list endpoint: when a class is full, the *normal* booking
call puts you on the waiting list instead. So you never end up there by accident, `book`
exits with code 2 and reports how many people are already queued. Run it again with
`--waitlist` if you want the spot.

## Center IDs

Classes are always fetched for a specific center. If you do not set one, the CLI uses the
home center on your membership, which is taken from the login response.

List the centers in a country with `actic centers` (`--country NO`, `SE` or `DE`):

```
$ actic centers --country NO
500  Actic Norge AS ()
544  Actic Asker (Askerområdet)
545  Actic Slemmestad (Askerområdet)
```

Norway has 3 entries, Sweden around 100 and Germany around 25; the first entry in each list
is the country's head office, not a gym. IDs are stable, so put yours in `ACTIC_CENTER` once:

```bash
echo 'ACTIC_CENTER=544' >> ~/.config/actic/env
```

Booking IDs combine the center and the class: `544p144115` is class `144115` at center `544`.
Pass `--center 544 --center 545` to see two gyms in the same schedule.

## The API

Base URL: `https://webapi.actic.se/` — shared by Norway, Sweden and Germany.

The token must be sent in **both** the `Authorization: Bearer <token>` and `Access-Token: <token>`
headers. With only `Authorization`, every data endpoint returns 401 `NotAuthorized` while
`login` and `check-auth-token` still succeed — so the failure looks like an expired token.

| Method | Path | Purpose |
|---|---|---|
| POST | `login` | `{username, password}` → `{success, accessToken, person:{personId:{center,id,externalId}}, ...}` |
| POST | `check-auth-token` | `{privilege:false}` → `{success:true}` |
| GET | `db/centers/{country}` | List centers |
| GET | `persons/{externalId}/centers/{centerIds}/classesAll` | Schedule; several centers comma-separated |
| GET | `persons/{externalId}/participations/classes` | Your bookings |
| POST | `persons/{externalId}/participations/{bookingId}` | Book a class (empty body) |
| DELETE | `persons/{externalId}/participations/{bookingId}/{participationId}` | Cancel |

IDs use the form `{center}p{id}`, e.g. `544p144115`.

Rules the API enforces: at most 4 active bookings (`participations_max`), the booking window
opens 10 days before the class (`booking_starts_days_before`), and the quota is counted
differently within 48 hours of the start (`booking_no_limit_hours_before_class`).

Mapped from a HAR capture of the web app plus its frontend bundle.

## License

MIT
