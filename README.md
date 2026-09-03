# atlas-deal-analyzer

Single-file deal analyzer (`index.html`) for the Atlas Home Buyers acquisitions team. It prices a property five ways — Wholesale, Novation, BRRRR, Rental and Creative Finance — from one shared property bar, renders a printable report, and can send a summary to Slack `#underwriting` through a Make.com webhook. The Atlas CRM Buttons Chrome extension opens it from a GoHighLevel (GHL) contact page with the property pre-filled via the query parameters below.

## GHL deep-link parameters

The extension builds `https://analyzer.atlashomebuyers.com/?<params>`. Every key is optional; unknown keys are ignored and blank or non-numeric values leave the analyzer field empty (placeholder shown). Numeric values must be plain numbers (e.g. `arv=250000`, `rate=6.5`). The analyzer applies `parseFloat` as-is: `250,000` is read as 250 and `$180,000` is dropped (field left blank). The extension extracts the leading number (dropping `$`, `,` and `%` and expanding `150k` to 150000) before building the link, so links it generates are always clean. The page always opens in Wholesale mode. The **Sent by extension** column marks the keys the current CRM Buttons extension actually puts in the link; `asking`, `purchase` and `monthly_rent` are legacy keys the analyzer still accepts (hand-built links, older extension builds) but the extension no longer sends.

| Query key | Analyzer field(s) | Sent by extension | Notes |
| --- | --- | --- | --- |
| `address` | `address` (property bar) | yes | Full street address. |
| `contact_name` | `sellerName`, GHL banner | yes | Seller / contact name. |
| `contact_id` | GHL banner, Slack payload `ghl.contact_id` | yes | GHL contact id, echoed back in the Slack payload. |
| `beds` | `beds` | yes | Number. |
| `baths` | `baths` | yes | Number (decimals allowed). |
| `sqft` | `sqft` | yes | Square feet; drives the $/sf rehab lines. |
| `year` | `yearBuilt` | yes | Integer year. |
| `arv` | `arv` | yes | After Repair Value. |
| `as_is` | `asIsValue` | yes | As-Is value; Rental/BRRRR purchase price defaults from it. |
| `mortgage` | `mortgageBalance`, `cfSubAmount` | yes | Mortgage balance; also seeds the Creative SubTo balance. |
| `arrears` | `cfArrearsLiens` | yes | Arrears / liens. Part of the seller payoff (see below). |
| `bottom_dollar` | `sellerNet` | yes | **Seller Bottom Dollar** — the seller's required *net* proceeds after the mortgage and liens are paid off. Not a purchase price. |
| `asking` | `sellerNet` | no — legacy | Legacy alias for `bottom_dollar` (ignored when `bottom_dollar` is also present). |
| `purchase` | `purchasePrice` (+ `purchasePriceTouched`) | no — legacy | Negotiated price; Rental/BRRRR keep it instead of deriving from As-Is. |
| `monthly_rent` | `monthlyRent` | no — legacy | Market rent per month. |
| `market_rent` | `monthlyRent` | yes | Same field as `monthly_rent`. |
| `rate` | `cfSubRate` | yes | Existing mortgage interest rate (%) for Creative SubTo. |
| `monthly_pi` | `cfSubPI` | yes | Existing mortgage principal + interest per month. |
| `taxes` | `cfTaxesMonthly`, `propertyTaxAnnual` (= monthly × 12) | yes | Monthly property taxes. |
| `insurance` | `cfInsuranceMonthly`, `insuranceAnnual` (= monthly × 12) | yes | Monthly insurance. |
| `hoa` | `cfHoaMonthly`, `hoaMonthly` | yes | Monthly HOA dues. |

### Seller payoff and bottom dollar

`payoff = mortgageBalance + cfArrearsLiens`. "Net to seller" is always `price − payoff`; the offer callout, the report's Property Snapshot, the underwriting flags and the Slack payload all use it. In Wholesale and Novation the bottom dollar is compared to the net at MAO: a danger flag when the MAO leaves the seller short (with whether the +$5,000 ceiling would cover it) or an informational chip when it is covered. In Creative Finance the bottom dollar is the default for **Cash to Seller** (when that field is still 0) and a chip warns when Cash to Seller is below it.

## Share links (`#d=` hash)

Every edit is debounced into `location.hash` as `#d=<base64url JSON>` and into `localStorage`. The hash stores only the form keys that differ from the defaults, the rehab line overrides, the mode, the step, whether the deal started from GHL (`ghl: true`, so a reload keeps the user's edits instead of re-applying the stale query) and the current view (`form` / `report`). **Copy Link** flushes the pending save and copies `location.href`; opening that link restores the exact deal (`decodeState` → `applyState`, with every value coerced to the type of its default).

## Send to Slack `#underwriting` (Make.com webhook)

`saveDealToGhl` first flushes the save so the hash is current, then POSTs JSON to the Make.com webhook. Top-level fields: `version`, `source`, `timestamp`, `dedup_key` (sha1 of address + mode + date), **`share_url`** (the page URL including the `#d=` hash — opens this exact deal), `ghl` (`contact_id`, `contact_name`, `prefilled`), `property` (`address`, `beds`, `baths`, `sqft`, `year_built`, `arv`, `as_is`, `mortgage_balance`, **`seller_bottom_dollar`**, **`arrears_liens`**, **`payoff`**, `seller_name`), `exit` (mode), `headline` (`primary_metric_label`, `primary_metric_value`, `grade`, mode-specific `secondary` — for Wholesale/Novation `net_to_seller` is MAO − payoff, floored at 0) and `full_calc` (the complete calculation, including `brrrr` and `creative` sub-objects when applicable).

## Development

No build step — open `index.html` directly. Headless checks used during development: `node --check` on the extracted `<script>` body, a canonical-deal collector that must stay byte-identical for deals that do not use new fields, and a sweep of every step of every mode plus the reports for `null` / `NaN` / `undefined` / `$-` and page errors.
