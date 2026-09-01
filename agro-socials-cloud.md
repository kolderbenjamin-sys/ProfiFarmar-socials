---
name: agro-socials-cloud
description: "Cloud/Routine varianta agro-socials-local skillu. Jedním během (bez interaktivního uživatele) vybere 3 nepostnuté publikované články z Profifarmar.cz, pro každý vytvoří Canva vizuál, nahraje na Cloudinary a naplánuje přes Buffer na 3 pevné časy (07:00 / 17:00 / 20:00 Europe/Prague) — Buffer je pak sám releasne postupně. Určeno pro Claude Code cloud Routine (Linux, bash/curl/jq, žádný interaktivní checkpoint). Trigger keywords: agro social cloud, routine social, cloud buffer schedule, naplánuj 3 příspěvky, agro social routine."
---

# Agro-Socials-Cloud Skill

Cloud/Routine verze `agro-socials-local`. Běží **jednou denně, autonomně**, bez čekání na potvrzení uživatele.
V jednom běhu:

1. vybere **3 nepostnuté** publikované články,
2. pro každý vytvoří Canva vizuál → Cloudinary URL,
3. napíše IG/FB texty,
4. naplánuje všech 6 postů (3× IG + 3× FB) přes Buffer na `customScheduled` s pevným `dueAt`
   **07:00 / 17:00 / 20:00 Europe/Prague** — Buffer je pak releasne sám, i když routine dávno doběhla.
5. zapíše publikované články do `posted-log.json` a commitne do repa (ochrana proti duplicitám).

> **Notifikace jen při chybě.** Když běh proběhne v pořádku, **neposílej žádnou push notifikaci** —
> run summary v transcriptu stačí. Notifikace se posílá **výhradně** při problému (viz Krok 9).

> **Bez interaktivního checkpointu.** Na rozdíl od `agro-socials-local` tahle verze **nečeká na potvrzení** —
> v Routine není nikdo, kdo by potvrdil. Místo toho se vizuály + texty zapíšou do run summary
> (Krok 8) pro zpětnou kontrolu po doběhnutí.

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
  článků, aby se stejný článek neopakoval.
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
> vybrat už dřív publikovaný článek. Automatické sloučení na začátku běhu tohle riziko
> eliminuje bez nutnosti manuálního zásahu uživatele.

---

## Krok 1 — Vyber 3 nepostnuté články (bash/curl/jq)

```bash
set -euo pipefail

: "${AI_API_KEY:?AI_API_KEY chybí — nastav v Environment routine}"

# Bez ?limit vrací API tiše jen 100 záznamů, a ne spolehlivě těch nejnovějších — vždy dotahuj vše.
response=$(curl -sS -H "Authorization: Bearer $AI_API_KEY" \
  "https://profifarmar.cz/api/webhook.php?limit=10000")

# posted-log.json = [{ "id": 123, "url": "...", "posted_at": "2026-06-30" }, ...]
posted_ids=$(jq '[.[].id]' posted-log.json 2>/dev/null || echo "[]")

candidates=$(echo "$response" | jq --argjson posted "$posted_ids" '
  (.data // .)
  | map(select(.status == "published" and .cover_image_url != null))
  | map(select(.id as $id | ($posted | index($id)) == null))
  | sort_by(.published_at) | reverse
')

echo "$candidates" > /tmp/candidates.json
echo "Nalezeno $(echo "$candidates" | jq length) nepostnutých kandidátů."
```

**Výběr 3 z `candidates`:**
- Vezmi top ~10 z `/tmp/candidates.json` (nejnovější první).
- Vyber **3**, které dávají smysl **i s odstupem** (routine nemusí běžet den co den) — přeskoč
  články se **sezónním konfliktem** (např. článek o suchu v červnu nepostuj v září) i pokud jsou
  novější než jiný bezpečný kandidát.
- Pokud kandidátů je méně než 3, publikuj kolik jde a do run summary (Krok 8) napiš, kolik chybí.
- Ulož vybrané jako `$article1`, `$article2`, `$article3` (JSON objekty) — z každého vytáhni
  `[TITULEK]`, `[KATEGORIE]`, `[DATUM]`, `[DATUM_SLUG]`, `[COVER_URL]`, `[ID]`, `[CLANEK_URL]`
  stejně jako v `agro-socials-local` Kroku 1.

---

## Krok 2 — Pro KAŽDÝ ze 3 článků: naplň Canva šablonu (MCP)

Identické s `agro-socials-local` Krokem 2 (šablona `DAHOBdpJ1tk` — ProfiFarmář 4:5 Stacked, `copy-design` →
`upload-asset-from-url` → `start-editing-transaction` → `perform-editing-operations` →
`commit-editing-transaction`). **Provede se 3×, jednou pro každý článek** — element IDs se čtou
znovu z `start-editing-transaction` pro každou novou kopii designu, nikdy se nesdílí mezi běhy smyčky.

