# Knowledge Graph Evaluation Questions

> **Purpose:** These questions are designed to trip up a general-purpose LLM that does
> not have access to the graph. Each question is answerable from
> `dave-tour-extraction.ttl` + `music_venue_ontology.ttl` but hard to guess.
>
> **Method:** Use an LLM-as-Judge pattern — give the question to a bare LLM, then
> compare its answer against the ground truth derived from SPARQL.

---

## Q1 — Multi-hop + Constraint (Venue Type × Amenity × Location)

**Question:** Which venue on Dave's tour is classified as a Nightclub and offers both
a VIP Lounge and Parking as amenities? Name the city and US state it is located in.

**Ground Truth:** The Fillmore Philadelphia, located in Philadelphia, Pennsylvania.

**Why it trips up LLMs:** An LLM will likely answer "The Fillmore" in San Francisco
(also a Nightclub in the ontology, and the most famous Fillmore). But that venue
is NOT on this tour and does not have both amenities in this dataset. There are three
distinct Fillmore venues across the two files — only one matches all constraints.

**Verification Query:**
```sparql
PREFIX mv: <http://getexcited.com/music-venues#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?venueName ?cityLabel ?stateLabel
WHERE {
  ?venue a mv:Nightclub ;
         mv:name ?venueName ;
         mv:offersAmenity ?a1 ;
         mv:offersAmenity ?a2 ;
         mv:isLocatedIn ?city .
  ?a1 rdfs:label "VIP Lounge"@en .
  ?a2 rdfs:label "Parking"@en .
  ?city rdfs:label ?cityLabel ;
        mv:locatedInState ?state .
  ?state rdfs:label ?stateLabel .
}
```

---

## Q2 — Open-World Aggregation Trap

**Question:** How many of Dave's tour events (mv:MusicEvent instances) take place at
venues classified specifically as an Arena? List every one.

**Ground Truth:** Exactly 6.
Olympiahalle (Munich), ACCOR ARENA (Paris), ING ARENA (Brussels),
PSD BANK DOME (Düsseldorf), Ziggo Dome (Amsterdam), Uber Arena (Berlin).

