---
title: OpenWayback CDXJ File Format
type: specification
status: proposed
version: 1.0
numbered: true
latest: true
version-of: openwayback-cdxj-format
---

Preamble
========

This specification covers the CDXJ file format used by OpenWayback 3.0.0 (and later) to index web archive contents (notably in 
WARC and ARC files) and make them searchable via a resource resolution service.

The format builds on the CDX file format originally developed by the Internet Archive for the indexing behind the WaybackMachine. 
This specification builds on it by simplifying the primary fields while adding a flexible JSON 'block' to each record, allowing 
high flexiblity in the inclusion of additional data.

The use of a JSON in this manner is not novel. This specification is focused on enumarating the exact fields *outside* the JSON
and creating a list of the most common JSON fields for cross compatibility reasons. 

For the purposes of this document, CDXJ will be understood to refer to this particular specification.

While we recognize that this format may have wider application (e.g. for data exchange) the primary purpose of this specification is
to establish a file format suitable for creating a simple, URI keyed, index to the resources of a web archive.


File Specification
==================

Each file is a plain text file, UTF-8 encoded. It should end each line with Unix style newline character (0x0A).

A CDXJ file that has been sorted can be refered to as a CDXJ *index* as it is easily searchable.


## Header / Format Version

Each file should begin with a line declaring the file format and file format version. This line is preceeded with a bang symbol 
(`!` - 0x21) so that it naturally sorts to the front of the file.

Example:

```
!OpenWayback-CDXJ 1.0
```

This line may be repeated any number of times, as long as they are all sequential, starting from the first line of the file. 
The is to accomodate the merging of multiple CDXJ files that may be generated at different times.

It is permissible to mix minor version numbers (e.g. `1.0` and `1.1`) in the same file as minor versions are required to be backwards 
compatible. In this scenario, parsing software should treat the entire file based on the highest observed version number.

It is not permissible to mix major version numbers (e.g. `1.0` and `2.0`) in the same file. It is understood that an increase in
the major version number indicates a change that is not backwards compatible. It is not possible to merge CDXJ files with 
different major version numbers.

Lines beginning with the bang symbol may *only* appear at the top of the file, regardless of whether the file has been sorted.


Resource Entries / Records
----------------

Following the header lines, each additional line should represent exactly one resource in a web archive. Typically in a WARC (ISO 28500) or ARC file, although the exact storage of the resource is not defined by this specification. Each such line shall be refered to as a *record*.


Field Specification
===================

Each line, or record, is composed of four (4) fields. 

The fields are seperated by spaces (0x20). Consequently, spaces may not appear in the fields, except for the last field (JSON block).

Additionally, only the last (JSON block) field may begin with an opening curly brace (`{` - 0x7B).

The first three fields are collectively known as the *sortable* fields.


Searchable URI
--------------

The first field is a searchable version of the URI that this resource refers to.

By *searchable*, we mean that the following transformations have been applied to it:

1. Canonicalization - See Appendix A
2. Sort-friendly URI Reordering Transform (SURT) - See Appendix B
3. The scheme is dropped from the SURT format

**Note:** While this is extremly similar to other CDX and CDXJ implementations, we note that a lot of them get the SURT format wrong. 
Most notably by omitting the starting parenthesis or dropping the trailing comma in the domain name.

E.g. in using OpenWayback 2's CDX server the URL `http://example.com/' would translate to:

```
com,example)/
```

The correct SURT transformation is ...

```
(com,example,)/
```

... once you include the third step of dropping the scheme. 

The first field may not begin with a bang character (`!` - 0x21). Only header lines may begin with this character.

For URI's containing an *Internationalized domain name* (IDN), this field should always use the IDN format and not the *Punycode* 
representation.


Timestamp
---------

The second field is a timestamp. It should correspond to the WARC-Date timestamp as of WARC 1.1.

> A UTC timestamp as described in the W3C profile of ISO8601 [W3CDTF].
> The timestamp shall represent the instant that data capture for record
> creation began. [...]
> 
> WARC-Date may be specified at any of the six levels of granularity
> described in [W3CDTF]. If WARC-Date includes a decimal fraction of a
> second, the decimal fraction of a second shall have a minimum of 1
> digit and a maximum of 9 digits. WARC-Date should be specified with as
> much precision as is accurately known. 

This is a notable departure from the original CDX format, that used a somewhat truncated timestamp (YYYYMMDDhhmmss). The level of accuracy
of the timestamp should match the accuracy that is available in the WARC (or other source material).

**Note:** All timestamps should be in UTC.

In general, this field is equivalent to the WARC-Date field of a WARC record.


Record Type
-----------

Indicates what type of record the current line refers to. This field is fully compatible with WARC 1.0 definition of 
WARC-Type (chapter 5.5 and chapter 6). 

For content not stored in WARCs, a reasonable equivalent should be chosen.

E.g. 

* **response** - Suitable for any record that contains the response from a server to a specific request (irrespective of protocol).
* **request** - Suitable for any record containing a request made to a server.
* **revisit** - Suitable for any record of a response from a server to a specific request, where the content body is equal to that of another record.


JSON block
----------

The fourth and final field is a single line JSON block. This should contain fully valid JSON data. The only limitation, beyond those 
imposed by JSON encoding rules, is that this may not contain any newline characters, either in Unix (0x0A) or Windows form (0x0D0A). The 
first occurance of a 0x0A constitutes the end of this field (and the record).

The order of key/value pairs within the JSON block is unspecified. It is not expected that records can be sorted, in any way, on the JSON 
block field.

