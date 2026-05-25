# CLAUDE.md

## Project Overview

Personal race results tracking website for Jake Walker. A static single-page application displaying running race results, personal bests, statistics, and upcoming events. Deployed via GitHub Pages to `raceresults.jakewalker.com`.

## Project Structure

```
index.html    - SPA (HTML + embedded JavaScript)
styles.css    - All CSS styles (linked from index.html)
races.json    - Race data (JSON array of race objects)
CNAME         - GitHub Pages custom domain config
.nojekyll     - Disables Jekyll processing on GitHub Pages
```

## Development

No build system, dependencies, or package manager. The site is purely static.

**Local development:** serve the directory with any static HTTP server:
```bash
python3 -m http.server 8000
```

**Deployment:** push to `main` branch on GitHub — GitHub Pages deploys automatically.

## Race Data Format (races.json)

Each race object:
```json
{
  "name": "Race Name",
  "date": "YYYY-MM-DD",
  "distanceKm": 21.1,
  "time": "H:MM:SS",
  "raceWebsite": "https://...",
  "resultsWebsite": "https://...",
  "stravaActivity": "https://..."
}
```

- Provide either `distanceKm` or `distanceMi` — the other is computed automatically.
- `paceKm`, `paceMi`, and `personalBest` are computed at load time; do not include them. PB is the fastest `time` at each `distanceKm`; ties go to the earliest `date`.
- Set `"excludeFromPB": true` to disqualify a race from PB consideration (e.g. short course, irregular conditions). Omit otherwise.
- Upcoming races omit `time` (or leave it as `""`).
- URLs can be empty strings if not available.

## Code Conventions

- **JavaScript:** camelCase functions/variables, `const`/`let` (no `var`), semicolons required, template literals for string interpolation, arrow functions for callbacks
- **CSS:** CSS custom properties (variables) defined on `:root`, kebab-case class names (e.g., `.pb-card`, `.filter-chip`), mobile breakpoints at 768px and 480px
- **HTML:** semantic HTML5 elements, `data-*` attributes for configuration, kebab-case class names
- CSS lives in `styles.css`, JS is embedded in `index.html`
