# Sylcon Villa Traffic

An installable web app (PWA) for daily drives from Sylcon Villa, Segun Bagicha.

- **Routes:** home to and from Springdale International School, the airport, Galito's Dhanmondi, DNCC Market Gulshan 1 and DGHS. Each route shows the total drive time, the time and traffic level through the **Moghbazar flyover** section and the **Elevated Expressway Tejgaon entrance**, a leave-now-or-wait verdict and a 2-hour outlook.
- **Points:** the two crossings by themselves, in 5-minute steps, with level of service, queue, throughput, a day replay and a copyable report.
- **Setup:** Google key, API balance, pickup schedule, usual trips, history, places.

Also in the app:

- **Usual trips.** Around your usual start times the app opens on that trip: home to Springdale at 7:30 AM and Springdale to home at 10:45 AM (Sunday to Thursday), home to the airport at 7:15 PM. The window is 75 minutes before to 45 minutes after. Times and on/off are in Setup.
- **Arrive by.** Under the route, enter when you need to be there. The app gives the latest time to leave, keeping 5 minutes spare. Home to Springdale starts with 8:00 AM.
- **My history.** Every Google reading is saved on the phone by route, day type (Sun to Thu, Friday, Saturday) and quarter hour. Once a route has 3 or more readings around a time of day, its estimate for that time comes from them; at 6 or more it relies on them fully.
- **Share ETA.** Opens the phone's share sheet, or WhatsApp, with "Leaving ... now. Arriving at ... about ...".
- **Home-screen shortcuts.** Long-press the app icon for Home to Springdale, Springdale to home, Home to Airport and Home to DGHS (Android, after installing).
- **Rain.** Automatic from Open-Meteo, a free weather service that needs no key. Rain raises the estimates; Google's live times already include it. Setup has Automatic, Raining and Dry.
- **Live traffic map.** In Points, "Load the map" shows Google's traffic colours around Moghbazar, Tejgaon and Hatirjheel. It loads only on tap.

**Without a Google key every figure is an estimate** from a typical Dhaka pattern for Sun to Thu, Friday and Saturday. With a key, route times and the two crossing times come from Google.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The whole app (HTML, CSS and JavaScript in one file) |
| `manifest.webmanifest` | App name, colours and icons |
| `sw.js` | Service worker for offline use |
| `icons/` | App icons |
| `.github/workflows/pages.yml` | Publishes the site to GitHub Pages on every push to `main` |

## How the Google data works

One request to the Google Routes API (Compute Routes, traffic-aware) covers one route.

1. **Learning a route (first pickup only).** The app asks Google for the route with its line on the map. It finds where that line passes the Holy Family Hospital end and the Satrasta end of the Moghbazar flyover, and the Tejgaon toll plaza of the expressway, and saves those points as checkpoints. If Google's route does not pass them, it asks once more, insisting on the flyover and the toll plaza. This costs 1 or 2 extra requests per route and direction, once.
2. **Every pickup after that.** The app sends the saved checkpoints as stops on the way. Google returns the time for each stretch, with and without traffic, so the flyover section and the expressway entrance get their own real times. Still one request.
3. The same reading corrects the Points estimates and scales the route's 2-hour outlook.

Things to know:

- Checkpoints force the route through the flyover section and the expressway entrance, because that is the way you drive. Switch a checkpoint off with the chips under the route ("Flyover", "Expressway entrance") if you do not use it on that route.
- The flyover section is measured from near Holy Family Hospital to Satrasta. Google decides whether to go over the flyover or under it; the card says "over the flyover" when Google's directions name the flyover.
- **Trips from home** time the flyover section and the Tejgaon entrance.
- **Trips to home** that use the expressway always leave it at the **Hatirjheel exit**. The app insists on that exit even when Google would pick another way, times the exit ramp (from 600 m before it to 250 m after it), and then times the Moghbazar crossing from the exit to the Holy Family end if the way home passes it. Trips to home without the expressway time the flyover section in the home direction.
- "Open in Maps" on a route opens Google Maps with the same checkpoints, so you can see exactly what is being measured. If a route looks wrong, use "Relearn all routes" in Setup.

### Pickup schedule and the free allowance

Automatic pickups happen only while the app is open and in front, for the route on screen:

| Dhaka time | Pickup |
| --- | --- |
| 7 to 8 AM | every 5 minutes |
| 8 AM to 7 PM | every 15 minutes |
| 7 to 8 PM | every 5 minutes |
| 8 PM to 7 AM | none (the Refresh button still works and uses 1 request) |

That is at most 68 automatic requests a day, about 2,100 a month if the app is open the whole window every day. Switching to a route whose reading is older than the current interval also makes one request.

Without the shared balance, each phone has its own monthly share, 2,500 by default (Setup, "This phone's share"). Two phones at 2,500 each stay inside 5,000. The app stops asking Google when the share is used up, and the balance is shown in the top bar and in Setup. Google's own total for the key is in Google Cloud Console.

### One shared balance for both phones

Both phones can add to one counter kept in a free Firebase Realtime Database. The balance in the top bar and in Setup then shows what is left across all phones, out of a shared monthly limit (4,900 by default, which leaves a small margin under 5,000).

