---
name: agro-socials-cloud
description: "Cloud/Routine varianta agro-socials-local skillu. Jedním během (bez interaktivního uživatele) najde dnešní dávku publikovaných článků na Profifarmar.cz (typicky 3/den), vybere 2 nejatraktivnější pro sociální sítě, pro každý vytvoří Canva vizuál, nahraje na Cloudinary a naplánuje přes Buffer na 2 pevné časy (07:00 / 20:00 Europe/Prague) — Buffer je pak sám releasne. Určeno pro Claude Code cloud Routine (Linux, bash/curl/jq, žádný interaktivní checkpoint). Trigger keywords: agro social cloud, routine social, cloud buffer schedule, naplánuj příspěvky, agro social routine."
---

# Agro-Socials-Cloud Skill

Cloud/Routine verze `agro-socials-local`. Běží **jednou denně, autonomně**, bez čekání na potvrzení uživatele.
V jednom běhu:

1. najde **dnešní dávku** nově publikovaných článků (na webu vychází typicky **3 články/den** ve stejnou dobu),
2. z nich vybere **2 nejatraktivnější pro sociální sítě** (viz kritéria v Kroku 1),
3. pro každý vytvoří Canva vizuál → Cloudinary URL,
4. napíše IG/FB texty,
5. naplánuje všechny 4 posty (2× IG + 2× FB) přes Buffer na `customScheduled` s pevným `dueAt`
   **07:00 / 20:00 Europe/Prague** — Buffer je pak releasne sám, i když routine dávno doběhla.
6. zapíše publikované (i vědomě přeskočené) články do `posted-log.json` a commitne do repa (ochrana proti duplicitám).

> **Bez interaktivního checkpointu.** Na rozdíl od `agro-socials-local` tahle verze **nečeká na potvrzení** —
> v Routine není nikdo, kdo by potvrdil. Místo toho se vizuály + texty zapíšou do run summary
> (Krok 8) pro zpětnou kontrolu po doběhnutí.

> **Notifikace jen při chybě.** Když běh proběhne v pořádku, **neposílej žádnou push notifikaci** —
> run summary v transcriptu stačí. Notifikace se posílá **výhradně** při problému (viz Krok 9).

---

## Předpoklady prostředí (cloud Routine)

- **Connectory zapnuté v routine:** Canva (Cloudinary a Buffer se volají přímo přes HTTP, nejsou
  potřeba jako Claude connector — viz Krok 4 a Krok 6).
- **Environment variables** (nastavit v `claude.ai/code/routines` při vytváření routine, sekce Environment —
  **ne** Windows `SetEnvironmentVariable`, to v Linux kontejneru nic neznamená):
  - `AI_API_KEY` — Profifarmar API token
  - `BUFFER_API_KEY` — Buffer token
  - `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
- **Setup script routine** by měl mít `curl` a `jq` (na většině cloud images jsou předinstalované;
  pokud ne, `apt-get install -y curl jq`).
- **Repo** obsahuje `posted-log.json` (stačí `[]` na začátku) — do něj se zapisují ID/URL už publikovaných
  (i vědomě přeskočených) článků, aby se stejný článek neopakoval ani znovu nezvažoval.
- **Push oprávnění:** commit `posted-log.json` půjde jen na `claude/`-prefixed branch, pokud v repo
  nastavení routine nezapneš "Allow unrestricted branch pushes" — bez toho se stav neuloží zpět do main.

---

## Sdílené proměnné (per článek, viz mapování kategorií a region v `agro-socials-local`)

Stejné jako v `agro-socials-local`: `[TITULEK]`, `[KATEGORIE]`, `[DATUM]`, `[DATUM_SLUG]`, `[COVER_URL]`, `[ID]`,
`[CLANEK_URL]`. Mapování `category_id → [KATEGORIE]` a auto-detekce regionu zůstávají beze změny
(viz `agro-socials-local/SKILL.md`).

---

## Krok 0 — Zkontroluj a sluč nedokončený PR z předchozího běhu

Než začneš cokoliv číst nebo plánovat, ověř, jestli z předchozího běhu nezůstal otevřený,
nesloučený pull request (větev `claude/`-prefixed → `main`). Pokud ano a je `mergeable_state: clean`,
rovnou ho sluč (`merge_pull_request`), teprve pak pokračuj Krokem 1.

> **Proč je to nutné:** commit `posted-log.json` (Krok 7) může skončit jen na `claude/`-prefixed
> větvi, ne přímo v `main` (viz Předpoklady prostředí). Pokud PR z minulého běhu zůstane
> nesloučený, tenhle běh by četl zastaralý/prázdný `posted-log.json` z `main` a mohl by
> vybrat už dřív publikovaný (nebo vědomě přeskočený) článek. Automatické sloučení na začátku
> běhu tohle riziko eliminuje bez nutnosti manuálního zásahu uživatele.

> **Pojistka, ne běžná cesta.** Od té doby, co Krok 7 slučuje PR hned v tom běhu, který ho založil,
> by tady normálně nemělo nic zbýt. Když přesto ano, znamená to, že minulému běhu sloučení selhalo
> (konflikt, oprávnění, červená CI) — tak se na ten PR podívej, než ho slučíš. Konflikt Krok 0
> neřeší: nech ho otevřený a napiš to do notifikace podle Kroku 9.

---

## Krok 1 — Najdi dnešní dávku a vyber z ní 2 nejatraktivnější články

```bash
set -euo pipefail

