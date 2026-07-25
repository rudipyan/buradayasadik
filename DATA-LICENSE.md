# Terms for the testimony

The `LICENSE` file in this repository is MIT and covers the source code. **It does not cover
the testimony.** Narratives, titles, recordings, place attributions and the emotion tags
attached to them are not open-licensed and are not in the public domain by virtue of sitting in
a public repository.

This document is a statement of terms and intent, not a legal instrument. It records how the
material is meant to be treated while the project is in progress; the binding version will have
to be settled with the Koç University ethics board and with the narrators before any of this
material is published or deposited.

## Reuse is limited by consent, not by copyright

The testimony collected for this project comes from a community that was subject to state
violence. What each narrator agreed to is the governing constraint on reuse, and consent given
for a doctoral research project is not consent for redistribution, republication, model
training, journalistic quotation, or reuse under any open licence. Those permissions have not
been asked for and must not be assumed. Where narrator consent and this document disagree, the
consent governs.

Do not copy, quote, redistribute or repurpose testimony from this repository without asking
first.

## Consent records are not yet in the data

This is a known gap, recorded in the audit as finding A4.1. The current story schema is:

```
id, person, title, lat, lng, emotions[], narrative, videoUrl?, locationKnown?, period, isGhost
```

There is no provenance block at all. Missing: interview date, interviewer, transcript or
recording identifier, consent record, the language the testimony was given in, how the
coordinate was determined, and who assigned the emotion tags — the narrator or the researcher.
That last one matters most, because the interface currently presents the emotion classification
as a property of the testimony rather than as an interpretation somebody made.

A `source` block carrying these fields is to be added after fieldwork, before any testimony is
shown outside a supervised demo.

## Current content status

Fieldwork has not happened. 17 of the 18 seed narratives are placeholder text (lorem ipsum) and
are not testimony of any kind. Only story 18 is real: the 1942 Varlık Vergisi, an Armenian
blacksmith in Üsküdar whose workshop was confiscated and who was exiled to a labour camp. The
titles are real in Turkish, English and Western Armenian. Nothing in the placeholder set should
be quoted, analysed or cited.

Story 18 and its linked recording are subject to the terms above in full.

## Permission requests

TODO (Rudi): add a contact address and the name of the responsible party here, and state which
ethics approval the fieldwork runs under.
