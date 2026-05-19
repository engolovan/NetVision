# AGENTS.md

## Cursor Cloud specific instructions

### Project Overview

NetVision is a video surveillance camera management platform (Russian-language UI). It is currently in early prototype stage — the main branch contains only a README.

### Codebase Structure

- `main` branch: Contains only `README.md`
- `cursor/system-list-fields-2920` branch: Static HTML/CSS/JS prototype for camera sources management UI (`index.html`)
- `cursor/thorx3-camera-stream-mapping-45eb` branch: Technical specification for Thorx3 integration (`docs/thorx3-camera-stream-mapping.md`)

### Running the Application

The application is a static HTML prototype with no build system, no backend, and no dependencies. To serve it locally:

```bash
python3 -m http.server 8080
```

Then open `http://localhost:8080/index.html` in a browser.

### Development Notes

- No package manager, no `node_modules`, no build step required.
- No linting or automated tests are configured yet.
- The HTML file uses inline CSS and JavaScript — no external dependencies.
- To work with the UI prototype, check out the `cursor/system-list-fields-2920` branch or copy `index.html` from it.
