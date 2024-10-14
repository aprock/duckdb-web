---
layout: community_extension_doc
title: shellfs
excerpt: |
  DuckDB Community Extensions
  Allow shell commands to be used for input and output

docs:
  extended_description: "The `shellfs` extension for DuckDB enables the use of Unix\
    \ pipes for input and output.\n\nBy appending a pipe character `|` to a filename,\
    \ DuckDB will treat it as a series of commands to execute and capture the output.\
    \ Conversely, if you prefix a filename with `|`, DuckDB will treat it as an output\
    \ pipe.\n\nWhile the examples provided are simple, in practical scenarios, you\
    \ might use this feature to run another program that generates CSV, JSON, or other\
    \ formats to manage complexity that DuckDB cannot handle directly.\n\n### Reading\
    \ input from a pipe\n\nCreate a program to generate CSV in Python:\n\n```python\n\
    #!/usr/bin/env python3\n\nprint(\"counter1,counter2\")\nfor i in range(10000000):\n\
    \    print(f\"{i},{i}\")\n```\n\nRun that program and determine the number of\
    \ distinct values it produces:\n\n```sql\nselect count(distinct counter1)\nfrom\
    \ read_csv('./test-csv.py |');\n┌──────────────────────────┐\n│ count(DISTINCT\
    \ counter1) │\n│          int64           │\n├──────────────────────────┤\n│ \
    \                10000000 │\n└──────────────────────────┘\n```\n\nWhen a command\
    \ is not found or able to be executed, this is the result:\n\n```sql\nSELECT count(distinct\
    \ column0) from read_csv('foo |');\nsh: foo: command not found\n┌─────────────────────────┐\n\
    │ count(DISTINCT column0) │\n│          int64          │\n├─────────────────────────┤\n\
    │                       0 │\n└─────────────────────────┘\n```\n\nThe reason why\
    \ there isn't an exception raised in this cause is because the `popen()` implementation\
    \ starts a shell process, but that shell process\n\n\n### Writing output to a\
    \ pipe\n\n```sql\n-- Write all numbers from 1 to 30 out, but then filter via grep\n\
    -- for only lines that contain 6.\nCOPY (select * from unnest(generate_series(1,\
    \ 30)))\nTO '| grep 6 > numbers.csv' (FORMAT 'CSV');\n6\n16\n26\n\n-- Copy the\
    \ result set to the clipboard on Mac OS X using pbcopy\nCOPY (select 'hello' as\
    \ type, from unnest(generate_series(1, 30)))\nTO '| grep 3 | pbcopy' (FORMAT 'CSV');\n\
    type,\"generate_series(1, 30)\"\nhello,3\nhello,13\nhello,23\nhello,30\n\n-- Write\
    \ an encrypted file out via openssl\nCOPY (select 'hello' as type, * from unnest(generate_series(1,\
    \ 30)))\nTO '| openssl enc -aes-256-cbc -salt -in - -out example.enc -pbkdf2 -iter\
    \ 1000 -pass pass:testing12345' (FORMAT 'JSON');\n\n```\n\n## Configuration\n\n\
    This extension introduces a new configuration option:\n\n`ignore_sigpipe` - a\
    \ boolean option that, when set to true, ignores the SIGPIPE signal. This is useful\
    \ when writing to a pipe that stops reading input. For example:\n\n```sql\nCOPY\
    \ (select 'hello' as type, * from unnest(generate_series(1, 300))) TO '| head\
    \ -n 100';\n```\n\nIn this scenario, DuckDB attempts to write 300 lines to the\
    \ pipe, but the `head` command only reads the first 100 lines. After `head` reads\
    \ the first 100 lines and exits, it closes the pipe. The next time DuckDB tries\
    \ to write to the pipe, it receives a SIGPIPE signal. By default, this causes\
    \ DuckDB to exit. However, if `ignore_sigpipe` is set to true, the SIGPIPE signal\
    \ is ignored, allowing DuckDB to continue without error even if the pipe is closed.\n\
    \nYou can enable this option by setting it with the following command:\n\n```sql\n\
    set ignore_sigpipe = true;\n```\n\n## Caveats\n\nWhen using `read_text()` or `read_blob()`\
    \ the contents of the data read from a pipe is limited to 2GB in size.  This is\
    \ the maximum length of a single row's value.\n\nWhen using `read_csv()` or `read_json()`\
    \ the contents of the pipe can be unlimited as it is processed in a streaming\
    \ fashion.\n\nA demonstration of this would be:\n\n```python\n#!/usr/bin/env python3\n\
    \nprint(\"counter1,counter2\")\nfor i in range(10000000):\n    print(f\"{i},{i}\"\
    )\n```\n\n```sql\nselect count(distinct counter1) from read_csv('./test-csv.py\
    \ |');\n┌──────────────────────────┐\n│ count(DISTINCT counter1) │\n│        \
    \  int64           │\n├──────────────────────────┤\n│                 10000000\
    \ │\n└──────────────────────────┘\n```\n\nIf a `limit` clause is used you may\
    \ see an error like this:\n\n```sql\nselect * from read_csv('./test-csv.py |')\
    \ limit 3;\n┌──────────┬──────────┐\n│ counter1 │ counter2 │\n│  int64   │  int64\
    \   │\n├──────────┼──────────┤\n│        0 │        0 │\n│        1 │        1\
    \ │\n│        2 │        2 │\n└──────────┴──────────┘\nTraceback (most recent\
    \ call last):\n  File \"/Users/rusty/Development/duckdb-shell-extension/./test-csv.py\"\
    , line 5, in <module>\n    print(f\"{i},{i}\")\nBrokenPipeError: [Errno 32] Broken\
    \ pipe\nException ignored in: <_io.TextIOWrapper name='<stdout>' mode='w' encoding='utf-8'>\n\
    BrokenPipeError: [Errno 32] Broken pipe\n```\n\nDuckDB continues to run, but the\
    \ program that was producing output received a SIGPIPE signal because DuckDB closed\
    \ the pipe after reading the necessary number of rows.  It is up to the user of\
    \ DuckDB to decide whether to suppress this behavior by setting the `ignore_sigpipe`\
    \ configuration parameter.\n"
  hello_world: '-- Generate a sequence only return numbers that contain a 2

    SELECT * from read_csv(''seq 1 100 | grep 2 |'');

    ┌─────────┐

    │ column0 │

    │  int64  │

    ├─────────┤

    │       2 │

    │      12 │

    │      20 │

    │      21 │

    │      22 │

    └─────────┘


    -- Get the first multiples of 7 between 1 and 3 5

    -- demonstrate how commands can be chained together

    SELECT * from read_csv(''seq 1 35 | awk "\$1 % 7 == 0" | head -n 2 |'');

    ┌─────────┐

    │ column0 │

    │  int64  │

    ├─────────┤

    │       7 │

    │      14 │

    └─────────┘


    -- Do some arbitrary curl

    SELECT abbreviation, unixtime from

    read_json(''curl -s http://worldtimeapi.org/api/timezone/Etc/UTC  |'');

    ┌──────────────┬────────────┐

    │ abbreviation │  unixtime  │

    │   varchar    │   int64    │

    ├──────────────┼────────────┤

    │ UTC          │ 1715983565 │

    └──────────────┴────────────┘

    '
extension:
  build: cmake
  description: Allow shell commands to be used for input and output
  excluded_platforms: wasm_mvp;wasm_eh;wasm_threads
  language: C++
  license: MIT
  maintainers:
  - rustyconover
  name: shellfs
  requires_toolchains: python3
  version: 1.0.0
repo:
  github: rustyconover/duckdb-shellfs-extension
  ref: 651981d540027681c06252d433e5590370131444

extension_star_count: 53

---

### Installing and Loading
```sql
INSTALL {{ page.extension.name }} FROM community;
LOAD {{ page.extension.name }};
```

{% if page.docs.hello_world %}
### Example
```sql
{{ page.docs.hello_world }}```
{% endif %}

{% if page.docs.extended_description %}
### About {{ page.extension.name }}
{{ page.docs.extended_description }}
{% endif %}

### Added Settings

<div class="extension_settings_table"></div>

|      name      |  description   | input_type | scope  |
|----------------|----------------|------------|--------|
| ignore_sigpipe | Ignore SIGPIPE | BOOLEAN    | GLOBAL |



---
