# Notes From HLS Draft Spec

[HTTP Live Streaming](https://tools.ietf.org/html/draft-pantos-http-live-streaming-23)

This is not intended to replace above linked document. Expect errors,
simplifications, and editorialization.

## Overview of Playlists and Streams

* Playlists must be UTF-8
* Master Playlists
* Variant Streams
* Media Playlists
  * Single format, resolution, and bitrate within a media playlist

## Media Segments

* URI
* Optional Byte range
* Duration in required [`EXTINF`](#EXINF) tag
* Segments have a "Media Sequence Number"
 * First segment can be explicitly numbered, or an implicit 0
 * Subsequent segments are implicitly numbered one more than the preceding segment.
* Must be encoded as
  * [MPEG-2 Transport Stream](http://www.iso.org/iso/catalogue_detail?csnumber=44169)
  * [Web VTT](https://w3c.github.io/webvtt/) (Web Video Text Tracks)
  * Various audio + ID3 tag packing
* Media Initialization Section
  * MPEG streams Program Association Table (PAT) and Program Map Table (PMT)
    * may be provided with each segment
    * may be declared in the [`EXT-X-MAP`](#EXT-X-MAP) tag
  * WebVTT initialization is a WebVTT header
  * Packed audio has no media initialization
* Subtitle (VTT) segments are synchronized with timestamp tags _e.g._ `X-TIMESTAMP-MAP=LOCAL:<cue time>,MPEGTS:<MPEG-2 time>`, or assumed to start with corresponding MPEG segment.

## Playlists

Apparently, clients MUST fail to parse invalid playlists.

* Must be identifiable by one of two forms:
  * path component of URI ending
    * `m3u8`
    * `m3u`
  * HTTP `Content-type` MUST be `application/vnd.apple.mpegurl` or `audio/mpegurl`
* Clients _should_ fail to parse otherwise

There are some rules about bitrate. There seem to be 3 distinctly defined bitrate defitions:

1. Segment Bitrate
2. Peak Segment Bitrate
3. Average Segment Bitrate

## Playlist Definition

* Playlists _must_ be encoded in `UTF-8` with no `BOM` or control characters except `CR` and `LF`
* Lines end in single `LF` or `CRLF`
* No white space unless specified
* Each line is
  * `URI`
    * Media Segment
    * Playlist File
  * Blank (ignored)
  * starts with `#`
    * Tags start with `#EXT` and are case-sensitive
    * otherwise is a comment and ignored

## Playlist Types

There are two kinds of playlists. No other playlists are valid.

* Master Playlist
  * All URI lines identify media playlists
* Media Playlist
  * All URI lines identify media segments

Relative URIs are relative to the playlist that contains them.

## Attribute Lists

Attribute lists are comma-separated key value pairs with the format:

```
AttributeName=AttributeValue
```

`AttributeName` matches `[A..Z]`, `[0..9]` and `-`

`AttributeValue` is one of 7 types:

1. `decimal-integer` : `[0..9]`, in range 0 to 2^64-1 (implies 1 to 20 digits)
2. `hexidecimal-sequence` : `0x` or `0X` prefix with characters `[0..9]`, `[A..F]`. Max length variable.
3. `decimal-floating-point` `[0..9]` and `.`, non-negative decimal
4. `signed-decimal-floating-point` `[0..9]`, `-` and `.`
5. `quoted-string` : within a pair of quotes (`0x22`). Must not contain CR (`0xD`) or LF (`0xA`) or " (`0x22`). Equality is determined by bytewise comparison.
6. `enumerated-string` : unquoted string pre-defined by the `Attribute` it's used in. will never contain `"`, `'`, or whitespace.
7. `decimal-resolution` : two `decimal-integer` separated by `x` define a width and a height

`AttributeName` defines the allowed types for `AttributeValue`. An `AttributeName` must not appear more than once in an attribute list. Clients shouldn't play lists that break the rule.

## Playlist Tags

### `EXTM3U`

Necessarily the first line of a playlist.

### `EXT-X-VERSION`

```
EXT-X-VERSION:<n>
```

`n` is the compatibility version. Appears in all versions that aren't compatible with version 1. If more than one version tag appears in a playlist, the player must reject the playlist.

## Media Segment Tags

Some number of media segment tags appear before media segment URIs. The only appear in media playlists, and cannot appear in master playlists.

### `EXTINF`

Applies only to the next media segment.

```
#EXTINF:<duration>,[<title>]
```

* Required tag per segment
* `duration` is decimal float or integer
* compatibility < 3, `duration` must be integer
* `duration` should be accurate to not accumulate error as segments are played back
* `[]` denotes optional
* `title` is a human-readable title of the segment

### `EXT-X-BYTERANGE`

Applies only to the next media segment. Indicates the next segment is defined by a range in the resource indicated by its URI. If tag isn't present, the segment is the entire resource specified by the URI.

```
#EXT-X-BYTERANGE:<n>[@<o>]
```

* `n` a decimal integer indicating the length of the range in bytes
* `o` a decimal integer indicating the start of the subrange as a byte-offset
  * if not present, begins at the next byte following subrange of previous segment, and a previous resource must exist that is also a subrange of the same resource as the segment this tag applies to.

### `EXT-X-DISCONTINUITY`

Discontinuity must be present if there's a change in
* file format
* number, type and identifiers of tracks
* timestamp sequence
and should be present for changes in
* encoding parameters
* encoding sequence

### `EXT-X-KEY`

Specifies how to decrypt segments. Applies to all segments after the tag until another `EXT-X-KEY` appears.

```
#EXT-X-KEY:<attribute-list>
```

| Attribute   | Required | Type                 |                                          |
| ----------- | :------: | -------------------- | ---------------------------------------- |
| `METHOD`    | YES      | enumerated-string    | `NONE`, `AES-128`, or `SAMPLE-AES`       |
| `URI`       | NO*      | quoted-string        | Where to obtain the key. Required if method is not `NONE` |
| `IV`        | NO?      | hexidecimal-sequence | 128-bit initialization vector for use with the key. Compatibility version >= 2 |
| `KEYFORMAT` | NO       | quoted-string        | Version >= 5. How the key is represented. Absence is implicit `identity` |
| `KEYFORMATVERSIONS` | NO | quoted-string      | Version >= 5. Integers separated by "/", indicates version compatibility of key? |

If tag is missing, content is not encrypted.

See sections [4.3.2.4](https://tools.ietf.org/html/draft-pantos-http-live-streaming-19#section-4.3.2.4) and [5.2](https://tools.ietf.org/html/draft-pantos-http-live-streaming-19#section-5.2) for more

### `EXT-X-MAP`

Media initialization

```
#EXT-X-MAP:<attribute-list>
```
| Attribute   | Required | Type          |     |
| ----------- | :------: | :-----------: | --- |
| `URI`       | YES      | quoted-string | URI to the Media Initialization Section |
| `BYTERANGE` | NO       | quoted-string | Byterange quoted string to location of MIS in the resource specified by the URI |

### `EXT-X-PROGRAM-DATE-TIME`

Associates the first sample of the segment with an absolute date. Applies only to the next segment. The date/time is formatted in [ISO-8601](https://tools.ietf.org/html/draft-pantos-http-live-streaming-19#ref-ISO_8601) and should both indicate time zone and fractional seconds. It _should_ provide millisecond accuracy.

```
#EXT-X-PROGRAM-DATE-TIME:<YYYY-MM-DDThh:mm:SSSZ>
```

### `EXT-X-DATERANGE`

Associates tags with date ranges

```
#EXT-X-DATERANGE:<attribute-list>
```

| `AttributeName` | Required | Type  |     |
| --------------- | :------: | :---: | --- |
| `ID` | YES | `quoted-string` | Uniquely identifies the date-range in the Playlist |
| `CLASS` | NO | `quoted-string` | Client-defined? |
| `START-DATE` | YES | `quoted-string` | [ISO-8601](https://tools.ietf.org/html/draft-pantos-http-live-streaming-19#ref-ISO_8601) date |
| `END-DATE` | NO | `quoted-string` | [ISO-8601](https://tools.ietf.org/html/draft-pantos-http-live-streaming-19#ref-ISO_8601) date must be later than `START-DATE` |
| `DURATION` | NO | `decimal-floating-point` | Number of seconds, must not be negative. Can be 0 to represent a single instance |
| `PLANNED-DURATION` | NO | `decimal-floating-point` | Number of seconds, must not be negative. Can be 0 to represent a single instance. Does not specify a definite, known duration for the range. |
| `X-<client-attributes>` | NO |  `quoted-string`, a `hexadecimal-sequence`, or `decimal-floating-point` | Namespace for client attributes. Should use reverse-DNS syntax. Name bust be valid. |
| `SCTE35-CMD`
| `SCTE35-OUT`
| `SCTE35-IN`
| `END-ON-NEXT` | NO | | value must be YES. Indicates the range ends at the start of the range with the same `CLASS` with the earliest `START-DATE` after this range's `START-DATE`. Range _must_ have a `CLASS`, and cannot overlap with ranges with the same `CLASS`. Use of tag means you can't use `END-DATE` or `DURATION` |

#### SCTE-35

SCTE-35 is for splicing in ads. [spec](http://www.scte.org/documents/pdf/Standards/ANSI_SCTE%2035%202014.pdf)

## Media Playlist Tags

Media Playlist Tags describe global playlist parameters. A single tag must not appear more than once in a playlist, an no such tag can appear in a master playlist.

### `EXT-X-TARGETDURATION`

This tag is required.

```
#EXT-X-TARGETDURATION:<s>
```

`s` is a decimal-integer in seconds. All segments in the media playlist _must_ be less than or equal to `s` in length when rounded to the nearest integer.

### `EXT-X-MEDIA-SEQUENCE`

```
#EXT-X-MEDIA-SEQUENCE:<number>
```

`number` is a decimal-integer indicates the number of the first segment in the playlist. Matching numbers across playlists _do not_ indicate matching content. The absence of this tag indicates the first segment number is 0. If it _does_ appear, it must appear before the first media segment in the file. The URI does not need to contain its segment number.

### `EXT-X-DISCONTINUITY-SEQUENCE`

```
#EXT-X-DISCONTINUITY-SEQUENCE:<number>
```

The discontinuity sequence tag must appear before the first media segment in the playlist and before any `EXT-X-DISCONTINUITY` tag. If it's absent, discontinuity 0 is assumed.

### `EXT-X-ENDLIST`

No more media segments will be added to the playlist.

### `EXT-X-PLAYLIST-TYPE`

An optional tag applying to the whole file.

```
#EXT-X-PLAYLIST-TYPE:<EVENT|VOD>
```

If the type is `EVENT`, media segments can only be added to the end. If it's `VOD`, the playlist will not change.

### `EXT-X-I-FRAMES-ONLY`

Compatibility 4 or greater. A little unclear, but each media segment needs to contain an I-Frame. Facilitates or enables seeking.

## `Master Playlist Tags`

### `EXT-X-MEDIA`

```
#EXT-X-MEDIA:<attribute-list>
```

| Attribute  | Required | Type              |                                                            |
| ---------  | :------: | :---------------: | ---------------------------------------------------------- |
| `TYPE`     | YES      | enumerated-string | one of `AUDIO`, `VIDEO`, `SUBTITLES` and `CLOSED-CAPTIONS` |
| `URI`      | NO       | quoted-string     | Identifies the Media Playlist                              |
| `GROUP-ID` | YES      | quoted-string     | "group to which *Rendition* belongs"                       |
| `LANGUAGE` | NO       | quoted-string     | [RFC 5646](https://tools.ietf.org/html/rfc5646) Primary language used in the *Rendition* |
| `ASSOC-LANGUAGE` | NO | quoted-string     | [RFC 5646](https://tools.ietf.org/html/rfc5646) |
| `NAME`     | YES      | quoted-string     | Name of the *Rendition* |
| `DEFAULT`  | NO       | enumerated-string | `YES` or `NO`. If `YES`, client should play this media until user says otherwise. Absence is `NO` |
| `AUTOSELECT` | NO     | enumerated-string | `YES` or `NO`. Absence is implicit `NO`. Makes media eligible for automatic playback based on environment reasons (_e.g._ language). |
| `FORCED`   | NO       | enumerated-string | `YES` or `NO`. Absence is implicit `NO`. Must only appear for `TYPE=SUBTITLES`. `YES` means content is essential (_e.g._ Elvish?) |
| `INSTREAM-ID` | NO    | quoted-string     | For closed-captions. See spec |
| `CHARACTERISTICS` | NO | quoted-string    | comma-separated UTIs |
| `CHANNELS`  | NO      | quoted-string     | "/"-separated list of parameters. The first is the maximum number of independent audio channels. Should be present if `TYPE` is `AUDIO`. Required if a Master Playlist contains two renditions encoded with the same codec but a different number of channels. |

[Rendition groups](https://tools.ietf.org/html/draft-pantos-http-live-streaming-19#section-4.3.4.1.1) represent alternate rendition of the same content. They need the same `GROUP-ID`, different `NAME`, no more than one `DEFAULT` member, and `AUTOSELECT` members must have a `LANGUAGE` attribute.

### `EXT-X-STREAM-INF`

```
#EXT-X-STREAM-INF:<attribute-list>
<URI>
```

| Attribute     | Required  | Type                   |                                                            |
| ---------     | :-------: | :---------------:      | ---------------------------------------------------------- |
| `BANDWIDTH`   | YES       | decimal-integer        | Bits per second, represents the peak segment bit rate      |
| `AVERAGE-BANDWIDTH` | NO  | decimal-integer        | Bits per second |
| `CODECS`      | should be | quoted-string          | comma-separated list of formats [RFC6381](https://tools.ietf.org/html/rfc6381) |
| `RESOLUTION`  | NO        | decimal-resolution     | Recommended for video content |
| `FRAME-RATE`  | NO        | decimal-floating-point | Maximum frame rate for all video in the variant stream. Rounded to 3 decimal places. Should be present if video is ever >30fps |
| `HDCP-LEVEL`  | NO        | enumerated-string      | `TYPE-0` or `NONE`. Should be present if conten't won't play without [HDCP](http://www.digital-cp.com/sites/default/files/specifications/HDCP%20on%20HDMI%20Specification%20Rev2_2_Final1.pdf). Clients without copy protection shoudn't load a variant with `HDCP-LEVEL` that isn't `NONE`. |
| `AUDIO`       | NO        | quoted-string          | Must match GROUP-ID attribute of an EXT-X-MEDIA tag with an `AUDIO` type |
| `VIDEO`       | NO        | quoted-string          | Must match GROUP-ID attribute of an EXT-X-MEDIA tag with an `VIDEO` type |
| `SUBTITLES`   | NO        | quoted-string          | Must match GROUP-ID attribute of an EXT-X-MEDIA tag with an `SUBTITLES` type |
| `CLOSED-CAPTIONS` | NO    | quoted-string          | Must match GROUP-ID attribute of an EXT-X-MEDIA tag with an `CLOSED-CAPTIONS` type |

When an `EXT-X-STREAM-INF` tag contains an `AUDIO`, `VIDEO`, `SUBTITLES`, or `CLOSED-CAPTIONS` attribute, it indicates that alternative Renditions of the content are available for playback of that Variant Stream with some [constraints](https://tools.ietf.org/html/draft-pantos-http-live-streaming-19#section-4.3.4.2.1).

### `EXT-X-I-FRAME-STREAM-INF`

```
#EXT-X-I-FRAME-STREAM-INF:<attribute-list>
```

> The EXT-X-I-FRAME-STREAM-INF tag identifies a Media Playlist file
  containing the I-frames of a multimedia presentation.  It stands
  alone, in that it does not apply to a particular URI in the Master
  Playlist.

| Attribute     | Required  | Type                   |                                                            |
| ---------     | :-------: | :---------------:      | ---------------------------------------------------------- |
| `BANDWIDTH`   | YES       | decimal-integer        | Bits per second, represents the peak segment bit rate      |
| `AVERAGE-BANDWIDTH` | NO  | decimal-integer        | Bits per second |
| `CODECS`      | should be | quoted-string          | comma-separated list of formats [RFC6381](https://tools.ietf.org/html/rfc6381) |
| `RESOLUTION`  | NO        | decimal-resolution     | Recommended for video content |
| `URI`         | NO        | quoted-string          | Identifies media playlist file |

### `EXT-X-SESSION-DATA`

Allows session data to be carried in the master playlist.

```
#EXT-X-SESSION-DATA:<attribute-list>
```

| Attribute     | Required  | Type              |                                                            |
| ---------     | :-------: | :---------------: | ---------------------------------------------------------- |
| `DATA-ID`     | YES       | quoted-string     | Identifies the data value. Should use reverse DNS namespace to avoid collisions. |
| `VALUE`       | X `URI`   | quoted-string     | Value. If `LANGUAGE` is specified, value should be human-readable in that language. Tag must have either `VALUE` or `URL`. |
| `URI`         | X `VALUE` | quoted-string     | URI to a [JSON](https://tools.ietf.org/html/rfc7159) resource. Tag must have either `VALUE` or `URL` |
| `LANGUAGE`    | NO        | quoted-string     | [RFC5646](https://tools.ietf.org/html/rfc5646) |

More than one session tag with the same `DATA-ID` may be present as long as they don't have overlapping `LANGUAGE`.

### `EXT-X-SESSION-KEY`

```
#EXT-X-SESSION-KEY:<attribute-list>
```

| Attribute   | Required | Type                 |                                          |
| ----------- | :------: | -------------------- | ---------------------------------------- |
| `METHOD`    | YES      | enumerated-string    | `AES-128`, or `SAMPLE-AES`               |
| `URI`       | NO*      | quoted-string        | Where to obtain the key. Required if method is not `NONE` |
| `IV`        | NO?      | hexidecimal-sequence | 128-bit initialization vector for use with the key. Compatibility version >= 2 |
| `KEYFORMAT` | NO       | quoted-string        | Version >= 5. How the key is represented. Absence is implicit `identity` |
| `KEYFORMATVERSIONS` | NO | quoted-string      | Version >= 5. Integers separated by "/", indicates version compatibility of key? |

Should match a `EXT-X-KEY` with the same method, `KEYFORMAT`, and `KEYFORMATVERSIONS`. Should be present if multiple variants use same encryption keys.

## Media or Master Playlist Tags

These tags can only appear once in a playlist. They shouldn't appear in both a master and a media playlist. If they do, the values in the Master Playlist should apply, and those in the Media Playlist should be ignored.

### `EXT-X-INDEPENDENT-SEGMENTS`

Media segments are encoded independently.

### `EXT-X-START`

The preferred point to start playing a playlist.

```
#EXT-X-START:<attribute-list>
```

| Attribute     | Required | Type                          |                                          |
| ------------- | :------: | ----------------------------- | ---------------------------------------- |
| `TIME-OFFSET` | YES      | signed-decimal-floating-point | Time offset from beginning of the playlist if positive. Time offset before the end of playlist if negative |
| `PRECISE`     | NO       | enumerated-string             | `YES` or `NO`. If yes, then presentation should start at `TIME-OFFSET` and media before that point should not be rendered. If `NO`, then the whole segment containing `TIME-OFFSET` should be rendered. |

## Client Responsibilities

* Clients must support HTTP.
* If clients encounter a `EXT-X-VERSION` tag with a higher version than supported, they _must_ stop playback.
* ignore unrecognized tags
* ignore attribute/value pair with unrecognized AttributeName
* ignore attribute/value pair with enumerated-string value type, recognized AttributeName and unrecognized enumerated value unless otherwise specified

### Playback

* Don't start less than 3 segments from the end unless there is a `EXT-X-ENDLIST` tag
* Periodically reload playlist unless
  * `EXT-X-PLAYLIST-TYPE` tag with a value of `VOD`
  * `EXT-X-PLAYLIST-TYPE` tag with a value of `EVENT` and the `EXT-X-ENDLIST` is present
* On first load, or subsequent load with new content, client must wait at least _target duration_ before reloading
* On subsequent loads with no new content, client must wait at least one half of _target duration_ before reloading
* Subsequent loads should verify that segment sequence number and URI are stable _w.r.t._ previous loads, and fail otherwise due to server error.
* Do *not* use Media Sequence Number to synchronize between streams. Load single alternate playlist, and use time stamps to synchronize.

#### Decryption

TODO