1. In the Firebase console, create a project (or use one you have), open **Build > Realtime Database** and create a database. Any location is fine.
2. Open the **Rules** tab, paste these rules and publish:

   ```json
   {
     "rules": {
       "dt": {
         "$code": {
           ".read": "$code.length >= 16",
           ".write": "$code.length >= 16"
         }
       }
     }
   }
   ```

3. Copy the database address from the **Data** tab. It looks like `https://<name>-default-rtdb.firebaseio.com` or `https://<name>-default-rtdb.<region>.firebasedatabase.app`.
4. On the first phone: Setup, Shared balance, paste the address, tap **Make a code**, then **Turn on shared balance**. What that phone already used this month is added to the shared count.
5. On the second phone: paste the same address and type the same code, then turn it on.

Notes:

- The database holds only request counts per month and per day, and the shared limit. It never holds the Google key.
- The code is the only protection: anyone who has both the address and the code can read and change the counter. Keep the code private. Nothing else in the database is readable.
- If a phone is offline, its counts wait on the phone and are sent on the next sync. Until then the other phone does not see them, so the shared number can be a few requests behind.
- Changing the limit on one phone changes it for both.

Requests that never reach Google (no connection) are not counted.

### Setting up the key

1. In Google Cloud Console, create a project and enable billing.
2. Enable **Routes API** only.
3. Create an **API key** and restrict it: Application restrictions, Websites, `https://<account>.github.io/*`; API restrictions, Routes API.
4. Under APIs and services, Routes API, Quotas, set a daily cap of about 160 requests. That is a hard stop on Google's side.
5. Add a billing budget alert.
6. Open the app, go to Setup, paste the key and save.

For the live traffic map, also enable **Maps JavaScript API** and add it to the key's API restrictions. Map loads are counted separately from route requests ("Dynamic Maps" on Google's pricing page, 10,000 free a month when this was written). The app counts its own map loads and shows the number under the map.

Check Google's current pricing page for the free monthly allowance of "Compute Routes Pro" before relying on the 5,000 figure. The requests use traffic-aware routing and fewer than 11 stops, and avoid the features that move a request to the more expensive Enterprise tier (traffic on the route line, tolls, two-wheelers).

## Passcode lock

The Google key, the Firebase database name and the shared-balance sync code are built into `index.html` **encrypted with the app passcode** (PBKDF2-SHA256 with 600,000 rounds, then AES-256-GCM). The repository and the website only ever contain the scrambled block, so reading the source does not reveal the key.

- Anyone opening the site sees only the lock screen. Without the passcode the app shows nothing and makes no Google requests.
- Enter the passcode once on each phone. The app then fills in the key and turns on the shared balance by itself, finding whether the database is in Singapore or the US.
- **Setup > Google key > Lock this phone** removes the key and the sync settings from that phone. Unlock again with the passcode.
- The passcode is not stored anywhere in the repository. Keep it private, and do not reuse it elsewhere.
- To change the passcode, the key or the database, the encrypted block has to be rebuilt. Ask for a rebuilt `index.html`, then upload it and raise the version in `sw.js`.

The Firebase rules from "One shared balance for both phones" must still be published, or the shared balance shows "The database refused the request".

## Publish on GitHub Pages

1. Create a repository and upload every file in this folder, including the hidden `.github` folder and `.nojekyll`.
2. Open **Settings > Pages**. Under **Build and deployment**, set **Source** to **GitHub Actions**.
3. Wait for "Deploy to GitHub Pages" in the **Actions** tab to turn green.
4. The app is live at `https://<account>.github.io/<repository>/`.

All paths are relative, so it works under a repository sub-path.

## Make an Android APK with PWABuilder

1. Open the live address on your phone once to confirm it loads.
2. Go to https://www.pwabuilder.com and paste the live address.
3. Choose **Package for stores > Android**, set a package ID such as `com.yourname.dhakatraffic`, and generate.
4. The zip contains the `.apk` for direct install, the `.aab` for Google Play and a signing key. **Keep the signing key and its password.** You need the same key for every update.
5. If the installed app shows a browser address bar, add the `assetlinks.json` from the PWABuilder zip at `/.well-known/assetlinks.json` on the site. Android needs that file at the root of the domain, so host the app in a repository named `<account>.github.io` or on a custom domain for a clean full-screen app.

The Google key and the balance are saved inside each installed app. Enter the key once on each phone.

## Updating the app

Replace `index.html`, then raise the version in `sw.js` (`dhaka-traffic-v11` to `dhaka-traffic-v12`) so installed copies refresh. Push to `main`.

## Changing things in the code

Near the top of the `<script>` in `index.html`:

- `PLACES`: the six places with their Google place IDs and coordinates. Coordinates can also be changed in Setup without touching the code.
- `ROUTES`: starting clear-road minutes and distance for each route (used only until Google has answered once), and which crossings each route times by default.
- `ANCH`: the landmarks used to find the crossings on Google's route line, including the Hatirjheel exit.
- `pickupEvery`: the pickup schedule.
- `TRIPS_DEF` and `ARR_DEF`: the usual trips and the starting arrive-by time.
- Home-screen shortcuts are in `manifest.webmanifest` under `shortcuts`.
- `PROFILES`: when the peaks happen in the estimate.
