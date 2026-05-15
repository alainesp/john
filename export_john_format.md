# John Format Export Playbook for Hash Suite

This is the John-side companion to Hash Suite's canonical format guide:

`C:\Users\alainesp\Desktop\Repositories\Hash_Suite\Hash_Suite\docs\format.md`

Use `docs\format.md` for Hash Suite's native contracts, wiring, test harness,
and implementation patterns. Use this playbook to extract the John facts and to
record the Hash Suite decisions that must be settled before coding.

The required artifact is a per-format **export brief**. Do not start the port
until the brief identifies the John sources, accepted syntax, canonical text,
binary/salt layout, converter scope, CPU plan, OpenCL plan, unsupported variants,
and test matrix.

## Ground Rules

- Port format behavior, not John's framework.
- Treat `prepare`, `valid`, `split`, `binary`, `salt`, `source`,
  `tunable_cost_value`, test vectors, common files, converter tools, and OpenCL
  files as one source contract.
- CPU text import and converter correctness are the first milestone.
- A full CPU+OpenCL port is complete only after GPU parity passes for every
  advertised Hash Suite OpenCL provider.
- If John has a relevant `*2john` converter and Hash Suite should import the raw
  application file, the converter is part of the normal export.
- Converter dependencies must be stdlib-only, vendored, or staged minimally so
  raw-file import works offline in Hash Suite's embedded Python runtime.
- Unsupported versions, ciphers, cost modes, encodings, and container variants
  must reject during import. Do not import hashes the cracker cannot reproduce.
- Binary and salt storage must be deterministic. Zero unused salt bytes and make
  `convert_to_string(convert_to_binary(x))` round-trip to the chosen canonical
  string.
- The test bar is strict: John vectors, invalid near-misses, conversion
  round-trips, CPU provider tests, converter import tests, and all advertised
  OpenCL provider tests.

## Audit Commands

Run these from the John tree:

```powershell
rg --files src run | rg "(<label>|opencl_<label>|<label>_common|2john)"
rg -n "FORMAT_LABEL|FORMAT_TAG|fmt_tests|struct fmt_main|FMT_EXTERNS_H|FMT_REGISTERS_H|john_register_one" src run
rg -n "prepare|valid|split|binary|salt|source|tunable_cost|set_salt|set_key|crypt_all|cmp_all|cmp_one|cmp_exact" src
rg --files run\opencl | rg "<label>|common|pbkdf|sha|aes|hmac"
```

Run these from the Hash Suite tree:

```powershell
Get-Content docs\format.md
rg -n "typedef struct Format|convert_to_binary|convert_to_string|is_valid_line|add_hash_from_line|impls|opencl_impls" Interface.h
rg -n "extern Format|formats\[\]|MAX_NUM_FORMATS|FORMAT_.*INDEX" Interface.h common.cpp
rg -n "python_get_hashes|import_.*hashes|site_import|Tools|StagePythonTools" python_embedding.cpp in_out.cpp Hash_Suite.vcxproj
rg -n "hash_format|hash_count_to_test|password_was_found|report_keys_processed|test_format" attack.cpp hash.cpp test_format.py
rg -n "FORMAT_MAX_LENGTH|SampleCase|<Format.name>|<case_id>" test_samples.py
```

Stable John anchors:

- `src/formats.h`: `fmt_params`, `fmt_methods`, and callback semantics.
- `src/*_fmt_plug.c` or `src/*_fmt.c`: CPU format metadata, crypt loop, compare
  behavior, and registration stanzas.
- `src/*_common_plug.c` and `src/*_common.h`: shared validation, binary/salt
  parsing, test vectors, constants, and salt structs.
- `src/opencl_*_fmt_plug.c`: John OpenCL host lifecycle, host/device structs,
  autotune, and compare behavior.
- `run/opencl/*.cl` and `run/opencl/*.h`: device math and constants.
- `run/*2john.*` and `src/*2john.c`: raw input syntax and conversion behavior.

Stable Hash Suite anchors:

