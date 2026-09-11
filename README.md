# actic-booking-cli

Kommandolinjeverktøy for å se, booke og avbooke gruppetimer hos [Actic](https://www.actic.no)
— samme API som `minesider.actic.no` bruker. Laget for å slippe å åpne weben for å ta en
CrossFit-time, og for å kunne automatisere påmelding senere.

Uoffisielt. Ingen tilknytning til Actic.

## Installasjon

```bash
git clone git@github.com:essoen/actic-booking-cli.git
cd actic-booking-cli
pip install -r requirements.txt        # kun `requests`
ln -s "$PWD/actic" ~/bin/actic         # valgfritt
```

Krever Python 3.8+.

## Oppsett

Innlogging leses fra miljøvariabler, eller fra første env-fil som finnes:

1. `$ACTIC_ENV_FILE`
2. `~/.config/actic/env`
3. `./.env`
4. `~/.env.vps`

```bash
mkdir -p ~/.config/actic
cp .env.example ~/.config/actic/env
chmod 600 ~/.config/actic/env
$EDITOR ~/.config/actic/env
```

Tokenet caches i `~/.config/actic/token.json` (chmod 600) og fornyes automatisk.
Verken passord eller token havner i repoet — `.gitignore` dekker `.env*` og `token.json`.

Finn ditt eget senter med `actic centers`; sett `ACTIC_CENTER` hvis du ikke trener på
Asker (544).

## Bruk

```bash
actic classes --days 7                     # timeplan
actic classes --name WOD --bookable        # kun WOD-timer det er plass på
actic classes --date 2026-09-14
actic mine                                 # egne bookinger
actic book "WOD 2026-09-14 06:30"          # eller booking-ID: 544p144115
actic book 544p144115 --waitlist           # godta ventelisteplass
actic cancel "WOD 2026-09-14"
actic centers
```

`--json` på alle kommandoer gir maskinlesbar output, tenkt for skript og agenter.
`--center` kan gjentas for å slå sammen flere sentre.

### Venteliste

APIet har ikke noe eget ventelisteendepunkt: er timen full, gir den *vanlige* booke-kallet
deg en ventelisteplass i stedet. For at det aldri skal skje ved et uhell avslutter `book`
med exit-kode 2 og forteller hvor mange som allerede står i kø. Kjør på nytt med
`--waitlist` hvis du vil ha plassen.

## APIet

Base: `https://webapi.actic.se/` — felles for Norge, Sverige og Tyskland.

Tokenet må sendes i **begge** headerne `Authorization: Bearer <token>` og
`Access-Token: <token>`. Med bare `Authorization` svarer alle data-endepunkter
401 `NotAuthorized`, mens `login` og `check-auth-token` virker — feilen ser derfor
ut som et utløpt token.

| Metode | Path | Formål |
|---|---|---|
| POST | `login` | `{username, password}` → `{success, accessToken, person:{personId:{center,id,externalId}}, ...}` |
| POST | `check-auth-token` | `{privilege:false}` → `{success:true}` |
| GET | `db/centers/{country}` | Senterliste |
| GET | `persons/{externalId}/centers/{centerIds}/classesAll` | Timeplan, flere sentre komma-separert |
| GET | `persons/{externalId}/participations/classes` | Egne bookinger |
| POST | `persons/{externalId}/participations/{bookingId}` | Book time (tomt body) |
| DELETE | `persons/{externalId}/participations/{bookingId}/{participationId}` | Avbook |

ID-er er på formen `{center}p{id}`, f.eks. `544p144115`.

Regler APIet håndhever: maks 4 aktive bookinger (`participations_max`), bookingvinduet
åpner 10 dager før timen (`booking_starts_days_before`), og kvoten teller annerledes
innenfor 48 timer før start (`booking_no_limit_hours_before_class`).

Kartlagt fra en HAR-fangst av minesider.actic.no pluss frontend-bundelen.

## Lisens

MIT