: "${AI_API_KEY:?AI_API_KEY chybí — nastav v Environment routine}"

# ⚠️ KRITICKÉ: bez parametru `?limit=` endpoint vrací jen 100 záznamů řazených podle `id`
# (UUID) — NE podle `published_at`! Protože UUID jsou prakticky náhodná, tohle ořezání
# NENÍ "posledních 100 podle data" — nejnovější články tak mohou být tiše mimo okno,
# i když existují (ověřeno v provozu 2026-08-11: bez limitu se ztratily 2 ze 3 článků
# nejnovější dávky; ověřeno znovu 2026-08-13 s `limit=1000` — pořád platí, endpoint
# limit respektuje a vrací přesně tolik řádků, kolik se o něj řekne). VŽDY volej s vysokým
# limitem (10000 s rezervou pro budoucí růst) a řaď si výsledek podle `published_at` sám.
response=$(curl -sS -H "Authorization: Bearer $AI_API_KEY" \
  "https://profifarmar.cz/api/webhook.php?limit=10000")

# posted-log.json = [{ "id": "...", "url": "...", "posted_at": "...", "note": "(volitelně, u přeskočených)" }, ...]
posted_ids=$(jq '[.[].id]' posted-log.json 2>/dev/null || echo "[]")

candidates=$(echo "$response" | jq --argjson posted "$posted_ids" '
  (.data // .)
  | map(select(.status == "published" and .cover_image_url != null))
  | map(select(.id as $id | ($posted | index($id)) == null))
  | sort_by(.published_at) | reverse
')

echo "$candidates" > /tmp/candidates.json
echo "Nalezeno $(echo "$candidates" | jq length) nepostnutých/nezvážených kandidátů."

# Dnešní dávka = kandidáti se stejným (nejnovějším) published_at časem — na webu vychází
# typicky 3 články/den ve stejnou dobu (stejný timestamp na vteřinu přesně).
latest_ts=$(echo "$candidates" | jq -r '.[0].published_at // empty')
todays_batch=$(echo "$candidates" | jq --arg ts "$latest_ts" '[.[] | select(.published_at == $ts)]')
echo "$todays_batch" > /tmp/todays_batch.json
echo "Dnešní dávka: $(echo "$todays_batch" | jq length) článků."
```

**Výběr 2 z `todays_batch`** (typicky 3 články → vyber 2 nejatraktivnější pro sociální sítě):
- **Vizuál nejdřív** — stáhni/prohlédni `cover_image_url` každého kandidáta. Vyřaď článek, jehož
  cover **neodpovídá obsahu** (např. na fotce je jiná značka stroje, než o které článek píše —
  reálný případ z provozu: článek o New Holland T7 XD měl cover s traktorem značky Fendt).
  Mezi zbylými dej přednost dynamickým, akčním záběrům (stroj v pohybu/práci) před statickými.
- **Atraktivita/aktuálnost** — zvýhodni témata s vizuálním „wow" efektem (nová technika, velká čísla
  ve špičkovém výkonu, neobvyklá inovace) a témata, která jsou **právě teď sezónně relevantní**
  (např. v srpnu obsah o probíhající sklizni > obecná firemní zpráva).
- **Sezónní/časový konflikt** — stejně jako dřív: přeskoč články, které by v den zveřejnění postu
  působily zastarale nebo zavádějícně (např. „jarní mrazy" v létě, sklizňová prognóza z jara
  když už je známá skutečná sklizeň, lhůta v článku už uplynula).
- Pokud po filtrování zbydou **méně než 2 použitelné** články v dnešní dávce (kolize/vyřazení),
  dobírej chybějící z `/tmp/candidates.json` mimo dnešní dávku (další nejnovější nepostnuté/
  nezvážené kandidáty bez stejných problémů).
- **Nevybraný 3. (a další) článek z dnešní dávky zapiš do `posted-log.json` jako přeskočený**
  (pole `note` s důvodem) — viz Krok 7 — ať se zítra znovu nenabízí jako kandidát.
- Ulož vybrané jako `$article1`, `$article2` (JSON objekty) — z každého vytáhni
  `[TITULEK]`, `[KATEGORIE]`, `[DATUM]`, `[DATUM_SLUG]`, `[COVER_URL]`, `[ID]`, `[CLANEK_URL]`
  stejně jako v `agro-socials-local` Kroku 1.

---

## Krok 2 — Pro KAŽDÝ z 2 článků: naplň Canva šablonu (MCP)

Identické s `agro-socials-local` Krokem 2 (šablona `DAHOBdpJ1tk` — ProfiFarmář 4:5 Stacked). Aktuální
Canva MCP tooling: `copy-design` → `upload-asset-from-url` → `read-design` (`open_transaction: true`,
vrátí `transaction_id` a element locator_ids) → `edit-design` (operace `replace_text`/`update_fill`,
`finalize: "keep_open"`) → `edit-design` znovu s `finalize: "commit"`. **Provede se 2×, jednou pro
každý článek** — element IDs se čtou znovu z `read-design` pro každou novou kopii designu, nikdy se
nesdílí mezi běhy smyčky.

Canva volání jdou přes MCP connector (`mcp__Canva__...`) — funguje v Routine bez úprav, pokud je
Canva zapnutá v seznamu connectorů dané routine.

---

## Krok 3 — Export PNG (2×)

Identické s `agro-socials-local` Krokem 3 — `mcp__Canva__export-design`, **pouze `width: 1080`**, nikdy
zároveň `height` (aspect ratio padding bug). Provede se pro každý ze 2 `[COPY_ID]`.

---

## Krok 4 — Cloudinary upload (bash/curl, 2×)

```bash
: "${CLOUDINARY_CLOUD_NAME:?chybí}"; : "${CLOUDINARY_API_KEY:?chybí}"; : "${CLOUDINARY_API_SECRET:?chybí}"

upload_to_cloudinary() {
  local png_url="$1" public_id="$2"
  local timestamp eager folder signature

  folder="SOCIALS"
  eager="w_1080,c_limit,q_auto"
  timestamp=$(date +%s)

  # Cloudinary signature = sha1(sorted params + api_secret), bez file/api_key
  signature=$(printf "eager=%s&folder=%s&overwrite=true&public_id=%s&timestamp=%s%s" \
    "$eager" "$folder" "$public_id" "$timestamp" "$CLOUDINARY_API_SECRET" | sha1sum | cut -d' ' -f1)

  curl -sS -X POST "https://api.cloudinary.com/v1_1/$CLOUDINARY_CLOUD_NAME/image/upload" \
    -F "file=$png_url" \
    -F "api_key=$CLOUDINARY_API_KEY" \
    -F "timestamp=$timestamp" \
    -F "signature=$signature" \
    -F "folder=$folder" \
    -F "public_id=$public_id" \
    -F "overwrite=true" \
    -F "eager=$eager"
}

# Volej pro každý ze 2 článků, public_id = social_[DATUM_SLUG]_[ID] (jen ASCII, bez diakritiky!)
result=$(upload_to_cloudinary "$PNG_EXPORT_URL" "social_${DATUM_SLUG}_${ID}")
cloudinary_url=$(echo "$result" | jq -r '.eager[0].secure_url // .secure_url')
echo "CLOUDINARY_URL: $cloudinary_url"
```

**Retry:** při chybě/timeoutu počkej 5 s, zkus 1× znovu. `public_id` musí být čisté ASCII
(`social_2026-06_falcon_sf700`), žádná diakritika/mezery/`·`.

Uprav podle skutečného `public_id` schématu, které používáš v `agro-socials-local` (folder `SOCIALS`).

---

## Krok 5 — Copywriting (1 IG + 1 FB text per článek, 2×)

Stejná stylistická pravidla jako v `agro-socials-local` Kroku 5 (IG: 1–3 věty, max 150 znaků, přesně 5
hashtagů; FB: 2–4 věty, max 250 znaků, 1–2 hashtagy, `[CLANEK_URL]` na konci). YouTube se přeskakuje.

> **Žádný confirmation checkpoint.** Místo čekání na potvrzení zapiš `[CLOUDINARY_URL]` + oba texty
> pro každý ze 2 článků do proměnné/pole pro finální run summary (Krok 8) a pokračuj rovnou Krokem 6.

---

## Krok 6 — Buffer: naplánuj (NE ihned publikuj) na 2 pevné časy

### Výpočet `dueAt` — DST-safe (Europe/Prague má CET/CEST)

```bash
compute_due_utc() {
  local time_str="$1"   # "07:00", "20:00"
  local now_epoch due_epoch
  now_epoch=$(date -u +%s)
  due_epoch=$(TZ="Europe/Prague" date -d "today $time_str" +%s)
  if [ "$due_epoch" -le "$now_epoch" ]; then
    due_epoch=$(TZ="Europe/Prague" date -d "tomorrow $time_str" +%s)
  fi
  date -u -d "@$due_epoch" +"%Y-%m-%dT%H:%M:%SZ"
}

due_1=$(compute_due_utc "07:00")
due_2=$(compute_due_utc "20:00")
```

> Routine musí startovat **před 07:00** místního času, jinak se slot 1 posune na zítra
> (funkce to ohlídá, ale zkontroluj schedule trigger routine — ideálně 05:00–06:00 Europe/Prague).

### Buffer JSON-RPC přes curl (get_account → list_channels → create_post ×4)

```bash
: "${BUFFER_API_KEY:?chybí}"

call_buffer() {
  local method="$1" params="$2"
  local raw
  raw=$(curl -sS -X POST "https://mcp.buffer.com/mcp" \
    -H "Authorization: Bearer $BUFFER_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Accept: text/event-stream, application/json" \
    -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"$method\",\"params\":$params}")
  # Odpověď může být čistý JSON, NEBO SSE (JSON v řádcích "data: ") — ošetři oba případy.
  if echo "$raw" | grep -q '^data: '; then
    echo "$raw" | grep '^data: ' | tail -1 | sed 's/^data: //'
  else
    echo "$raw"
  fi
}

org_id=$(call_buffer "tools/call" '{"name":"get_account","arguments":{}}' \
  | jq -r '.result.content[0].text | fromjson | .organizations[0].id')
channels=$(call_buffer "tools/call" "{\"name\":\"list_channels\",\"arguments\":{\"organizationId\":\"$org_id\"}}" \
  | jq -r '.result.content[0].text | fromjson')
ig_id=$(echo "$channels" | jq -r '.[] | select(.service=="instagram") | .id')
fb_id=$(echo "$channels" | jq -r '.[] | select(.service=="facebook") | .id')

schedule_post() {
  local channel_id="$1" text="$2" image_url="$3" alt="$4" due="$5" platform="$6"
  local meta
  if [ "$platform" = "instagram" ]; then
    meta='{"instagram":{"type":"post","shouldShareToFeed":true}}'
  else
    meta='{"facebook":{"type":"post"}}'
  fi
  local args
  args=$(jq -n --arg ch "$channel_id" --arg text "$text" --arg url "$image_url" \
    --arg alt "$alt" --arg due "$due" --argjson meta "$meta" '
    { channelId: $ch, text: $text,
      assets: [ { image: { url: $url, metadata: { altText: $alt } } } ],
      mode: "customScheduled", schedulingType: "automatic", dueAt: $due,
      metadata: $meta }')
  call_buffer "tools/call" "{\"name\":\"create_post\",\"arguments\":$args}"
}

# Pro každý ze 2 článků (cloudinary_url, texty, due) zavolej 2x — IG a FB:
schedule_post "$ig_id" "$ig_text_1" "$cloudinary_url_1" "$titulek_1" "$due_1" "instagram"
schedule_post "$fb_id" "$fb_text_1" "$cloudinary_url_1" "$titulek_1" "$due_1" "facebook"
schedule_post "$ig_id" "$ig_text_2" "$cloudinary_url_2" "$titulek_2" "$due_2" "instagram"
schedule_post "$fb_id" "$fb_text_2" "$cloudinary_url_2" "$titulek_2" "$due_2" "facebook"
```

**Klíčový rozdíl oproti `agro-socials-local`:** primární `mode` je tady **vždy `customScheduled`**
(ne `shareNow`) — chceme, aby Buffer sám vyčkal a publikoval v přesný čas, ne hned.

---

## Krok 7 — Zapiš posted-log.json a commitni

Zapiš **oba vybrané (naplánované)** i **nevybraný/é přeskočené** články z dnešní dávky — přeskočené
s `note`, aby se zítra znovu nenabízely jako kandidáti (viz Krok 1).

```bash
jq -n \
  --arg id1 "$id1" --arg url1 "$clanek_url_1" \
  --arg id2 "$id2" --arg url2 "$clanek_url_2" \
  --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '[{id: $id1, url: $url1, posted_at: $ts}, {id: $id2, url: $url2, posted_at: $ts}]' > /tmp/new_entries.json

# Pro každý nevybraný článek z /tmp/todays_batch.json přidej záznam s "note" (důvod přeskočení):
# jq -n --arg id "$skipped_id" --arg url "$skipped_url" --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
#   --arg note "nevybrán jako 1 ze 2 nejatraktivnějších z dnešní dávky" \
#   '[{id: $id, url: $url, posted_at: $ts, note: $note}]' >> ... (sluč do stejného pole)

jq -s '.[0] + .[1]' posted-log.json /tmp/new_entries.json > /tmp/merged.json
mv /tmp/merged.json posted-log.json

git add posted-log.json
git commit -m "agro-socials-cloud: posted $(date -u +%Y-%m-%d)"
git push -u origin "$(git branch --show-current)"
```

> Pokud push selže s oprávněním na `claude/`-prefixed branch, viz sekci **Předpoklady prostředí**
> výše — zapni "Allow unrestricted branch pushes" pro tohle repo v nastavení routine, jinak se
> `posted-log.json` neuloží a příští běh bude riskovat duplicity.

### Založ PR a rovnou ho sluč

Po pushnutí založ pull request (`create_pull_request`, `claude/`-prefixed větev → `main`) a jakmile
je `mergeable_state: clean`, **sluč ho hned v tomhle běhu** (`merge_pull_request`) — nenechávej ho
otevřený na ruční sloučení ani na úklid v Kroku 0 příštího běhu.

> **Proč hned:** dokud PR visí otevřený, `main` neobsahuje dnešní zápis do `posted-log.json`.
> Každá další session nebo routine, která startuje z `main`, pak vidí zastaralý stav a spoléhá
> na to, že Krok 0 příštího běhu úklid stihne. Když se mezitím `posted-log.json` na `main` změní
> jinou cestou, PR dostane konflikt, Krok 0 ho neumí vyřešit (slučuje jen `clean` PR) a zastaralý
> stav zůstane — s rizikem duplicitního postu. Okamžité sloučení tuhle závislost odstraní.

> Sluč **jen PR, který jsi právě v tomhle běhu založil** a jen když je `clean` a bez červené CI.
> Pokud sloučení selže (konflikt, chybějící oprávnění, nezelená CI), nech PR otevřený, pokračuj
> Krokem 8 a pošli notifikaci podle Kroku 9 — otevřený PR pak dojede Krok 0 příštího běhu.

---

## Krok 8 — Run summary (nahrazuje interaktivní kontrolní bod)

```
✅ Naplánováno 2 příspěvky (4 posty celkem) — [datum běhu]

1) "[TITULEK_1]" ([KATEGORIE_1]) → 07:00 Europe/Prague
   🖼️  [CLOUDINARY_URL_1]
   📸 IG: "[ig_text_1]"
   📘 FB: "[fb_text_1]"

2) "[TITULEK_2]" ([KATEGORIE_2]) → 20:00 Europe/Prague
   ...

⏭️  Přeskočeno z dnešní dávky: "[TITULEK_3]" — [důvod, např. "cover fotka neodpovídá značce"]

posted-log.json aktualizován a commitnut.
[Pokud dnešní dávka měla méně než 2 použitelné články]: ⚠️ dobráno z backlogu — zkontroluj kvalitu obsahu.
```

Run summary je **jen do transcriptu** — nikdy z něj nedělej push notifikaci. O notifikacích rozhoduje Krok 9.

---

## Krok 9 — Notifikace (POUZE při chybě)

Routine běží, když u toho nikdo není. Notifikace vytrhne uživatele z toho, co zrovna dělá, takže se
posílá jen tehdy, když s tím **musí něco udělat**.

**NEPOSÍLEJ notifikaci, když běh dopadl dobře.** Konkrétně: naplánovaly se všechny posty,
`posted-log.json` se commitnul, žádný krok neselhal → **žádná notifikace**, jen run summary
z Kroku 8 do transcriptu. Tichý běh = úspěšný běh.

**POŠLI notifikaci** (`PushNotification`, `status: "proactive"`) jen v těchto případech:

| Situace | Co napsat |
|---|---|
| Chybí env proměnná (`AI_API_KEY`, `BUFFER_API_KEY`, Cloudinary) | která chybí + že běh vůbec nezačal |
| Profifarmar API nevrátí články (HTTP chyba, prázdná odpověď) | status kód / co přišlo |
| Canva krok selže (copy, upload assetu, editace, export) | u kterého článku a v jaké fázi |
| Cloudinary upload selže i po 1 retry | který článek + chybová hláška |
| Buffer odmítne post (jakýkoli z těch, co se měly naplánovat) | kolik z kolika prošlo + chyba u těch zbylých |
| `posted-log.json` se nepodaří commitnout/pushnout | že hrozí duplicity v příštím běhu |
| **Částečný úspěch** — naplánovalo se míň postů, než mělo | co prošlo, co ne, a co je potřeba dodělat ručně |
| Nedostatek použitelných kandidátů (0 článků k postnutí) | že dnes nevyjde nic a proč |

Formát zprávy — do `<routine_summary>` tagů, první věta je banner na telefon,
zbytek je tělo e-mailu, ať se dá jednat bez otevírání session:

```
<routine_summary>
Buffer odmítl 2 ze 6 postů (IG 17:00 a 20:00) — chyba "asset URL unreachable".
Vizuály na Cloudinary jsou v pořádku, ostatní 4 posty naplánované.
Zbylé dva je potřeba naplánovat ručně, nebo pustit routine znovu.
</routine_summary>
```

> Notifikaci pošli **hned, jak na problém narazíš** — ne až na konci běhu. Zbytek dávky pak dokonči
> normálně (co jde naplánovat, naplánuj) a v notifikaci uveď, co prošlo a co ne.

---

## Troubleshooting (doplňky k `agro-socials-local`)

| Problém | Řešení |
|---|---|
| Routine nemá `curl`/`jq` | Doplň setup script routine: `apt-get update && apt-get install -y curl jq` |
| `posted-log.json` neexistuje | Založ v repu s obsahem `[]` před prvním během |
| Push na main odmítnut | Zapni "Allow unrestricted branch pushes" pro repo v nastavení routine, nebo směřuj na `claude/state` branch a mergi manuálně |
| Slot 07:00 naplánován na zítra místo dneška | Trigger routine startuje po 07:00 místního času — přesuň trigger na 05:00–06:00 Europe/Prague |
| `dueAt` posunuté o hodinu | Nepoužívej pevný `+02:00`/`+01:00` offset — vždy přes `TZ="Europe/Prague" date -d ...` (funkce `compute_due_utc`) |
| Stejný článek publikován 2× | `posted-log.json` se nekomitnul z předchozího běhu — zkontroluj Krok 7 a push oprávnění |
| **Nejnovější články chybí v `candidates`** | Endpoint bez `?limit=` vrací jen 100 záznamů řazených podle `id` (UUID), NE podle data — vždy volej s `?limit=10000` (viz Krok 1) |
| Cloudinary signature error | Signature musí sedět přesně na parametry a jejich pořadí — abecedně, bez `file`/`api_key`, viz `upload_to_cloudinary` |
| Buffer 406 Not Acceptable | Header `Accept: text/event-stream, application/json` musí být přítomný |
| Buffer odmítne `customScheduled` | Ověř, že `schedulingType: "automatic"` a `dueAt` je validní ISO8601 UTC (`Z` suffix) |
| Canva element nenalezen | Element IDs vždy z aktuální `read-design` (`open_transaction: true`) response, ne z tabulky v `agro-socials-local` |
| Cover fotka neodpovídá obsahu článku (např. jiná značka stroje) | Vyřaď z výběru v Kroku 1, zapiš do `posted-log.json` jako přeskočený s `note`, dobírej dalšího kandidáta |