Canva volání jdou přes MCP connector (`mcp__canva__...`) — funguje v Routine bez úprav, pokud je
Canva zapnutá v seznamu connectorů dané routine.

---

## Krok 3 — Export PNG (3×)

Identické s `agro-socials-local` Krokem 3 — `mcp__canva__export-design`, **pouze `width: 1080`**, nikdy
zároveň `height` (aspect ratio padding bug). Provede se pro každý ze 3 `[COPY_ID]`.

---

## Krok 4 — Cloudinary upload (bash/curl, 3×)

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

# Volej pro každý ze 3 článků, public_id = social_[DATUM_SLUG]_[ID] (jen ASCII, bez diakritiky!)
result=$(upload_to_cloudinary "$PNG_EXPORT_URL" "social_${DATUM_SLUG}_${ID}")
cloudinary_url=$(echo "$result" | jq -r '.eager[0].secure_url // .secure_url')
echo "CLOUDINARY_URL: $cloudinary_url"
```

**Retry:** při chybě/timeoutu počkej 5 s, zkus 1× znovu. `public_id` musí být čisté ASCII
(`social_2026-06_falcon_sf700`), žádná diakritika/mezery/`·`.

Uprav podle skutečného `public_id` schématu, které používáš v `agro-socials-local` (folder `SOCIALS`).

---

## Krok 5 — Copywriting (1 IG + 1 FB text per článek, 3×)

Stejná stylistická pravidla jako v `agro-socials-local` Kroku 5 (IG: 1–3 věty, max 150 znaků, přesně 5
hashtagů; FB: 2–4 věty, max 250 znaků, 1–2 hashtagy, `[CLANEK_URL]` na konci). YouTube se přeskakuje.

> **Žádný confirmation checkpoint.** Místo čekání na potvrzení zapiš `[CLOUDINARY_URL]` + oba texty
> pro každý ze 3 článků do proměnné/pole pro finální run summary (Krok 8) a pokračuj rovnou Krokem 6.

---

## Krok 6 — Buffer: naplánuj (NE ihned publikuj) na 3 pevné časy

### Výpočet `dueAt` — DST-safe (Europe/Prague má CET/CEST)

```bash
compute_due_utc() {
  local time_str="$1"   # "07:00", "17:00", "20:00"
  local now_epoch due_epoch
  now_epoch=$(date -u +%s)
  due_epoch=$(TZ="Europe/Prague" date -d "today $time_str" +%s)
  if [ "$due_epoch" -le "$now_epoch" ]; then
    due_epoch=$(TZ="Europe/Prague" date -d "tomorrow $time_str" +%s)
  fi
  date -u -d "@$due_epoch" +"%Y-%m-%dT%H:%M:%SZ"
}

due_1=$(compute_due_utc "07:00")
due_2=$(compute_due_utc "17:00")
due_3=$(compute_due_utc "20:00")
```

> Routine musí startovat **před 07:00** místního času, jinak se slot 1 posune na zítra
> (funkce to ohlídá, ale zkontroluj schedule trigger routine — ideálně 05:00–06:00 Europe/Prague).

### Buffer JSON-RPC přes curl (get_account → list_channels → create_post ×6)

```bash
: "${BUFFER_API_KEY:?chybí}"

call_buffer() {
  local method="$1" params="$2"
  curl -sS -X POST "https://mcp.buffer.com/mcp" \
    -H "Authorization: Bearer $BUFFER_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Accept: text/event-stream, application/json" \
    -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"$method\",\"params\":$params}" \
  | grep '^data: ' | tail -1 | sed 's/^data: //'
}

org_id=$(call_buffer "tools/call" '{"name":"get_account","arguments":{}}' | jq -r '.organizations[0].id')
channels=$(call_buffer "tools/call" "{\"name\":\"list_channels\",\"arguments\":{\"organizationId\":\"$org_id\"}}")
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