It is legal to store any amount of data in the JSON block. The following keys, however have a defined meaning. Further, some of these
fields are required.

Defined JSON keys:

* **uri** (*required*) - The value should be the non-transformed URI used for the searchable URI (first sortable field).
* **sha** (*recommended*) - A Base32 encoded SHA-1 digest of the payload that this record refers to. Omit if the URI has no intrinsic payload. For revisit records, this is the digest of the original payload. The algorithm prefix (e.g. `sha-1`) is not included in this field. See **dig** for alternative hashing algorithms.
* **dig** - A Base32 encoded output of a hashing algorithm applied to the URI's payload. This should include a prefix indicating the algorithm.
* **hsc** - HTTP Status Code. Applicable for *response* records for HTTP(S) URIs.
* **mct** - Media Content Type (MIME type). For HTTP(S) *response* records this is typically the "Content-Type" from the HTTP header. This field, however, does not specify the origin of the information. It may be used to include content type that was derived from content analysis or other sources.
* **ref** (*required*) - A URI that resolves to the resource that this record refers to. This can be any well defined URI scheme. For the most common web archive use case of warc filename plus offset, see Appendix C. For other use cases, existing schemes can be used or new ones devised.
* **rid** (*recommended*) - Record ID. Typically WARC-Record-ID or equivalent if not using WARCs. In a mixed environment, you should ensure that record ID is unique.
* **cle** - Content Length. The length of the content (uncompressed), ignoring WARC headers, but including any HTTP headers or similar.
* **ple** - Payload Length. The length of the *payload* (uncompressed). The exact meaning will vary by content type, but the common case is the length of the document, excluding any HTTP headers in a HTTP response record.
* **rle** - Record Length. The length of the record that this line refers to. This is the entire record (including e.g. WARC headers) as written on disk (compressed if stored compressed).
* **rct** - Record Concurrant To. The record ID of another record that the current record is considered to be 'concurrant' to. See further WARC chapter 5.7 (WARC-Concurrent-To).
* **rou** (*recommended*) - Revisit Original URI. Only valid for records of type *revisit*. Contains the URI of the record that this record is considered a revisit of.
* **rod** (*recommended*) - Revisit Original Date. Only valid for records of type *revisit*. Contains the timestamp (equivalent to sortable field #2) of the record that this record is considered a revisit of.
* **roi** - Revisit Original record ID. Only valid for records of type *revisit*. Contains the record ID of the record that this record is considered a revisit of.

Additionally, the three sortable fields can be redundantly stored in the JSON block, if so desired using the following keys:

1. Searchable URI - **ssu** (**s**ortable **s**earchable **U**RI)
2. Timestamp - **sts** (**s**ortable **t**ime**s**tamp)
3. Record Type - **srt**  (**s**ortable **r**ecord **t**ype)

To reduce the possibility of incompatibility with future versions of CDXJ, the use of custom keys longer than 3 characters is recommended.

Alternatively, new keys may be discussed and agreed upon by the OpenWayback community prior to a revised CDXJ version being released. 
Allowing for quicker implementation of additional keys.


Sorting File / Index
====================

A sorted CDXJ file is considered a CDXJ *index*, allowing lookup on URI (and timestamp) using simple binary searches. The CDXJ structure 
is designed to facilitate this. A non-sorted CDXJ file is still valid as far as this specification goes. It is just not usable for
searching. 

When sorted, the sorting should be done based on the native byte values of the characters. This is equivalent to using the GNU sort with 
the collation settings `LC_ALL=C`.

The structure of the CDXJ is designed so that using common sorting tools (e.g. GNU sort utility) works as expected. Provided that the correct collation settings are set as described above.


Appendices 
==========

Appendix A | Canonicalization
-----------------------------

Canonicalization is applied to URIs to remove trivial differences in the URIs that do not reflect that the URI reference different 
resources. Examples include removing session ID parameters, unneccessary port declerations (e.g. :80 when crawling HTTP).

OpenWayback implements its own canonicalization process. Typically, it will be applied to the searchable URIs in CDXJ files. You can, 
however, use any canonicalization scheme you care for (including none). You must simple ensure that the same canonicalization process is 
applied to the URIs when performing searches. Otherwise they may not match correctly.


Appendix B | Sort-friendly URI Reordering Transform (SURT)
----------------------------------------------------------

SURT is a transformation applied to URIs which makes their left-to-right representation better match the natural hierarchy of domain 
names.

A URI `<scheme://domain.tld/path?query>` has SURT form `<scheme://(tld,domain,)/path?query>`.

Conversion to SURT form also involves making all characters lowercase, and changing the 'https' scheme to 'http'. Further, the '/' after 
a URI authority component -- for example, the third slash in a regular HTTP URI -- will only appear in the SURT form if it appeared in 
the plain URI form.


Appendix C | 'warcfile' URI Scheme
----------------------------------

The 'warcfile' URI scheme shall be assumed to have the following structure:

```
warcfile:<warc-filename>#<offset>
```

The `<warc-filename>` is the full filename of the WARC (including any suffixes) but excluding any path information. WARC filenames are 
assumed to be unique.

The `<offset>` is the number of bytes from the start of the WARC when the relevant record begins.

Example URI:

```
warcfile:IAH-20070824123353-00393-heritrix2.nb.no.arc.gz#25523382
```

To fully resolve a URI with this scheme, an index mapping WARC filenames to specific locations is needed.


Appendix D | Example CDXJ File
------------------------------

```
!OpenWayback-CDXJ 1.0



```