- `docs\format.md`: canonical destination guide.
- `Interface.h`: `Format`, protocol ids, indexes, and public callbacks.
- `common.cpp`: registered `Format` objects.
- `in_out.cpp`: text and raw-file import routing.
- `python_embedding.cpp`: embedded Python import bridge and `get_hashes` path.
- `Hash_Suite.vcxproj`: source compilation and Python tool staging.
- `attack.cpp`, `hash.cpp`, and `test_format.py`: tests, benchmarks, providers,
  found-record mapping, and progress accounting.
- `test_samples.py`: real sample-file import and crack verification.

## Sample Discovery Links

When building the sample export plan, check these sources before inventing new
fixtures. Prefer raw encrypted files when Hash Suite has or should have a raw
converter/import path; prefer text hashes when the format has no raw container.

- [openwall/john-samples](https://github.com/openwall/john-samples): maintained
  raw encrypted files, captures, wallets, documents, databases, archives, and
  related sample artifacts. Filenames or nearby `password.txt` files often carry
  the cleartext.
- [Openwall sample non-hashes](https://openwall.info/wiki/john/sample-non-hashes):
  historical index for raw encrypted sample files and format-specific notes. The
  page points to `openwall/john-samples`, but it can still be useful for
  passwords, provenance, and format groupings.
- [Openwall sample hashes](https://openwall.info/wiki/john/sample-hashes):
  text hash encodings, plaintexts, and larger sample hash files for classic
  account/hash formats.
- [Hashcat example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes):
  public text hash examples and, for some binary/container formats, direct links
  to sample files under `hashcat.net/misc/example_hashes/`. Unless that page
  says otherwise, Hashcat examples use `hashcat` as the cleartext.
- [Hashcat hash format guidance](https://hashcat.net/wiki/doku.php?id=hash_format_guidance):
  extraction/conversion notes for formats where the example hash alone is not
  enough to understand how users obtain the attackable string.
- [Hashcat test modules](https://github.com/hashcat/hashcat/tree/master/tools/test_modules):
  generated test-vector logic by mode ID. Use this as a secondary source when
  the example-hashes page lacks a variant, and verify the resulting syntax
  against Hash Suite's supported boundary before adding it as an active sample.

Keep unsupported-but-interesting external examples under a format-local
`needs-review` folder in `..\Hashes_Samples` with a short README explaining
whether they failed import, cracking, converter extraction, length limits,
edition/backend support, or another explicit boundary.

## John Extraction Checklist

Read `src/formats.h` when the meaning of a callback is unclear. For each format,
extract only the facts Hash Suite needs.

From `fmt_params`:

- `label`, `format_name`, `algorithm_name`, and `signature`.
- `plaintext_min_length`, `plaintext_length`, `binary_size`, `binary_align`,
  `salt_size`, and `salt_align`.
- `flags`, especially case handling, 8-bit/UTF-8/encoding behavior,
  `FMT_SPLIT_UNIFIES_CASE`, `FMT_HUGE_INPUT`, `FMT_DYNA_SALT`,
  `FMT_NOT_EXACT`, OpenMP, and mask support.
- `tunable_cost_name` and every reported cost value.
- `tests`, including non-ASCII plaintexts and unusual syntaxes.

From `fmt_methods`:

- `prepare`: whether John builds the real ciphertext from account fields,
  usernames, domains, RIDs, paths, or adjacent tokens.
- `valid`: accepted syntax, lengths, alphabets, version/cipher checks, and
  multi-part return behavior.
- `split`: canonical case, aliases, or multiple crackable parts.
- `binary` and `salt`: exact digest bytes, salt fields, endian transforms,
  embedded costs, blob lengths, subtypes, and precomputed state.
- `source` and `convert` helpers: how canonical text is reconstructed.
- `set_salt`, `set_key`, `crypt_all`, `cmp_all`, `cmp_one`, and `cmp_exact`:
  candidate layout, salt selection, result comparison, and exact verification.

Also inspect:

- Common files shared between CPU and OpenCL.
- Registration under `FMT_EXTERNS_H` and `FMT_REGISTERS_H`.
- OpenCL host structs, buffers, kernels, build options, and autotune loops.
- Converter scripts and C tools that produce John-compatible hash lines.

## Required Export Brief

Create this brief before implementation. Put it in the PR body, issue, design
note, or top-of-change notes so reviewers and later agents can audit the port.

```markdown
# Export Brief: <Format>

## Source Inventory
- John CPU format:
- John common files:
- John OpenCL host:
- John OpenCL kernels/includes:
- John converters:
- Registration anchors:
- Test vector anchors:
- Audit commands run:

## Accepted Syntax And Canonicalization
- John labels/signatures:
- Raw and prefixed forms accepted:
- Account fields consumed by `prepare`:
- Case, base64, hex, Unicode, and encoding rules:
- Canonical Hash Suite string:
- Invalid near-misses to reject:

## Binary And Salt Layout
- Hash Suite `binary_size`:
- Hash Suite `salt_size`:
- Binary byte/word order:
- Salt struct fields and zeroing rule:
- Cost/subtype fields stored in salt:
- `convert_to_binary` return value:
- `value_map_index0/1` decision:
- Round-trip example:

## Costs, Subtypes, And Support Boundary
- Supported versions/ciphers/modes:
- Unsupported John-accepted variants:
- Tunable cost names and values:
- Benchmark/test cost choices:

## Converter Plan
- Raw file extensions or sources:
- John converter behavior to preserve:
- Hash Suite `get_hashes(filename)` output contract:
- Dependency vendoring/staging plan:
- Runtime error strings:
- Text-import parity check:

## Sample Export Plan
- John sample anchors:
- External sample source URLs checked:
- Hash Suite sample filename(s) under `..\Hashes_Samples`:
- `test_samples.py` case id(s):
- Expected `Format.name` and imported hash count:
- Expected cleartexts and unsupported/too-long cleartexts:
- Secret loader strategy:
- Smoke or full-only selection:
- Sample verification commands:

## Hash Suite Identity And Wiring
- `Format.name`, description, and prefix:
- `db_id` and `*_INDEX`:
- Source file name:
- Registry/project/help/test files to edit:
- Closest Hash Suite pattern from `docs\format.md`:

## CPU Plan
- Provider protocol:
- Candidate length limit:
- Salt iteration strategy:
- Compare and exact verification:
- Thread-safety notes:
- Progress reporting:

## OpenCL Plan
- Deferred or included in this change:
- Provider protocols to advertise:
- John kernel math to reuse:
- Host/device layout mapping:
- Buffer/index/found-record mapping:
- CPU/GPU parity tests:

## Unsupported Variants
- Rejected syntaxes:
- Rejected raw-file cases:
- User-visible error text:

## Test Matrix
- John vectors:
- Invalid near-misses:
- Conversion round-trips:
- Same-salt and multi-salt cases:
- Hash Suite sample-file cases:
- CPU `test_format.py` commands:
- Converter compile/import/sample tests:
- `test_samples.py` commands:
- OpenCL `test_format.py` commands for every advertised provider:
```

## Export Workflow

1. **Inventory John sources.** Find CPU, common, OpenCL, kernel, converter, and
   registration files. Record commands and anchors in the brief.
2. **Define syntax.** Combine `prepare`, `valid`, `split`, converter output, and
   tests into one accepted-input contract. Decide the single canonical string
   Hash Suite will store/export.
3. **Define storage.** Choose deterministic `binary_size`, `salt_size`, endian
   layout, cost/subtype fields, zeroing rules, and value-map indexes.
4. **Plan raw-file import.** If a converter exists and raw import is in scope,
   make it import-safe and offline-capable before wiring it into Hash Suite.
5. **Export test samples.** Copy or derive useful John samples into Hash Suite's
   `..\Hashes_Samples` directory and add focused
   [test_samples.py](../Hash_Suite/Hash_Suite/test_samples.py) cases. Check
   [openwall/john-samples](https://github.com/openwall/john-samples) first for
   maintained raw application, capture, wallet, document, and database samples.
6. **Implement CPU first.** Follow `docs\format.md` for `Format` wiring,
   validation/import, conversion, CPU providers, tests, and benchmarks.
7. **Add OpenCL second.** Port or rewrite the John device math to Hash Suite's
   generated-source and protocol model. Advertise only protocols that pass.
8. **Verify the matrix.** Run focused tests first, then adjacent CPU/GPU/provider
   cells as risk requires. Record any skipped cells and remaining risk.

## Converter Export Policy

John's `run\*2john.py`, `run\*2john.pl`, and `src\*2john.c` files often define
the real-world input format more accurately than `valid()` does. Treat a
converter as in scope when Hash Suite users should import the original encrypted
file, capture, wallet, database, or document directly.

For Python converters:

- Stage the converter under Hash Suite's `..\Python Scripts\` directory.
- Preserve normal command-line behavior when practical.
- Add a safe Hash Suite entry point:

  ```python
  def get_hashes(filename: str) -> list[str]:
      ...
      return ["john-compatible-hash-or-human-readable-error"]
  ```

- Do not perform work, parse arguments, print warnings, or call `sys.exit()` at
  import time.
- Move optional dependency failures to `get_hashes()` and return an error string
  instead of breaking module import.
- Prefer stdlib extraction. If third-party code or data files are required,
  vendor the minimum and update Hash Suite's staging target; the default project
  staging copies top-level `..\Python Scripts\*.py` files into `Tools\`.
- Remember that Hash Suite embeds Python with `site_import = 0`, so user
  installed packages are not available by default.

Hash Suite raw-file import wiring belongs after text import works:

- `python_embedding.cpp`: add `import_<format>_hashes(ImportParam* param)` near
  `import_office_hashes`. Call `python_get_hashes("Tools.<script>", filename)`,
  validate each returned string with the format object, insert canonical hashes,
  save the imported tag range, show nonblocking errors for non-hash returns, and
  set completion/end flags.
- `in_out.cpp`: route raw file extensions before the generic text importer reads
  the file. Use case-insensitive extension checks for document-style formats.
- `Hash_Suite.vcxproj`: update staging if the converter needs package
  directories, data files, or native helpers.

## Test Sample Export Policy

When John has usable real samples, export the smallest stable set into Hash
Suite's `..\Hashes_Samples` directory and wire it into
[test_samples.py](../Hash_Suite/Hash_Suite/test_samples.py). Prefer raw
application, capture, wallet, document, or database samples when converter import
is in scope; use text-hash samples when the format has no raw container.

Use [openwall/john-samples](https://github.com/openwall/john-samples) as the
first place to look for maintained real-world sample files. It carries the
sample-non-hash corpus that used to be tracked on the Openwall wiki and is often
better than mining John source vectors when a raw converter/import path exists.

For each exported sample:

- Add or update `FORMAT_MAX_LENGTH` for the `Format.name`.
- Add a stable `SampleCase`: `case_id`, filename, `Format.name`, expected hash
  count, expected cleartexts, timeout, and `smoke`/`notes` when needed.
- Use an inline `secrets` tuple for short fixed cases. Add a small
  `SecretLoader` helper when the cleartexts already live in a text sample or
  need filtering.
- Mark only fast, representative cases as `smoke=True`.
- For import-only samples, use empty `secrets` and explain the missing cleartext
  in `notes`.
- If the sample includes too-long or unsupported cleartexts, record that support
  boundary in the export brief; the harness filters expected secrets by
  `FORMAT_MAX_LENGTH`.

Sample verification commands from the Hash Suite tree:

```powershell
python test_samples.py --format "<Format.name>" --list
python test_samples.py --case <case_id> --config RelWithDebInfo --platform x64
python test_samples.py --format "<Format.name>" --config RelWithDebInfo --platform x64
python test_samples.py --case <case_id> --config RelWithDebInfo --platform x64 --backend gpu --gpu 0
```

Use the GPU sample command only when the format advertises a GPU backend and the
sample is small enough to be useful. For raw-file import, compare converter
output with John's converter once, then keep the `test_samples.py` case as the
repeatable regression.

Converter verification:

```powershell
python -m py_compile "..\Python Scripts\<script>.py"
python -c "import sys; sys.path.insert(0, r'..\Python Scripts'); import <script>; print(<script>.get_hashes(r'<sample>'))"
python test_format.py "<Format.name>" --config RelWithDebInfo --platform x64
python test_samples.py --case <case_id> --config RelWithDebInfo --platform x64
```

When a sample exists, compare converter output with John's converter once, then
use `test_samples.py` to keep raw-file import, hash insertion, and cracking
covered by one repeatable command.

## OpenCL Export Policy

John OpenCL formats usually have a host file plus one or more `run\opencl`
kernels. Do not copy the lifecycle mechanically. Hash Suite OpenCL generally
uses format-local generated source and provider-specific init functions.

Extract from John:

- Device math, constants, and helper headers from `run\opencl`.
- Host salt/input/output structs from `src\opencl_*_fmt_plug.c`.
- Build options, loop structure, kernel sequence, and autotune assumptions.
- Compare width and exact verification behavior.

Map into Hash Suite:

- `Format.opencl_impls[]` entries only for supported providers.
- `OpenCL_Param`, generated source helpers, and buffer slots documented in
  `docs\format.md`.
- `binary_values`, `salts_values`, `salt_index`, and `same_salt_next` layouts
  that match the CPU storage decision.
- Found callbacks that use original hash indexes, not unique-salt indexes.

OpenCL is a second milestone. If it is deferred, leave unsupported/null entries
as described in `docs\format.md` and state the deferral in the export brief.

## Reference Example: Ansible

Ansible is a useful John split because CPU, common parsing, converter, and
OpenCL are separate:

- `src\ansible_common.h`: shared constants such as `$ansible$`, salt/blob
  lengths, and the custom salt struct.
- `src\ansible_common_plug.c`: test vectors, `$ansible$` validation, binary
  extraction from the checksum, salt packing for iterations, salt bytes, blob,
  blob length, and checksum.
- `src\ansible_fmt_plug.c`: CPU metadata, `FMT_CASE | FMT_8_BIT |
  FMT_HUGE_INPUT`, PBKDF2-HMAC-SHA256 plus HMAC-SHA256 work, and compare
  callbacks.
- `src\opencl_ansible_fmt_plug.c`: OpenCL buffers, salt transfer, kernel build
  options, autotune, PBKDF2 loop kernels, final kernel, and result compare.
- `run\opencl\ansible_kernel.cl`: device-side PBKDF2/HMAC work.
- `run\ansible2john.py`: raw Ansible Vault file conversion into the
  John-compatible `$ansible$...` string.

For Hash Suite, the export brief should make one shared validation/conversion
plan, a CPU provider plan, a converter `get_hashes()` plan for vault files, and
a deferred-or-included OpenCL plan. The CPU/common/OpenCL split is a source
inventory model, not a destination file layout requirement.

## Completion Checklist

- Export brief is complete and matches the audited John files.
- No volatile snapshots, current format counts, or copied registry tables were
  used as authority; anchors and commands are recorded instead.
- John vectors import and crack.
- Invalid near-misses reject.
- `convert_to_binary` and `convert_to_string` round-trip canonical text.
- Raw converter output imports through the same validation path as text hashes.
- Converter modules are import-safe and work offline after staging.
- Exported sample files live under Hash Suite's `..\Hashes_Samples`, and
  `test_samples.py --case <case_id>` passes for each new sample case.
- CPU tests pass for the advertised provider protocol.
- Same-salt and multi-salt cases map found passwords to the correct hash index.
- OpenCL tests pass for every advertised provider, or OpenCL is explicitly
  deferred and not advertised.
- Hash Suite wiring, benchmark/test support, help/UI text, and project files are
  updated only when the format is product-visible.
- `git diff --check` is clean.