# Pro každý ze 3 článků (cloudinary_url, texty, due) zavolej 2x — IG a FB:
schedule_post "$ig_id" "$ig_text_1" "$cloudinary_url_1" "$titulek_1" "$due_1" "instagram"
schedule_post "$fb_id" "$fb_text_1" "$cloudinary_url_1" "$titulek_1" "$due_1" "facebook"
schedule_post "$ig_id" "$ig_text_2" "$cloudinary_url_2" "$titulek_2" "$due_2" "instagram"
schedule_post "$fb_id" "$fb_text_2" "$cloudinary_url_2" "$titulek_2" "$due_2" "facebook"
schedule_post "$ig_id" "$ig_text_3" "$cloudinary_url_3" "$titulek_3" "$due_3" "instagram"
schedule_post "$fb_id" "$fb_text_3" "$cloudinary_url_3" "$titulek_3" "$due_3" "facebook"
```

**Klíčový rozdíl oproti `agro-socials-local`:** primární `mode` je tady **vždy `customScheduled`**
(ne `shareNow`) — chceme, aby Buffer sám vyčkal a publikoval v přesný čas, ne hned.

---

## Krok 7 — Zapiš posted-log.json a commitni

```bash
jq -n \
  --arg id1 "$id1" --arg url1 "$clanek_url_1" \
  --arg id2 "$id2" --arg url2 "$clanek_url_2" \
  --arg id3 "$id3" --arg url3 "$clanek_url_3" \
  --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '[$id1,$id2,$id3] | to_entries' > /tmp/new_entries.json
# (uprav dle skutečné struktury — přidej { id, url, posted_at: $ts } pro každý)

jq -s '.[0] + .[1]' posted-log.json /tmp/new_entries.json > /tmp/merged.json
mv /tmp/merged.json posted-log.json

git add posted-log.json
git commit -m "agro-socials-cloud: posted $(date -u +%Y-%m-%d)"
git push
```

> Pokud push selže s oprávněním na `claude/`-prefixed branch, viz sekci **Předpoklady prostředí**
> výše — zapni "Allow unrestricted branch pushes" pro tohle repo v nastavení routine, jinak se
> `posted-log.json` neuloží a příští běh bude riskovat duplicity.

---

## Krok 8 — Run summary (nahrazuje interaktivní kontrolní bod)

```
✅ Naplánováno 3 příspěvky (6 postů celkem) — [datum běhu]

1) "[TITULEK_1]" ([KATEGORIE_1]) → 07:00 Europe/Prague
   🖼️  [CLOUDINARY_URL_1]
   📸 IG: "[ig_text_1]"
   📘 FB: "[fb_text_1]"

2) "[TITULEK_2]" ([KATEGORIE_2]) → 17:00 Europe/Prague
   ...

3) "[TITULEK_3]" ([KATEGORIE_3]) → 20:00 Europe/Prague
   ...

posted-log.json aktualizován a commitnut.
[Pokud méně než 3 kandidáti]: ⚠️ pouze N/3 článků publikováno — nedostatek nepostnutých kandidátů.
```

Run summary je **jen do transcriptu** — nikdy z něj nedělej push notifikaci. O notifikacích rozhoduje Krok 9.

---

## Krok 9 — Notifikace (POUZE při chybě)

Routine běží, když u toho nikdo není. Notifikace vytrhne uživatele z toho, co zrovna dělá, takže se
posílá jen tehdy, když s tím **musí něco udělat**.

**NEPOSÍLEJ notifikaci, když běh dopadl dobře.** Konkrétně: naplánovalo se všech 6 postů,
`posted-log.json` se commitnul, žádný krok neselhal → **žádná notifikace**, jen run summary
z Kroku 8 do transcriptu. Tichý běh = úspěšný běh.

**POŠLI notifikaci** (`PushNotification`, `status: "proactive"`) jen v těchto případech:

| Situace | Co napsat |
|---|---|
| Chybí env proměnná (`AI_API_KEY`, `BUFFER_API_KEY`, Cloudinary) | která chybí + že běh vůbec nezačal |
| Profifarmar API nevrátí články (HTTP chyba, prázdná odpověď) | status kód / co přišlo |
| Canva krok selže (copy, upload assetu, editace, export) | u kterého článku a v jaké fázi |
| Cloudinary upload selže i po 1 retry | který článek + chybová hláška |
| Buffer odmítne post (jakýkoli z těch, co se měly naplánovat) | kolik z 6 prošlo + chyba u těch zbylých |
| `posted-log.json` se nepodaří commitnout/pushnout | že hrozí duplicity v příštím běhu |
| **Částečný úspěch** — naplánovalo se míň než 6 postů | co prošlo, co ne, a co je potřeba dodělat ručně |
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
| Cloudinary signature error | Signature musí sedět přesně na parametry a jejich pořadí — abecedně, bez `file`/`api_key`, viz `upload_to_cloudinary` |
| Buffer 406 Not Acceptable | Header `Accept: text/event-stream, application/json` musí být přítomný |
| Buffer odmítne `customScheduled` | Ověř, že `schedulingType: "automatic"` a `dueAt` je validní ISO8601 UTC (`Z` suffix) |
| Canva element nenalezen | Element IDs vždy z aktuální `start-editing-transaction` response, ne z tabulky v `agro-socials-local` |
