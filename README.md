# Logiscal Dispatch prototype

A clickable, single-file prototype of the Logiscal Route Dispatch app described in `docs/build-spec.md`. Everything is simulated in the page: there is no server, no database and no outside call other than Google Fonts.

**Live:** https://logiscal-dispatch-demo.netlify.app

## What it shows

Logiscal uploads its Onfleet task export (the real column layout: each Driver Number is one route, hub rows are skipped), the app prices every route from its market's parameters, recognizes routes it has seen before by their stops, and a dispatcher offers each route to one Gigee driver at a time by text. Drivers reply YES or NO on the simulated phone panel on the right. Driver records and eligibility come from Airtable; the app only reads them.

Screens: Dispatch board (with Staffing health), Site check-in, Upload and review (drop a real Onfleet export or use the scrubbed Indianapolis sample), Onfleet check, Markets and pricing, Route detail, Driver picker, Needs attention, Messages inbox, Closeout and pay, Lanes, Drivers, Settings.

## Demo conventions

- The clock starts Tue Nov 17, 2026 at 2:40 PM and runs one demo minute per real second, so a 15 minute offer expires in 15 seconds.
- Refresh the page to reset everything.
- The role switcher in the header toggles Dispatcher, Client admin, Regional Lead and Gigee ops. Pricing access (edit, view, none) is set per user on Markets and pricing.
- The guide at the top of the board lists the walkthrough steps.

## Editing

`index.html` is the whole app. The seeded scenario (drivers, lanes, three weeks of Austin history, two Fridays of Indianapolis history (Nov 6 and 13), today's routes) is at the top of the script. The Indianapolis sample file is embedded in a `sampleIndy` script block; it is a scrubbed copy of the Oct 15 CookUnity export with its dates moved to Fri Nov 20 (made-up house numbers, customers and drivers). Dropping that same file on Upload and review plays exactly like the sample button. Seed dates are written as October day numbers and shifted by `SHIFT` at the top of the script. Pushing to `main` redeploys the site.

Rules the demo encodes, for reference: six route statuses, one open offer per driver, 15 minute expiry, release outcomes by who initiated and the 24 hour line, reconfirm and final confirm windows, pay math with the route fee, the reliability score, lane recognition by shared stops with last-driver suggestion, offer texts that show a stop range instead of an exact count, performance holds (suggested after a no-show or late bail), site check-in where a no-show starts a same-day rescue, staffing health against a 20% bench, market pricing by effort (base per drop, B2B and apartment extras, miles, first leg, hourly floor, glide on per-stop pay, margin target), pay locked once texted, a confirm text to the driver Onfleet already has on a route, and the Onfleet check that compares Onfleet's driver with the app's. See the spec for the full definitions.
