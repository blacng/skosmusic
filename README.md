# SKOS Music Venue Vocabulary & Ontology

A linked data project for describing music venues using [SKOS](https://www.w3.org/2004/02/skos/) (Simple Knowledge Organization System) and [OWL](https://www.w3.org/OWL/) (Web Ontology Language). All resources are serialized in [Turtle](https://www.w3.org/TR/turtle/) (`.ttl`).

## Namespaces

| Prefix | URI | Description |
|--------|-----|-------------|
| `mv:` | `http://getexcited.com/music-venues#` | Music venue concepts and classes |
| `geo:` | `http://getexcited.com/music-venues/geo#` | Geographic location classes and individuals |

## Files

### `music_venue_vocabulary.ttl` — SKOS Concept Scheme

The core controlled vocabulary defining the **MusicVenueVocabulary** concept scheme with two top concepts:

**Venue Type** — a taxonomy of music venue categories:

```
Venue
├── Indoor Venue
│   ├── Concert Hall
│   ├── Arena
│   ├── Nightclub
│   ├── Live Music Club
│   └── Cabaret
├── Outdoor Venue
│   ├── Amphitheater
│   ├── Stadium
│   └── Fairground
└── Multi-Unit Venue
    ├── Performing Arts Building
    └── Community Center
```

**Venue Location** — a geographic hierarchy:

```
Continent
├── North America (United States, Canada, Mexico)
├── South America (Brazil, Argentina, Colombia)
├── Europe (United Kingdom, Germany, France, Spain)
├── Asia (Japan, South Korea, India)
├── Africa (South Africa, Nigeria, Kenya)
└── Oceania (Australia, New Zealand)
```

Concepts include `skos:prefLabel`, `skos:altLabel`, `skos:definition`, and `skos:exactMatch` links to Wikidata entities where available.

### `venues_type.ttl` — SKOS Venue Type Concepts

A focused SKOS concept scheme (`mv:venue-type`) containing just the venue type hierarchy with broader/narrower relationships, alternative labels, and Wikidata exact matches.

### `music_venue_ontology.ttl` — OWL Ontology

A formal OWL ontology that provides class-level modeling for the domain:

- **Facility & Amenity hierarchy** — `Facility > Venue`, `Facility > Amenity > Room | EatingOrDrinkingEstablishment | ParkingArea`
- **Venue type classes** — `Venue > IndoorVenue | OutdoorConcertVenue | MultiUnitVenue` with subclasses mirroring the SKOS taxonomy
- **Location classes** — `Location > Continent | Country | City`
- **Object properties** — `isLocatedIn`, `offersAmenity`, `locatedInContinent`, `locatedInCountry`, `capitalCity`, `mostPopulousCity`
- **Datatype properties** — `hasSeatingCapacity` (`xsd:integer`), `hasSeating` (`xsd:boolean`)
- **Individuals** — 6 continents, 18 countries, and 24 cities with labels and music-venue-related descriptions

## Getting Started

### Prerequisites

- Python 3.13+
- [uv](https://docs.astral.sh/uv/) for dependency management

### Setup

```bash
uv sync
```

### Validate Turtle syntax

You can validate the `.ttl` files using a tool like [rapper](https://librdf.org/raptor/rapper.html):

```bash
rapper -i turtle -c music_venue_vocabulary.ttl
rapper -i turtle -c venues_type.ttl
rapper -i turtle -c music_venue_ontology.ttl
```

## License

This project is provided for educational and research purposes.