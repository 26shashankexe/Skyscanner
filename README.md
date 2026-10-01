# Hoen Scanner

A tiny Dropwizard microservice that loads rental-car and hotel listings from
two JSON files and exposes a single `POST /search` endpoint that filters
them by city.

**Stack:** Java 19, Dropwizard 4.0 (Jetty + Jersey + Jackson under the hood), Maven.

## Project layout

```
hoen-scanner/
├── pom.xml
├── config.yml
└── src/main/
    ├── java/com/skyscanner/
    │   ├── HoenScannerApplication.java   # entry point, loads data, registers resource
    │   ├── HoenScannerConfiguration.java # empty config class (room to grow)
    │   ├── Search.java                   # deserialised request body { "city": "..." }
    │   ├── SearchResult.java             # serialised response item
    │   └── SearchResource.java           # the /search endpoint
    └── resources/
        ├── banner.txt                    # printed on startup
        ├── rental_cars.json               # sample car listings
        └── hotels.json                    # sample hotel listings
```

## How to run it

**Option A — IntelliJ (matches the task instructions)**
1. Open the `hoen-scanner` folder as a project in IntelliJ.
2. Let it detect the Maven build and click **Load Maven Project**.
3. Install OpenJDK 19 if IntelliJ prompts you to.
4. Create a Run Configuration for `HoenScannerApplication` with program
   arguments: `server config.yml`
5. Click the green run arrow. You should see `Welcome to Hoen Scanner!`
   in the logs once it's up.

**Option B — command line (if you have Maven + JDK 19 installed locally)**
```bash
mvn package
java -jar target/hoen-scanner.jar server config.yml
```

The service listens on `http://localhost:8080` (admin/health checks on `:8081`).

## Testing it

With Postman (or `curl`), send:

```bash
curl -X POST http://localhost:8080/search \
  -H "Content-Type: application/json" \
  -d '{"city": "petalborough"}'
```

You should get back a JSON array mixing car and hotel results for
Petalborough. Try `rustburg` and `shaleport` too (the bundled sample data
covers all three) — and try an unknown city or a malformed body to see the
empty-list / 422 validation behavior.

## What I'd extend first

1. **Case-insensitive / trimmed matching.** Right now `result.getCity().equals(search.getCity())`
   is a strict, case-sensitive match — "Petalborough" and "petalborough " (trailing space)
   won't match. Normalize both sides (`.trim().toLowerCase()`) before comparing.
2. **Proper "no city" / empty-body handling.** `@NotNull @Valid` on `Search` rejects a
   missing body, but an empty string `{"city": ""}` currently just returns an empty list
   rather than a clear 4xx. Add a `@NotEmpty` constraint on the `city` field.
3. **Split into two endpoints or add a `kind` filter.** `/search` currently always
   returns both cars and hotels mixed together. A `kind` query param (`?kind=hotel`)
   or separate `/search/cars` and `/search/hotels` resources would make the API more
   useful to a real client.
4. **Load data once at startup into a faster lookup structure.** For two small JSON
   files, the linear scan in `search()` is fine — but swapping `searchResults` for a
   `Map<String, List<SearchResult>>` keyed by lowercased city would make lookups O(1)
   and scale better if the dataset grows.
5. **Add tests.** Dropwizard ships with `dropwizard-testing` for exactly this — a
   `DropwizardAppExtension` (JUnit 5) spins up the app in-process so you can POST to
   `/search` and assert on the response without Postman.

## A note on process

Since this looks like it's paired with a job application, treat this repo as a
reference to check your understanding against — not a drop-in submission. The parts
worth doing yourself: forking/cloning the real starter repo, getting IntelliJ + the
JDK set up, running it, testing with Postman, and the git commit/push. That's likely
part of what's being evaluated, and you'll want to be able to talk through the code
comfortably in a follow-up conversation.
