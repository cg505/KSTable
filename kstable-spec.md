# KSTable specification

Version 0.0

## Motivation

[K-Shoot MANIA](https://www.kshootmania.com/) and [Unnamed SDVX Clone](https://github.com/Drewol/unnamed-sdvx-clone) are two games that provide a similar gameplay experience and use [the same charting format](https://github.com/m4saka/ksh/blob/master/ksh_format.md). There are many community-made charts for these games (which we will call KSM charts for simplicity). KSM charts include difficulty levels assigned by the chart author - but singular ratings from 1-20 are often insufficient or inaccurate in describing the difficulty of a chart.

BMS has solved a similar problem by introducing "difficulty tables". A difficulty table is a centralized set of chart ratings. The table itself does not define the charts - they usually provided elsewhere for download, and one chart can have multiple different chart ratings in various tables.

Some difficulty tables have already been established for KSM charts - for instance, the [os/us tables](https://docs.google.com/spreadsheets/u/1/d/e/2PACX-1vRH1beDh3I76qS-UUppW4_GibEj9bBDgcpO1XM4SrMylnSZYjyPjPYcMuQHaMrB-JGpIdsZtb-0wzmp/pubhtml) or the older [insane KSM table](https://docs.google.com/spreadsheets/d/1vdYPSEtdv1ngOUuspoKdOndFlFtFXjHbAeND_ujpock/htmlview).

BMS difficulty tables have an advantage over the current KSM chart tables: they abide by a [well-defined data specification](https://bmsnormal2.syuriken.jp/bms_dtmanager.html) that allows clients to display difficulty tables in games and IR servers to display table ratings alongside rankings. KSM charts currently have no such specification, and the aforementioned tables are simply spreadsheets that players can reference outside the game.

This document seeks to define a spec that serves a purpose similar to the BMS difficulty table spec.

### Prior work

This spec should be able to represent the existing tables mentioned above (os/us and insane). Examples from these tables will be drawn throughout.

For os/us, there is an existing [KSM score tool](https://ux.getuploader.com/ksmscoretool/download/5) which contains the table data and is able to create corresponding collections in KSM. However, this tool has its own limitations, mainly that

- table data is packaged with the tool, so any updates to the table require a new release of the tool, and

- it only supports KSM and does not make its data source available to other clients (USC) or other uses such as IR servers.

The KSTable spec should be usable by this tool or a similar one in the future without such limitations.

### Clarifications

- An "optional" key may be omitted, or may be present with a value of `null`. Any key that is not marked as optional is required and must not have a value of `null`.

## What is a KSTable?

A KSTable is a JSON representation of a related series of table levels. For instance, os or us could each be represented as a KSTable, and would have table levels such as os5 or us17.

> Note: The JSON spec requires JSON strings to use Unicode encoding. KSTable JSON should use UTF-8 and must not have a BOM.

A KSTable is always a single JSON object which represents all information about the difficulty table. It should be stored in a single file or at a single HTTP endpoint.

<details>
<summary>Design decision: single file</summary>

BMS tables split the table information into two files: the header and the data. The header essentially contains metadata about the table and levels. The data contains the list of charts with their assigned table level.

As far as I can tell, this is mostly useful because initially a client can fetch only the header to get basic info about the table, then fetch the data lazily.

In practice, the data file for large BMS tables is no larger than a few MiB, so there is not a huge gain to loading lazily. The complexity of maintaining multiple files is not worth this benefit.
</details>

Unless otherwise specified below, a KSTable author must not include JSON keys are not defined as part of the spec. However, a consumer parsing a KSTable should be permissive of unexpected keys, in case the KSTable abides by a newer version of the spec.

The top-level KSTable object follows this schema:

```
{
  "name": string,
  "prefix": string,
  "url": string?,
  "version": {
    "breaking": number,
    "minor": number
  },
  "meta": {
    "homepage": string?,
    "description": string?,
    "updated": number?
  },
  "levels": Level[]
}
```

- `name` is the humanized name of the table.

- `prefix` is the prefix used before table level names, e.g. "us" or "●".

- `url` is the direct URL from which this JSON can be fetched. It is an optional parameter and should be omitted when the KSTable is not available at a consistent URL.

  A GET request to this URL must return the most recent version of this KSTable, and should set the HTTP header `Content-Type: application/json`.

  <details>
  <summary>Design decision: canonical URLs</summary>

  In BMS tables, the canonical URL of a table is a human-readable HTML page which includes a `<meta>` element pointing to the header JSON URL. This is kind of nice because it lets clients and humans use the same URL to reference the table. However, in practice it is a massive headache for clients, since they now have to use both HTML and JSON parsing.

  Since the canonical URL defined here is _not_ human readable, the spec defines a different URL in `meta.homepage` which should serve that purpose. This is a minor usability annoyance for end-users but makes client implementation drastically simpler.
  </details>

- `version` is an object corresponding to the version listed at the very top of this document. For version `x.y`, `breaking` is `x` and `minor` is `y`. The idea is that any backwards-compatible "minor" change to the specification will increment the "minor" version, but any breaking changes to the specification must increment the "breaking" version and reset the minor version to 0.

  KSTable authors must always use the exact version of the specification that their KSTable implements. For this version of the spec, that is `{"breaking": 0, "minor": 0}`. KSTable authors should always implement the latest version of the KSTable spec available in this document in this repository.

  KSTable consumers should accept any KSTable with the same breaking version and a greater minor version than the consumer implements. Consumers can also choose to implement support for multiple versions.

  New keys can be introduced to the spec by a minor version bump. This is why it's important for consumers to ignore unknown keys unless otherwise specified in the spec.

  > Note: While the breaking version is 0, breaking changes will be allowed with only a minor version bump. This will be removed with the release of version 1.0.

  > Note: Clarifications or changes in this document that do not materially change the definition of the specification do not require any version bumps. Any change to the actual defined schema or permitted behavior requires a version bump.

  > Note: The `version` key and its schema will _never_ change in future versions of the spec.

- `meta` is an object providing additional metadata about the table:

  - `homepage` is an optional URL for a human-readable website about the table. This should not be the same as `url` above.

  - `description` is an optional human-readable description of the table.

  - `updated` is the date the table was last modified, represented as an integer UTC unix time (number of seconds since the unix epoch). It is optional, but KSTable authors should not remove it between versions of the same table.

  KSTable authors can include additional keys here, but should strive to update the spec to include any additional useful keys so that they become standardized.

- `levels` is an array defining the different table levels. The object schema for each level is defined below. The array is ordered in terms of increasing difficulty. (That is, the easiest level should come first and the hardest level should come last.)

<details>
<summary>Design decision: courses</summary>

This specification does not support the definition of courses like BMS tables do. Some KSM tables do have courses (e.g. os/us "Shooter's License") but in general it doesn't make a lot of sense that these courses should be strictly tied to table definition. It's possible that a future version of the spec will support courses.
</details>

## Levels

A table level is represented by the following schema:

```
{
  "name": string,
  "meta": {
    "description": string?,
    "unique": boolean?
  },
  "charts": Chart[]
}
```

- `name` is the name of the table without the `prefix`. For instance, if the KSTable sets `prefix` as `"●"`, then a level with the name `"2"` defines the table level ●2.

  > Note: This must not be a number. Even if the name of the table is in fact a number, it must be represented as a string.

- `meta` is an object providing additional metadata about the table level:

  - `description` is an optional human-readable description or short name for this table level.

  - `unique` is an optional boolean that indicates whether or not this table level is for charts that are non-standard in some way. If this is key is provided for any level in a KSTable, it should be provided for all levels in that KSTable.

    This is used by the os/us tables for levels such as "us4*".

  KSTable authors can include additional keys here, but should strive to update the spec to include any additional useful keys so that they become standardized.

- `charts` is an unsorted array of chart objects that fall under this table level. The schema for a chart object is defined below.

<details>
<summary>Design decision: level numbers</summary>

One addition to this schema that was considered is a numerical representation of the difficulty of the level. However, it's not always clear what is a good way to assign these numbers. For instance, should os1* have a larger number than os1? What number should be assigned to us0*?

Instead, we just avoid the issue entirely. For rudimentary purposes, the ordering of the levels should be sufficient. If such a value is desired in the future, it can probably be added to `meta`.
</details>

## Charts
Charts form the fundamental building block of a table. When this spec says "chart", it means an _individual difficulty_ (for instance, magical*kittens Challenge). A song can have multiple charts, for Light, Challenge, Extended, or Infinite, and for clarity we call this set of charts a "chart song". (It's important to remember that one musical song can have multiple chart songs.)

The chart object schema is meant to provide a reliable way to identify a specific chart. It is defined as:

```
{
  "title": string,
  "artist": string,
  "chart_author": string,
  "difficulty_index": number,
  "chart_level": number,
  "hashes": {
    "chart_file_sha1": string
  },
  "download_url": string?,
  "pack": {
    "name": string,
    "dir": string
  }?,
  "sabun_pack": string?,
  "sabun_download_url": string?
}
```

- `title` is the song title as defined in the chart.

  > Note: Even if it has a mistake (e.g. Agitatus in B4UT SUMMER DIARY 2017 which has an erroneous leading space), the title here must match exactly the title defined in the chart. Otherwise, you cannot use the title to identify the chart. The same is true for artist and effector.

- `artist` is the song artist as defined in the chart.

- `chart_author` is the chart author (ksh `effect` option) as defined in the chart.

- `difficulty_index` is an integer between 0 and 4 (inclusive).

  For KSH charts, the value is 0 for `difficulty=light`, 1 for `difficulty=challenge`, 2 for `difficulty=extended`, and 3 for any other difficulty value.

  For KSON charts, this is exactly the value of `meta.difficulty.idx`.

  <details>
  <summary>Design decision: using difficulty index</summary>

  This specification should be resilient to the future, so we look to the KSON spec to determine the best way to handle difficulty. KSON has [an entire object dedicated to specifying the difficulty of a chart](https://github.com/m4saka/ksh2kson/blob/master/kson_format.md#metadifficulty). Realistically, for identification purposes, we are not too concerned with the difficulty name. In fact nearly every chart uses one of `light`, `challenge`, `extended`, or `infinite`, so the difficulty index value is probably sufficient.

  The logic for determining the index from the KSH difficulty value is taken [from ksh2kson](https://github.com/m4saka/ksh2kson/blob/c357a89b43464323cb3e273abdcd071019082154/src/main.cpp#L49-L67).
  </details>

- `chart_level` is an integer between 1 and 20 (inclusive) corresponding to the level as defined in the chart. This is _not_ related to the table level.

- `hashes` defines hashes that can be used to identify the chart. Currently, there is only one.

  > Note: Using a hash is the _only_ reliable way to identify a chart. You may expect other combinations of fields to be sufficient, for instance title+artist+pack+difficulty+level. You would be surprised to find that #curtaincall by sak + wellow has two Infinite 20 charts in B4UT×KUOC K-shoot mania Package 2016 - one is os5 and the other is os9*.

  - `chart_file_sha1` is the SHA1 hash of the unmodified original chart file, represented as a lowercase hexidecimal string of length 40. "The unmodified original chart file" here refers to the KSH or KSON file for the chart.

    > Note: Any changes to the chart file, including to "unimportant" fields such as `illustrator`, will cause this hash to be invalid. Even changing the encoding of the file will break this hash.

  In future versions of the spec, other useful hashes may be added - for instance if the KSON spec defines a unique hash based on the chart data.

  <details>
  <summary>Design decision: hash selection</summary>

  Identifying charts by their hash is a tried and true method used by BMS tables. BMS uses md5, which is perfectly fine.

  KSM does not use hashes to identify charts internally. However, USC already uses the SHA1 hash of the chart file to identify charts in its internal database, so this makes the hash selection an obvious choice. USC IR also uses this SHA1 hash as the primary way to identify charts.

  KSM IR uses a [custom hashing method](https://github.com/m4saka/kshootmania-v1/blob/1c75880b545d1232eeffc4bb3fc19704a3622f73/src/ir.hsp#L46-L68) for KSH files that intentionally excludes some options (e.g. title, artist, etc), `#define`, and comments. This is a reasonable approach but still has some flaws - for instance, empty lines can still affect the hash.
  Since this hashing method is unique to KSM IR, is somewhat painful to implement, and does not support KSON, it's not included in the current spec.

  In the future, if some kind of good metadata-independent hash is defined for KSH and KSON, its adoption into this spec would make sense.
  </details>

- `download_url` is the URL for downloading this chart or the pack containing this chart. It is optional, but it should be defined if the chart is available for download. It doesn't need to point directly to the download, the homepage for the pack is also acceptable.

- `pack` represents the pack that this chart can be found in. It is optional, but it must not be defined if the chart was not released as part of a pack and it should be defined if the chart was released as part of a pack.

  It has the following fields, which are both required if `pack` is defined:

  - `name` is the name of the pack.

  - `dir` is the directory under the songs folder that the pack should use.

    <details>
    <summary>Design decision: why include dir?</summary>

    The `dir` field as defined here is basically what the os/us KSM score tool uses to find the right chart. It's not perfect, but the spec should have a superset of that dataset in order to be able to replace it.

    It's also worth noting that this is always well-defined. Every pack needs to reside in a specific folder, otherwise sabuns that refer to chart songs in that pack will not work.
    </details>

- `sabun_pack` is the name of the pack the chart requires if it's a sabun. It should be defined if the chart is a sabun that relies on another pack, and it must not be defined if it isn't a sabun.

  Here, sabun means that the chart refers to resources (such as the jacket or the audio) from a chart song that is downloaded separately. The chart will not work properly unless both the the original chart song and the sabun are downloaded.

  This should not be the same as `pack.name`.

- `sabun_download_url` is a URL that can be used to download the chart song that the sabun depends on. It should be defined if the chart is a sabun, and it must not be defined if it isn't a sabun.

  This should not be the same as `download_url`.
