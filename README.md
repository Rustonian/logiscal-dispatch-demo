# Logiscal Dispatch prototype

A clickable, single-file prototype of the Logiscal Route Dispatch app described in `docs/build-spec.md`. Everything is simulated in the page: there is no server, no database and no outside call other than Google Fonts.

**Live:** https://logiscal-dispatch-demo.netlify.app

## What it shows

Logiscal uploads its OpenFleet order CSV, the app merges orders into routes (routes are built in Onfleet and arrive with a route ID), and a dispatcher offers each route to one Gigee driver at a time by text. Drivers reply YES or NO on the simulated phone panel on the right. Driver records and eligibility come from Airtable; the app only reads them.

Screens: Dispatch board, Upload and review, Route detail, Driver picker, Needs attention, Messages inbox, Closeout and pay, Lanes, Drivers, Settings.

## Demo conventions

- The clock starts Fri Oct 16, 2026 at 2:40 PM and runs one demo minute per real second, so a 15 minute offer expires in 15 seconds.
- Refresh the page to reset everything.
- The role switcher in the header toggles Dispatcher, Client admin and Gigee ops.
- The guide at the top of the board lists the walkthrough steps.

## Editing

`index.html` is the whole app. The seeded scenario (drivers, lanes, three weeks of history, today's routes) is at the top of the script. Pushing to `main` redeploys the site.

Rules the demo encodes, for reference: six route statuses, one open offer per driver, 15 minute expiry, release outcomes by who initiated and the 24 hour line, reconfirm and final confirm windows, pay math with the route fee, the reliability score, and lane recognition by shared stops with last-driver suggestion. See the spec for the full definitions.
