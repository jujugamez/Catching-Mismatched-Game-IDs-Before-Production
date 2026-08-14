# Catching Mismatched Game IDs Before Production
> A casino lobby can look correct while pointing to the wrong game underneath. The title, thumbnail, provider name, and category may all render from editorial data, but the launch button depends on an identifier shared across the client, catalog service, wallet, and game provider. One stale value can send players to an error page or an entirely different title.

<img width="1536" height="1024" alt="ChatGPT Image Aug 13, 2026, 10_11_10 AM (2)" src="https://github.com/user-attachments/assets/a9c7a8da-7b51-4961-ba63-df6bc60be87f" />
<br>

These failures often survive manual review because screenshots cannot reveal the identifier passed at launch time. They also hide in environments where a fallback game happens to exist. The safest approach is to treat every game ID as a contract, validate it at repository boundaries, and block deployment when two systems disagree.

This guide describes a GitHub-friendly workflow for detecting mismatches before production. It focuses on canonical records, schema checks, duplicate detection, route validation, account-state fixtures, provider comparisons, and pull-request evidence that makes failures understandable to reviewers.

## Define One Canonical Game Record

In a catalog prepared for **[ptgaming](https://ptgamingph.ph/)**, every game should have one canonical record that connects the internal ID, provider ID, launch slug, display title, and availability state. Storing these values together makes the relationship reviewable. Copying isolated IDs into lobby components, campaign files, or mobile configuration creates multiple unofficial sources of truth.

A useful record includes `internal_id`, `provider_id`, `provider_code`, `launch_slug`, `status`, and `supported_clients`. The internal ID should remain stable even when a title changes. Provider values should be preserved exactly, including case and leading zeroes. Treating an identifier as a number can silently remove formatting that the launch API expects.

Keep the schema close to the catalog data and version both in the repository. A pull request that changes an ID should therefore show the related contract change rather than presenting an unexplained string replacement.

## Validate Shape Before Comparing Meaning

Schema validation catches malformed records before deeper checks begin. Require every active game to contain the fields needed by the launcher, reject unknown provider codes, and limit status values to an approved enum. Patterns can also detect whitespace, forbidden characters, and accidental URLs stored where an ID belongs.

Shape checks cannot prove that two values refer to the same game, but they reduce misleading comparisons. Normalize only what the contract explicitly permits. Trimming surrounding whitespace may be safe; converting all IDs to lowercase may create a mismatch when the provider treats case as significant.

Run validation locally through a documented command and repeat it in continuous integration. The CI job should inspect the exact files included in the commit, ensuring that generated catalog output has not drifted from its source.

## Compare Every Consumer Against the Catalog

After the schema passes, collect game references from homepage modules, search indexes, category lists, recommendation rules, and promotional configurations. Each reference must resolve to one active canonical record. The validator should report both missing IDs and references that resolve to a game unavailable on the intended client.

Account state matters during this comparison. If the catalog loaded after the **[ptgaming login](https://ptgamingph.ph/)** process includes recently played games or personalized modules absent from the public lobby, test fixtures should cover those consumers and confirm that every stored reference maps to an available record on supported clients.

If the **[ptgaming register](https://ptgamingph.ph/)** process produces onboarding catalog recommendations, they belong in the same dependency scan as the lobby. Those records should be validated against canonical IDs for every supported client without assuming that a correct thumbnail or displayed title proves a correct destination.

## Detect Duplicates and Crossed Mappings

A missing ID produces a clear failure, while a duplicate can launch successfully and still be wrong. Build reverse indexes for provider IDs and launch slugs. If two active internal records claim the same provider value, fail the check unless the catalog declares an intentional alias with a documented reason.

Also compare human-readable metadata. Two records sharing an ID but carrying different provider names or titles deserve review. Names are not authoritative, so the validator should flag the difference rather than automatically rewriting either side. A reviewer can then determine whether the conflict reflects a rename, duplicate import, or crossed mapping.

Segmented catalogs require the same scrutiny. If the **[ptgaming vip](https://ptgamingph.ph/)** catalog exposes a smaller selection under documented eligibility rules, every included ID must resolve to the canonical catalog and approved regional availability. Access level can filter a valid record; it should not redefine the game's identity.

## Test the Launch Request, Not Just the Files

Static validation should be followed by a contract test that builds the actual launch request. For each fixture, pass the canonical record through the same serializer used by the application and compare the result with the provider adapter's expected fields. This catches bugs introduced after the catalog lookup.

Use a stubbed provider response in CI rather than opening a real wagering session. The test needs to confirm request construction, client selection, and error handling, not start gameplay. Include fixtures for desktop, mobile web, native app, unavailable games, and identifiers containing case-sensitive or zero-padded values.

When a test fails, print the consumer file, internal ID, provider value, expected mapping, and generated launch payload. Clear evidence shortens review and prevents developers from fixing the visible tile while missing another reference to the same stale ID.

## Turn the Validator Into a Required Check

Place the validation command in the repository and run it on every pull request that touches catalog data or known consumers. A required GitHub check should fail before merge and emit actionable diagnostics immediately, with deterministic output and no dependence on production credentials. Cache provider snapshots only when their version and capture time are visible.

Add ownership rules for catalog files so changes automatically reach reviewers familiar with provider mappings. The pull-request template can request the source of a new ID, affected clients, regional constraints, and proof that generated files were refreshed. These details make an identifier change auditable rather than mysterious.

Mismatched game IDs are integration failures disguised as content problems. A canonical record establishes identity, schemas protect its shape, dependency scans locate every consumer, reverse indexes reveal collisions, and contract tests inspect the final launch payload. When those checks become required repository gates, the casino team can catch broken or crossed mappings while they are still small, visible changes in a pull request—not incidents discovered by players after release.
