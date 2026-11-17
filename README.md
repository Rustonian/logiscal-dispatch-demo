# Logiscal Dispatch prototype

A clickable, single-file prototype of the Logiscal Route Dispatch app described in `docs/build-spec.md`. Everything is simulated in the page: there is no server, no database and no outside call other than Google Fonts.

**Live:** https://logiscal-dispatch-demo.netlify.app

## What it shows

Logiscal uploads its Onfleet task export (the real column layout: each Driver Number is one route, hub rows are skipped), the app prices every route from its market's parameters, recognizes routes it has seen before by their stops, and a dispatcher offers each route to one Gigee driver at a time by text. Drivers reply YES or NO on the simulated phone panel on the right. Driver records and eligibility come from Airtable; the app only reads them.

Screens: Dispatch board (with Staffing health), Site check-in, Upload and review (drop a real Onfleet export or use the scrubbed Indianapolis sample), Needs attention, Daily closeout, Markets and pricing (admins only), Route detail (with Adjust route), Driver picker, Lanes, Drivers, Settings.

## Demo conventions

- The clock starts Tue Nov 17, 2026 at 2:40 PM and runs one demo minute per real second, so a 15 minute offer expires in 15 seconds.
- Refresh the page to reset everything.
- The role switcher in the header toggles Dispatcher, Client admin and Gigee ops. Markets and pricing is visible to admins only (client admin and Gigee ops).
- The guide at the top of the board lists the walkthrough steps.
- Blank slate: open the page with `?blank`, or use "Start with a blank slate" in the guide or "Clear sample data" in Settings. It drops every sample route, driver, lane, offer, text and import, keeps markets, hubs, settings and users, and starts the clock at the real time. It stays blank across refreshes in that browser until you click "Restore sample data". Drivers on your file are added from the upload preview, and a hub the app doesn't know (new facilityId) can be added from the preview too, under a new market with pricing copied for now.

## Editing

`index.html` is the whole app. The seeded scenario (drivers, lanes, three weeks of Austin history, two Fridays of Indianapolis history (Nov 6 and 13), today's routes) is at the top of the script. The Indianapolis sample file is embedded in a `sampleIndy` script block; it is a scrubbed copy of the Oct 15 CookUnity export with its dates moved to Fri Nov 20 (made-up house numbers, customers and drivers). Dropping that same file on Upload and review plays exactly like the sample button. Seed dates are written as October day numbers and shifted by `SHIFT` at the top of the script. Pushing to `main` redeploys the site.

Rules the demo encodes, for reference: six route statuses, one open offer per driver, 15 minute expiry, release outcomes by who initiated and the 24 hour line, reconfirm and final confirm windows, pay math with the route fee, the reliability score, lane recognition by shared stops with last-driver suggestion, offer texts that show a stop range instead of an exact count, site check-in where a no-show starts a same-day rescue, staffing health against a 20% bench, market pricing by effort (base per drop, B2B and apartment extras, miles, first leg, hourly floor, glide on per-stop pay, margin target), pay locked once texted, a confirm text to the driver Onfleet already has on a route, uploads that only add routes not already in the app (a repeat file imports nothing), Adjust route for stops and pay changed in Onfleet afterward, and a daily closeout where a dispatcher signs off each route as ready to pay. See the spec for the full definitions.
