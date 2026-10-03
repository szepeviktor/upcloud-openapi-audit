https://developers.upcloud.com/api/1.3/upcloud-openapi.json

# UpCloud OpenAPI verification instructions

When asked to verify the UpCloud schema, execute every step below from the repository root. Report findings without automatically correcting or reformatting the source document. Continue after failed checks unless invalid JSON prevents a subsequent check.

## 1. Check HTTP response headers

Before checking the document, inspect the response headers from the source URL. Use a GET request, follow redirects, and discard the body so this checks the endpoint used for downloads rather than relying on HEAD behavior:

```bash
curl_chrome145 --http2 --compressed --location --silent --show-error \
  --dump-header - --output /dev/null \
  https://developers.upcloud.com/api/1.3/upcloud-openapi.json
```

Inspect the final response after redirects. Record the UTC check time, HTTP version, status, and all returned headers. Require a successful `200` response and a JSON `content-type` (allow parameters such as `charset`). Report missing headers and changes from the reference below; do not fail the audit merely because a dynamic header changed.

Reference response observed on 2026-10-03:

```http
HTTP/2 200
date: Sat, 03 Oct 2026 08:13:11 GMT
content-type: application/json
server: cloudflare
strict-transport-security: max-age=15552000; includeSubDomains
last-modified: Thu, 01 Oct 2026 16:29:13 GMT
etag: W/"6abe8a59-25af0e"
cf-cache-status: DYNAMIC
content-encoding: br
cf-ray: a44a7c707a41d388-FRA
```

Check HSTS, content type, and available cache validators (`last-modified` and `etag`). Treat `date`, `cf-ray`, cache status, and negotiated content encoding as variable metadata. Treat `last-modified` and `etag` as version indicators, not fixed expected values. Brotli (`br`) is acceptable when negotiated, but is not required. If using a local input, clearly distinguish the current remote headers from the identity of the local document; do not imply they describe the same version unless verified.

## 2. Establish the input

- Read applicable `AGENTS.md` instructions and check the Git working tree.
- Use `upcloud-openapi.json` as the input. If asked to check the latest published document, download the URL above to a temporary file using `curl_chrome145`, `curl_firefox147`, or `curl_safari260`. Preserve the local document unless replacement was requested. Use the selected input in all subsequent commands.
- Use temporary files for validator helpers and downloaded validation schemas. Do not install dependencies globally or into this repository. Run Node.js validators through `npx`.
- Record the input's SHA-256 hash, OpenAPI version, download time if applicable, and tool versions. Do not assume that previous findings still apply.

```bash
sha256sum upcloud-openapi.json
jq -r '.openapi' upcloud-openapi.json
```

## 3. Check JSON syntax

```bash
jq empty upcloud-openapi.json
```

Exit code `0` means the JSON parses. If this fails, continue only checks that do not require parsed JSON.

## 4. Check formatting

```bash
bash -o pipefail -c \
  'jq -j . upcloud-openapi.json | diff -u upcloud-openapi.json -'
```

Exit code `0` means the document matches jq's default two-space formatting without a final newline. `-j` intentionally omits the final newline. Keep `pipefail`, report differences, and do not overwrite the input.

## 5. Check characters

List all lines containing literal non-ASCII characters:

```bash
grep --perl-regexp --with-filename --line-number \
  '[^\x00-\x7F]' upcloud-openapi.json
```

Then check for characters outside printable ASCII and the permitted typographic characters `“`, `”`, `’`, `—`, and `–`:

```bash
env LC_ALL=C.UTF-8 grep --perl-regexp --with-filename --line-number \
  '[^ -~“—’–”]' upcloud-openapi.json
```

Grep exit code `1` means no matches. Report code points and occurrence counts for unexpected characters. Distinguish permitted non-ASCII punctuation from policy violations. These checks inspect literal file characters, not decoded Unicode escape sequences.

## 6. Validate the entire document with Hyperjump

Use the official OpenAPI **schema-base** and Hyperjump. Do not use Ajv for this check: its documented limitation with nested `$dynamicAnchor` declarations affects the official OpenAPI schema. Do not replace `$dynamicRef` with `$ref` in the validation schema.