**Why it trips up LLMs:** An LLM might include Ziggo Dome Club (same city, similar
name, but it's a MusicClub not an Arena, and has no event). It might also include
US venues like Brooklyn Paramount or The Anthem, which sound arena-like but are
Concert Halls. Getting the exact count of 6 requires precise type knowledge.

**Verification Query:**
```sparql
PREFIX mv: <http://getexcited.com/music-venues#>

SELECT ?venueName ?eventLabel
WHERE {
  ?event a mv:MusicEvent ;
         rdfs:label ?eventLabel ;
         mv:performedAt ?venue .
  ?venue a mv:Arena ;
         mv:name ?venueName .
}
```

---

## Q3 — Negation Trap (European Amenities)

**Question:** Name all European tour venues in the dataset that offer at least one
amenity (mv:offersAmenity). If none do, state that explicitly.

**Ground Truth:** None. Zero European venues declare any amenity in the extraction.

**Why it trips up LLMs:** An LLM will almost certainly hallucinate that large arenas
like ACCOR ARENA or Ziggo Dome have bars, parking, food stands, etc. These venues
surely have amenities in real life, but the graph does not assert any — and the
question asks about the dataset, not the real world. This is a classic open-world
assumption trap.

**Verification Query:**
```sparql
PREFIX mv: <http://getexcited.com/music-venues#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?venueName ?amenityLabel
WHERE {
  ?venue mv:name ?venueName ;
         mv:offersAmenity ?amenity ;
         mv:isLocatedIn ?city .
  ?city mv:locatedInCountry ?country .
  ?country mv:locatedInContinent mv:Europe .
  ?amenity rdfs:label ?amenityLabel .
}
```

---

## Q4 — Same-City Disambiguation

**Question:** One city in the dataset contains two distinct venues that belong to
different OWL classes. Name the city, both venues, and their respective types.

**Ground Truth:** Amsterdam — Ziggo Dome (mv:Arena) and Ziggo Dome Club (mv:MusicClub).

**Why it trips up LLMs:** The names are extremely similar, so an LLM might treat them
as one venue or get the types reversed. It might also guess a US city like New York
(which has many venues in the ontology but not from this extraction's tour data).

**Verification Query:**
```sparql
PREFIX mv: <http://getexcited.com/music-venues#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?cityLabel ?name1 ?type1 ?name2 ?type2
WHERE {
  ?v1 mv:name ?name1 ; a ?type1 ; mv:isLocatedIn ?city .
  ?v2 mv:name ?name2 ; a ?type2 ; mv:isLocatedIn ?city .
  ?city rdfs:label ?cityLabel .
  FILTER(?v1 != ?v2 && ?type1 != ?type2)
  FILTER(?type1 != mv:IndoorVenue && ?type2 != mv:IndoorVenue)
}
```

---

## Q5 — Temporal + New-Country Constraint

**Question:** Which tour event has the earliest clock-time start (not earliest date),
and in which country — one that was NOT already present in the base ontology — does
it take place?

**Ground Truth:** The Brussels event on 2026-02-06 at 18:30. The venue is ING ARENA,
located in Belgium — a country minted as a new entity in this extraction because it
did not exist in the base ontology.

**Why it trips up LLMs:** Requires parsing xsd:dateTime values for clock time (not
just date), then cross-referencing which countries are "new" vs pre-existing in the
ontology. An LLM has no way to distinguish new vs existing countries without graph
access.

**Verification Query:**
```sparql
PREFIX mv:      <http://getexcited.com/music-venues#>
PREFIX prov:    <http://www.w3.org/ns/prov#>
PREFIX extract: <http://getexcited.com/music-venues/data/extraction/>
PREFIX rdfs:    <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?eventLabel ?venueName ?date ?countryLabel
WHERE {
  ?event a mv:MusicEvent ;
         rdfs:label ?eventLabel ;
         mv:performedAt ?venue ;
         mv:eventDate ?date .
  ?venue mv:name ?venueName ;
         mv:isLocatedIn ?city .
  ?city mv:locatedInCountry ?country .
  ?country rdfs:label ?countryLabel ;
           prov:wasGeneratedBy extract:activity-dave-tour-2026 .
}
ORDER BY ?date
```

---

## Q6 — Consecutive Events in Same State

**Question:** Name all pairs of chronologically consecutive tour events that take
place in different cities within the same US state. For each pair, name the state.

**Ground Truth:** Two pairs:
1. Oakland (Mar 31) → Hollywood (Apr 3) — both in California
2. Dallas (Apr 10) → Houston (Apr 11) — both in Texas

**Why it trips up LLMs:** Requires knowing the exact chronological order AND the
state assignment for each city. An LLM might not know that Hollywood (in this
dataset) is in California, or might insert a non-existent event between Oakland
and Hollywood. The Dallas→Houston pair is only 1 day apart, which is another
detail an LLM would need to guess correctly.

**Verification Query:**
```sparql
PREFIX mv:   <http://getexcited.com/music-venues#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?date1 ?city1 ?date2 ?city2 ?stateLabel
WHERE {
  ?e1 a mv:MusicEvent ; mv:eventDate ?date1 ; mv:performedAt ?v1 .
  ?e2 a mv:MusicEvent ; mv:eventDate ?date2 ; mv:performedAt ?v2 .
  ?v1 mv:isLocatedIn ?c1 . ?c1 mv:locatedInState ?s .
  ?v2 mv:isLocatedIn ?c2 . ?c2 mv:locatedInState ?s .
  ?s rdfs:label ?stateLabel .
  ?c1 rdfs:label ?city1 .
  ?c2 rdfs:label ?city2 .
  FILTER(?date2 > ?date1 && ?c1 != ?c2)
  FILTER NOT EXISTS {
    ?e3 a mv:MusicEvent ; mv:eventDate ?date3 .
    FILTER(?date3 > ?date1 && ?date3 < ?date2)
  }
}
ORDER BY ?date1
```

---

## Q7 — The Unwritten Path (Entity-Linked but No Event)

**Question:** One venue from the base ontology (not newly minted in the extraction)
is entity-linked to the Ticketmaster source page via provenance annotations but has
NO tour event (mv:MusicEvent) in this dataset. Name it and state its venue type.

**Ground Truth:** Irving Plaza, classified as mv:MusicClub in the ontology. It
appears in the extraction with PROV-O annotations (from a fan review) but no
mv:MusicEvent references it via mv:performedAt.

**Why it trips up LLMs:** An LLM would have to distinguish between "mentioned in
provenance" and "has an event." Irving Plaza is famous enough that an LLM might
assume it hosts a tour date. Conversely, it might not even know Irving Plaza
appears in this dataset at all.

**Verification Query:**
```sparql
PREFIX mv:      <http://getexcited.com/music-venues#>
PREFIX prov:    <http://www.w3.org/ns/prov#>
PREFIX extract: <http://getexcited.com/music-venues/data/extraction/>

SELECT ?venue ?venueName ?venueType
WHERE {
  ?venue prov:wasGeneratedBy extract:activity-dave-tour-2026 .
  ?venue a ?venueType .
  ?venue mv:name ?venueName .
  ?venueType rdfs:subClassOf* mv:Venue .
  FILTER NOT EXISTS {
    ?event a mv:MusicEvent ;
           mv:performedAt ?venue .
  }
}
```

---

## Q8 — Aggregation + Multi-hop (States with One City)

**Question:** How many distinct US states in the dataset contain exactly one city
that has a tour venue? List every such state.

**Ground Truth:** 7 states — Colorado (Denver), Georgia (Atlanta),
Pennsylvania (Philadelphia), Illinois (Chicago), Massachusetts (Boston),
New York (Brooklyn), Maryland (Silver Spring).

The two states with more than one city are California (Oakland, Hollywood) and
Texas (Dallas, Houston).

**Why it trips up LLMs:** Requires enumerating all state→city mappings and counting.
An LLM might miss Silver Spring/Maryland (from reviews, no event) or miscategorize
Brooklyn as part of New York City rather than having its own state mapping.

**Verification Query:**
```sparql
PREFIX mv:   <http://getexcited.com/music-venues#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?stateLabel (COUNT(DISTINCT ?city) AS ?cityCount)
WHERE {
  ?city a mv:City ;
        mv:locatedInState ?state ;
        mv:locatedInCountry mv:UnitedStates .
  ?state rdfs:label ?stateLabel .
  ?venue mv:isLocatedIn ?city ;
         mv:name ?venueName .
}
GROUP BY ?stateLabel
HAVING (COUNT(DISTINCT ?city) = 1)
ORDER BY ?stateLabel
```

---

## Q9 — Admission Policy Trap

**Question:** Which venue is the only one in the dataset with an age-restricted
admission policy? What type of venue (OWL class) is it, and in which state is
it located?

**Ground Truth:** MGM Music Hall at Fenway — a ConcertHall in Massachusetts.
The policy is "21+".

**Why it trips up LLMs:** An LLM would almost certainly guess a Nightclub or
MusicClub, since those are the venue types most commonly associated with age
restrictions. But in this dataset, the only age-restricted venue is a ConcertHall.
The specific combination of name + type + state + policy is very hard to guess.

**Verification Query:**
```sparql
PREFIX mv:   <http://getexcited.com/music-venues#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?venueName ?venueType ?policyLabel ?stateLabel
WHERE {
  ?venue mv:hasAdmissionPolicy ?policy ;
         mv:name ?venueName ;
         a ?venueType ;
         mv:isLocatedIn ?city .
  ?policy rdfs:label ?policyLabel .
  ?city mv:locatedInState ?state .
  ?state rdfs:label ?stateLabel .
  FILTER(?venueType != mv:IndoorVenue)
}
```

---

## Q10 — Provenance Offset (Impossible Without Data)

**Question:** According to the Open Annotation text position selectors (oa:start),
which extracted venue appears at the highest character offset in the Ticketmaster
source document, and which appears at the lowest? Give exact offset values.

**Ground Truth:**
- **Lowest:** Fox Theater - Oakland at `oa:start 1884`
- **Highest:** Commodore Ballroom at `oa:start 10878`

**Why it trips up LLMs:** Character offsets in a specific web page are pure graph
data with zero real-world signal. No amount of reasoning or world knowledge can
produce these numbers — this question has a 0% chance of being answered without
graph access.

**Verification Query:**
```sparql
PREFIX mv:      <http://getexcited.com/music-venues#>
PREFIX prov:    <http://www.w3.org/ns/prov#>
PREFIX oa:      <http://www.w3.org/ns/oa#>
PREFIX extract: <http://getexcited.com/music-venues/data/extraction/>

SELECT ?venueName ?start
WHERE {
  ?venue mv:name ?venueName ;
         prov:wasGeneratedBy extract:activity-dave-tour-2026 ;
         prov:wasDerivedFrom ?sel .
  ?sel oa:start ?start .
}
ORDER BY ?start
```

---

## Scoring Rubric (LLM-as-Judge)

For each question, score the bare LLM response on a 0–3 scale:

| Score | Criteria |
|-------|----------|
| **3** | Exact match on all entities, counts, and constraints |
| **2** | Correct main entity but wrong on a secondary detail (e.g., right venue, wrong state) |
| **1** | Partially relevant answer that shows some knowledge but misses the key constraint |
| **0** | Hallucinated answer, wrong entity, or refused to answer |

**Expected bare-LLM baseline:** 0–4 out of 30 total points. Questions Q3, Q5, Q7,
and Q10 are specifically designed to score 0 for any LLM without graph access.
