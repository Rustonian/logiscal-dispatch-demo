# Logiscal Dispatch prototype

A clickable, single-file prototype of the Logiscal Route Dispatch app described in `docs/build-spec.md`. Everything is simulated in the page: there is no server, no database and no outside call other than Google Fonts.

**Live:** https://logiscal-dispatch-demo.netlify.app

## What it shows

Logiscal uploads its Onfleet task export (the real column layout: each Driver Number is one route, hub rows are skipped), the app prices every route from its market's parameters, recognizes routes it has seen before by their stops, and a dispatcher offers each route to one Gigee driver at a time by text. Drivers reply YES or NO on the simulated phone panel on the right. Driver records and eligibility come from Airtable; the app only reads them.

Screens: Dispatch board, Coverage (routes vs active drivers plus bench, per area and day), Site check-in, Upload routes (drop a real Onfleet export), Needs attention, Daily closeout, Markets and pricing (admins only; pick a market from a dropdown, or the All markets overview table that flags values far from the median or wrong on their own), Route detail (with Adjust route), Driver picker, Lanes, Drivers, Settings.

## Demo conventions

- The app starts empty: no routes, drivers, lanes or history. Markets, hubs, settings and users are kept. Upload an Onfleet task export to fill it. Drivers on the file are added from the upload preview, and a hub the app doesn't know (new facilityId) can be added from the preview too, under a new market with pricing copied for now.
- The clock starts at the real current time and runs one demo minute per real second, so a 15 minute offer expires in 15 seconds.
- Board tabs: Earlier (any route dated before today, shown only when there is one), Today, Tomorrow, the next two days by weekday, and Later. Earlier and Later cards show the route date.
- Refresh the page to reset everything.
- The role switcher in the header toggles Dispatcher, Client admin and Gigee ops. Markets and pricing is visible to admins only (client admin and Gigee ops).

## Editing

`index.html` is the whole app. Sample data is switched off by `const BLANK = true` in the script; set it to `false` to bring back the old seeded scenario and walkthrough guide. The seeded scenario (drivers, lanes, three weeks of Austin history, two Fridays of Indianapolis history (Nov 6 and 13), today's routes) is at the top of the script. The Indianapolis sample file is embedded in a `sampleIndy` script block; it is a scrubbed copy of the Oct 15 CookUnity export with its dates moved to Fri Nov 20 (made-up house numbers, customers and drivers). Dropping that same file on Upload routes plays exactly like the sample button. `samples/onfleet_export_2026-11-19_multi_market.csv` is a made-up three-market file for Thu Nov 19: Austin (Round Rock facility 1, Buda facility 2), Indianapolis (Greenwood facility 4) and Columbus (Obetz facility 7), 16 routes and 343 orders, three of them unassigned in Onfleet. Seed dates are written as October day numbers and shifted by `SHIFT` at the top of the script. Pushing to `main` redeploys the site.

Rules the demo encodes, for reference: six route statuses, one open offer per driver, 15 minute expiry, release outcomes by who initiated and the 24 hour line, reconfirm and final confirm windows, pay math with the route fee, the reliability score, lane recognition by shared stops with last-driver suggestion, offer texts that show a stop range instead of an exact count, site check-in where a no-show starts a same-day rescue, coverage against a 20% bench, market pricing by effort (base per drop, B2B and apartment extras, miles, first leg, hourly floor, glide on per-stop pay, margin target), pay locked once texted, a confirm text to the driver Onfleet already has on a route, uploads that only add routes not already in the app (a repeat file imports nothing), Adjust route for stops and pay changed in Onfleet afterward, and a daily closeout where a dispatcher signs off each route as ready to pay. See the spec for the full definitions.