1. Read the input's `openapi` version.
2. Consult <https://spec.openapis.org/oas/> and select the latest dated `schema-base` revision for that minor version. Record the exact URL; do not assume an earlier revision is still current.
3. Load the matching Hyperjump module, e.g. `@hyperjump/json-schema/openapi-3-1` for OpenAPI 3.1, through `npx`.
4. Import `@hyperjump/json-schema/formats` and explicitly enable format validation with `setShouldValidateFormat(true)`.
5. Import `BASIC` from `@hyperjump/json-schema/experimental` for individual error locations.
6. Download the official schema-base and its OpenAPI schema, dialect, and meta-schema dependencies to a temporary directory. Register them under their original `$id` values without modifying their contents. If an identical URI is already registered, unregister it before registering the downloaded version.
7. Validate the entire input with `validate(schemaBaseUri, document, BASIC)`. Save the complete result as `hyperjump-schema-base-result.json`.
8. Return exit code `0` for a valid document, `1` for invalid input, and a distinct nonzero code for execution errors. Do not confuse dependency or runtime failures with validation failures.

Core helper code, after imports and dependency registration:

```javascript
const result = await validate(schemaBaseUri, document, BASIC);
fs.writeFileSync(
  "hyperjump-schema-base-result.json",
  JSON.stringify(result, null, 2) + "\n"
);
process.exitCode = result.valid ? 0 : 1;
```

Create a temporary Node.js helper and run it through `npx`. Explicitly resolve modules from npx's temporary package environment if they are not available relative to the helper. Do not install dependencies into the project as a workaround.

Interpret the output as follows:

- Group `$schema`/`jsonSchemaDialect` constant mismatches separately from other errors. Official OpenAPI 3.1 schema-base requires the corresponding OpenAPI base dialect; a different dialect can fail this check without proving a general OpenAPI violation.
- For each distinct non-dialect error, report its instance JSON Pointer, keyword, offending value, and suggested correction where the intended meaning is clear.
- Inspect `pattern` values rejected as `regex`, especially anchors such as `\A` and `\z`. Do not assume a replacement preserves the original semantics.
- Do not claim that schema validation resolves all document `$ref` targets or validates examples against their models. Those require separate checks.

If schema-base rejects dialect declarations, also run Hyperjump against the unmodified official plain **schema** revision, with formats enabled. Save the result as `hyperjump-schema-result.json`. Explain that this checks the OpenAPI structure but does not validate embedded Schema Objects. A plain-schema pass is not a schema-base pass.

## 7. Lint OpenAPI with Redocly

Run Redocly CLI after Hyperjump as a separate check. Consult <https://redocly.com/docs/cli/commands/lint> when checking current options. Record the CLI version and the configuration used. Inspect any existing Redocly configuration and ignore file; do not silently suppress findings or generate a new ignore file.

First check the `spec` ruleset and save machine-readable results:

```bash
npx --yes @redocly/cli lint upcloud-openapi.json \
  --extends spec --format json --max-problems 10000 \
  > redocly-spec-result.json
```

Then run the `recommended` ruleset to check documentation and API design conventions:

```bash
npx --yes @redocly/cli lint upcloud-openapi.json \
  --extends recommended --format markdown --max-problems 10000 \
  > redocly-recommended-result.md
```

Capture each command's exit code independently and continue the audit after lint findings. Keep stderr available to distinguish lint findings from runtime or dependency failures. Use `rtk proxy` when saving output so RTK does not filter or truncate the artifacts. The default problem limit is 100; if even 10000 truncates the report, increase it and rerun.

- Check unresolved `$ref` targets, duplicate operation IDs or parameters, path-parameter consistency, schema and enum type mismatches, and example-related findings enabled by the selected ruleset.
- Separate `spec` results from additional `recommended` findings. Do not classify every recommended-rule failure as an OpenAPI specification violation.
- Group findings by rule and severity, include counts and representative JSON Pointers or source locations, and explain their practical impact.
- If existing configuration customizes rules or suppresses findings, record that explicitly. Use a temporary explicit configuration when an unmodified built-in ruleset is needed; do not change project configuration during the audit.
- Do not use `--generate-ignore-file` or modify the source to make lint pass.

Include both result files and their exit codes in the final dated report. Keep the Hyperjump result separate: linting does not replace validation against the official schema-base.

## 8. Check spelling

Read and apply the `typos-triage` skill. Inspect the existing typos configuration, then run:

```bash
mise exec typos -- typos --force-exclude upcloud-openapi.json
```

If typos is directly available, use `typos` instead. Do not install it without authorization.

Read each finding in context and distinguish prose errors, external identifiers, and false positives. Earlier examples include:

