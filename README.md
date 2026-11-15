# topo-demos — Crypto Risk Guard and Climate CII demo panels

Two self-contained, static proof-of-concept panels built on the same topological
signal-processing toolkit. No server, no build step: open the HTML files in a
browser. These are demonstrations of method, not products with predictive claims.

| Demo | Folder | What it shows |
|---|---|---|
| Crypto Risk Guard | [`crypto/index.html`](crypto/index.html) | Topological risk index for BTC / ETH / SOL early-warning panels |
| Climate CII | [`climate/index.html`](climate/index.html) | Climate Impact Index for regions (Europe, India, Australia, Texas) |

See [`index.html`](index.html) for a landing page linking both demos.

## Quick start

```bash
# Option 1: open the file directly
xdg-open index.html

# Option 2: serve locally (recommended for JSON loading)
python3 -m http.server 8000
# http://localhost:8000
```

Each folder contains `README.md`, `Ограничения_и_честность.md` (limitations and
honesty notes), figures, example CSV/JSON inputs and a demo report.

## Data

All inputs are small example series committed for demonstration. Crypto panels
use daily OHLC-derived risk series (`risk_BTC.csv`, `risk_ETH.csv`,
`risk_SOL.csv`); climate panels use regional CII series and summaries. They are
illustrative snapshots, not live feeds.

## Limitations

- Backtests are in-sample demonstrations; no transaction costs, slippage or
  live execution.
- Climate index weights are illustrative and not validated for insurance or
  regulatory use.
- Not investment advice. Not a climate-risk certification.
- The panels are intentionally kept out of the pinned repositories; this repo is
  a method showcase.

## License

MIT — see [LICENSE](LICENSE). Individual demo limitations are in each folder.
