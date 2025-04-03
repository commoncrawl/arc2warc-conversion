Conversion of Common Crawl ARC Files to WARC
============================================

In this project we want to share our experiences with converting
Common Crawl's older ARC archives to WARC:
- issues observed during the conversion
- tools used for the conversion
- tests to verify the resulting WARC files


## Background

The first three crawls run by the Common Crawl Foundation (CCF) used
the ARC file format as primary archive format for web captures. The
ARC files have been written with varying crawler software. In total,
there are 130 TiB ARC data:

- May 2008 - Jan 2009 (`crawl-001`), Nutch
  - 12 TiB ARC files
- July 2009 - Sept 2010 (`crawl-002`), Nutch
  - 29 TiB ARC files
- Jan 2012 - June 2012,
  [commoncrawl-crawler](https://github.com/commoncrawl/commoncrawl-crawler),
  - 89 TiB ARC files
  - text and metadata extracts (Hadoop sequence files)

The conversion of ARC data to the WARC format is motivated by
- low (and further dropping) support for the ARC file format by data
  and text processing tools
- bugs and glitches in the ARC files written by CCF's crawler software

Several format issues in the ARC files were detected in 2019 when the
ARC data was indexed using [warcio](https://github.com/webrecorder/warcio)
and [PyWB](https://pywb.readthedocs.io/en/latest/). The indexing succeeded
after some modifications and work-arounds were made.

Mostly open questions are
- the conversion and transfer of metadata from ARC to WARC, both on the record and file level ("warcinfo" record)
- crawl/fetch metadata stored in HTTP headers
- and the required rewriting of HTTP headers
- whether to convert file by file or to group 10 ARC files into one WARC to meet the [1 GB WARC files size recommendation](https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/#annex-c-informative-warc-file-size-and-name-recommendations)) (100 MB for ARC)


## References and Documentation

### The 2008 - 2012 Archives

- 2008 - 2012 crawls
  - location and data formats: <https://commoncrawl.atlassian.net/wiki/spaces/CRWL/pages/2850886/About+the+Data+Set>
- 2012 crawl - numbers and statistics:
  - <https://commoncrawl.atlassian.net/wiki/spaces/CRWL/pages/4292610/Data+Set+Size+Statistics+-+2012>
  - <https://commoncrawl.org/blog/startup-profile-swiftkeys-head-data-scientist-on-the-value-of-common-crawls-open-data>
  - <https://commoncrawl.org/blog/a-look-inside-common-crawls-210tb-2012-web-corpus>
  - <https://docs.google.com/file/d/1_9698uglerxB9nAglvaHkEgU-iZNm1TvVGuCW7245-WGvZq47teNpb_uL5N9/edit>


### The ARC File Format

- <https://archive.org/web/researcher/ArcFileFormat.php>
- <https://en.wikipedia.org/wiki/Heritrix#Arc_files>


### The WARC File Format

- <https://en.wikipedia.org/wiki/Web_ARChive>
- <https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/>


### ARC to WARC Conversion Software

- warcio <https://warcio.readthedocs.io/en/latest/#arc-files>
- jwarc <https://github.com/iipc/jwarc>

Principally, any software able to read ARC and write WARC can be
used. We focus on warcio and jwarc.


### WARC Validation Software

(if not listed as ARC-to-WARC conversion software)

- FastWARC <https://resiliparse.chatnoir.eu/en/latest/api/fastwarc.html>
- `warc-tiny` https://github.com/JustAnotherArchivist/little-things>

Feature matrix (March 2025):

|                         | warcio | jwarc | FastWARC | warc-tiny |
| ----------------------- | :----: | :---: | :------: | :-------: |
| Software version        |  1.7.5 | 0.31.1|   0.15.1 |  579d589  |
|                         |        |       |          |           |
| Validates also ARC      |    ✔   |    ✔  |     ✘    |    ✘      |
|                         |        |       |          |           |
|                         |        |       |          |           |
| WARC-Block-Digest       |    ✔   |    ✔  |     ✔    |     ✔     |
| WARC-Payload-Digest     |    ✔   |    ✔  |     ✔    |     ✔     |
| WARC-Target-URI         |    ✘   |    ✘  |    ✘     |     ✘     |
|                         |        |       |          |           |
|                         |        |       |          |           |



## Format Issues in Common Crawl ARC Files

### Issues Discovered 2019 While Indexing ARC Data

1. invalid URIs causing SURT canonicalization to fail
   - seen in `crawl-002` (2009/2010)
   - examples
     ```
     http://www.babelicious.com%3Fnats=stiff7788:partner:BBLCS,0,0,0,0
     http://www.insuranceforpets.net]www.insuranceforpets.net/
     ```
   - fixed in `pywb/utils/canonicalize.py`
     1. try to fix the URL ([2103a7d](https://github.com/commoncrawl/pywb/commit/2103a7da02fd8e90e21a796095ad972ed9f14af4))
     2. do not fail but skip ([6dc9c39](https://github.com/commoncrawl/pywb/commit/6dc9c395201be219c89862d72f62e4261cb497fb))
   - test resources
     - `test/resources/arc/crawl-002_2009_09_17_12_1253241189984_12-4827319.arc.gz`
       (`s3://commoncrawl/crawl-002/2009/09/17/12/1253241189984_12.arc.gz`, offset 4827319, length 4260)
     - `test/resources/arc/crawl-002_2010_02_16_114_1266352769711_14-7060652.arc.gz`
       (`s3://commoncrawl/crawl-002/2010/02/16/14/1266352769711_14.arc.gz`, offset 7060652, length 1959)

2. white space in URLs breaks ARC header
   - seen in 2012 crawl
   - fixed in `warcio/recordloader.py`
     ([a8a0014](https://github.com/commoncrawl/pywb/commit/a8a0014408aeda258eba8143f7ae18a279b515a3))
   - notes:
     - the URL spec ([RFC 1738](https://datatracker.ietf.org/doc/html/rfc1738))
       and the living [WHATWG URL standard](https://url.spec.whatwg.org/) do not allow white
       space in URLs. However, the [Java URL class](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/net/URL.html)
       does not complain, while the [Java URI class](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/net/URI.html)
       does. Such, white space in URLs is a common issue in Java-based crawlers.
     - while the ARC descriptions use "URLs", the WARC spec requires a URI ([WARC-Target-URI](https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/#warc-target-uri))
     - test resources
       - `test/resources/arc/crawl-2012_1341690165832_1341699469441_1478-7224105.arc.gz`
         (`s3://commoncrawl/parse-output/segment/1341690165832/1341699469441_1478.arc.gz`, offset 7224105, length 10280)
   
3. wrong content-length in ARC header
   - seen in 2012 crawl
   - reported and discussed: <https://groups.google.com/forum/#!topic/common-crawl/P40niQBb8GY>
   - fixed in
     - `pywb/indexer/archiveindexer.py` ([fd4ace1](https://github.com/commoncrawl/pywb/commit/fd4ace13e4c61bbd10034e7b2233c53b7d69fa4b))
     - `warcio/archiveiterator.py` ([b431f7d](https://github.com/commoncrawl/pywb/commit/b431f7d23a5514186b116a25cafb943f2d57b83c))
   - test resources
     - `test/resources/arc/crawl-2012_1341690165636_1341785606830_6-0-4421.arc.gz`
       (`s3://commoncrawl/parse-output/segment/1341690165636/1341785606830_6.arc.gz`, offset 0, length 4421)

Note: issues 2 and 3 have already been fixed in 2012 after they where
reported on CCF's discussion group.  However, the policy was to keep
the erroneous ARC files in place but to not include them in a list of
"valid ARC files".  Erroneously, also the outdated ARC files were
tried to index in 2019 in the first pass. That way the issues were
"rediscovered" in 2019.



## Commands to Convert an ARC into a WARC File

If not specified otherwise, all commands in this and following section use these shell variables:
```bash
ARC_FILE=test/resources/arc/crawl-002_2010_02_16_114_1266352769711_14-7060652.arc.gz
WARC_FILE=test/output/warc/$(basename $arc_file .arc.gz).warc.gz
```

- warcio
  ```
  warcio recompress --verbose $ARC_FILE $WARC_FILE
  ```


## Commands to Validate WARC Files

- `warcio check --verbose $WARC_FILE`
- `fastwarc check --verify-payloads $WARC_FILE`
- `java -jar jwarc-0.31.1.jar validate --verbose $WARC`
- `warc-tiny verify $WARC`


## ARC and WARC Metadata

- file-level metadata
  - [warcinfo record](https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/#warcinfo)
  - cf. <https://groups.google.com/g/warc-tools/c/YKInCZg2BGw>
  - field mappings from ARC `filedesc` to `warcinfo`
  - ARC-to-WARC conversion metadata
    - see [warc-specifications#52](https://github.com/iipc/warc-specifications/issues/52) "WARC-Conversion-Software and WARC-Conversion-Command fields"

Metadata stored in HTTP headers of ARC records
- crawler content limit / truncated payload
  - [WARC-Truncated header](https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/#warc-truncated)
  - cf. <https://commoncrawl.org/errata/content-is-truncated>
  - poorly documented for CCF's ARC crawls
    - <https://groups.google.com/d/topic/common-crawl/hQTRnWahcHA/discussion>
    - 2 MiB or 500 kiB?
    - marked by HTTP header `x-commoncrawl-ContentTruncated` in ARC files
      - values: `TruncatedInDownload` and `TruncatedInInflate` (can be combined)
- identified page encoding
  - ARC: in HTTP header `x-commoncrawl-DetectedCharset`
  - WARC: in [WARC metadata record](https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/#metadata)
- ...?


## Required Rewriting of HTTP Headers

CCF's ARC and WARC files store the payload with content and transfer encodings removed.
However, in the ARC files the HTTP headers `Content-Encoding` and `Transfer-Encoding` are preserved, e.g.
```
Content-Encoding:gzip
Transfer-Encoding:chunked
```

This was also the case for CCF WARC files written in 2013 – 2016/2018 and caused troubles with WARC parsers trying to decode the content or transfer encoding:
- [Common Crawl saves gzipped body in extracted form](https://groups.google.com/g/common-crawl/c/XiLLXX1KSUs/m/anuZq8FCCgAJ), fixed in [commoncrawl/nutch@3551eb6](https://github.com/commoncrawl/nutch/commit/3551eb6dbb7f7152a13d2e4eb0f8eb6014dc8252)
- finally addressed in 2018 ([August 2018 crawl](https://commoncrawl.org/blog/august-2018-crawl-archive-now-available)):
  - the original fields `Content-Encoding`, `Transfer-Encoding` and `Content-Length` are preserved using the prefix `X-Crawler-`
  - the length of the payload after decoding is saved in a new `Content-Length` header
- see also: [warc-specifications#22](https://github.com/iipc/warc-specifications/issues/22) "Clarify whether Transfer-Encoding can or should be preserved"