- `hel` in `fi-hel1` or `fi-hel2`: intentional zone identifiers.
- `initator`: an apparent spelling error in an API field name. Check the external contract before recommending a rename.

Do not treat these examples as exhaustive expected results. Do not change source text or add ignore rules during the audit.

## 9. Proofread the English manually

Extract unique human-readable text from documentation fields, example messages, display names, and schema comments:

```bash
jq -r '
  [.. | objects
   | (.description?, .summary?, .title?,
      .message?, .detail?, .reason?, .error_message?, .msg?,
      ."x-displayName"?, ."$comment"?)
   | strings
   | select(length > 0)]
  | unique[]
' upcloud-openapi.json > schema-text.txt
```

This field list has been manually reviewed for the current document. It is not guaranteed to capture every human-readable string in future versions: `example`, `examples`, `name`, `value`, and other fields may also contain prose. If coverage is uncertain, extract all nonempty strings with their JSON Pointer paths and inspect the omitted fields before extending this list. Do not include technical values indiscriminately.

Read and apply the `english-article-proofreader` skill. Read the entire extracted file in manageable chunks; check that tool output was not truncated. Do not use an automated grammar service.

- Check grammar, articles, agreement, prepositions, possessives, sentence completeness, natural phrasing, and terminology.
- Treat short labels, headings, identifiers, and code examples according to their context; they need not be complete sentences.
- Distinguish definite errors, optional improvements, and house-style decisions.
- Check consistency between British and American spelling and product capitalization.
- Group repeated issues. Quote the original, explain the problem, and suggest a correction.
- Give line numbers in `schema-text.txt`; do not confuse them with source JSON line numbers.
- Do not fact-check technical claims or modify the source as part of proofreading.

Save the English report as `pass-two-proofread.md`, using standalone GitHub-flavored Markdown suitable for a Gist. Use filenames and line numbers rather than local absolute links in that report.

## 10. Produce API statistics

Count operations only under HTTP method keys: `get`, `put`, `post`, `delete`, `options`, `head`, `patch`, and `trace`. Exclude path-level parameters, descriptions, summaries, extensions, and references. Resolve Path Item references if present, or explicitly limit the statistics to inline operations.

Report the following:

1. **API size:** path count, total operations, and operations by HTTP method.
2. **Operations by service:** group by operation tags, with a separate untagged count. Explain that operations with multiple tags can contribute to multiple service totals.
3. **CRUD coverage:** group collection and item paths into logical resources and identify create, read, update, and delete capabilities. Distinguish read-only resources, action endpoints, and apparent gaps. Do not infer a defect merely from a missing HTTP method on one path.
4. **Documentation coverage:** operations with nonempty `summary` and `description`, and request/response media types with examples. Define each denominator; keep request and response example coverage separate.
5. **Response coverage:** operations documenting successful `2xx` and explicit `4xx`/`5xx` responses. Include wildcard ranges such as `2XX` and `4XX`. Report `default` separately because it does not identify a response class.
6. **Deprecated elements:** counts and JSON Pointers for operations, parameters, and Schema Object properties marked `deprecated: true`. Distinguish this flag from prose mentioning deprecation.

Use jq or temporary Python/Node.js scripts as needed. State limitations caused by references or ambiguous resource grouping.

## 11. Produce a dated Markdown report

As the final step, save a standalone GitHub-flavored Markdown audit report in the repository root. Use the UTC date of the audit in the filename: `upcloud-openapi-report-YYYY-MM-DD.md` (for example, `upcloud-openapi-report-2026-10-03.md`). If that filename already exists, append the UTC time as `-HHMMSS` rather than overwriting an earlier report. State the full UTC audit timestamp inside the report.

Provide a compact table listing each check as pass, fail, findings, or not completed. Include:

- Input identity and tool versions.
- HTTP status and response headers, including differences from the reference response.
- Reproduction commands or helper invocation.
- Counts and representative locations for findings.
- Separate schema-base and plain-schema results if both were run.
- Separate Redocly `spec` and `recommended` results, including rule counts, severities, exit codes, and any configuration or ignored findings.
- Links to complete validation results, extracted text, and the proofreading report.
- Suggested fixes, compatibility concerns, and verification limits.

Leave `upcloud-openapi.json` unchanged unless corrections were explicitly requested.

After saving the report, return a concise outcome and a link to the dated Markdown file.
