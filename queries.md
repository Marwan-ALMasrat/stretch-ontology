# Sample SPARQL Queries — Football Ontology

All three queries run against `ontology.ttl` loaded into an `rdflib.Graph`.

---

## Query 1 — Subclass-aware: All persons associated with Al-Faisaly

This query uses `rdfs:subClassOf*` (zero-or-more hops) to retrieve every
`FootballPerson` linked to Al-Faisaly, whether they are a `Player` (via
`:playsFor`) or a `Coach` (via `:manages`). It does not hard-code either
subclass — new subclasses added later are picked up automatically.

```sparql
PREFIX :     <http://example.org/football/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?person ?name ?role
WHERE {
    # Walk the subclass chain: Player and Coach are both FootballPerson
    ?roleClass rdfs:subClassOf* :FootballPerson .
    ?person a ?roleClass ;
            rdfs:label ?name .

    # Match players via :playsFor OR coaches via :manages
    {
        ?person :playsFor :alFaisaly .
        BIND("Player" AS ?role)
    }
    UNION
    {
        ?person :manages :alFaisaly .
        BIND("Coach" AS ?role)
    }
}
ORDER BY ?role ?name
```

**Expected result on this fixture:** 3 players (Ali Hassan, Omar Khalil,
Yazan Nour) + 1 coach (Karim Saleh) = 4 rows.

---

## Query 2 — SKOS-disambiguated: Find a club by any label variant

A user might type "الوحدات", "Wehdat", or "Al-Wehdat SC" — all refer to the
same club. This query checks both `skos:prefLabel` and `skos:altLabel` so any
surface form resolves to the canonical URI and its home stadium.

```sparql
PREFIX :     <http://example.org/football/>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?club ?canonicalName ?stadium ?stadiumCity
WHERE {
    # Match the user-supplied surface form against either label predicate
    ?club ?labelPred "الوحدات" .
    FILTER (?labelPred = skos:prefLabel || ?labelPred = skos:altLabel)

    # Retrieve canonical name and home stadium
    ?club skos:prefLabel ?canonicalName ;
          :homeStadium   ?stadium .
    ?stadium :locatedIn  ?stadiumCity .
}
```

**Expected result:** 1 row — Al-Wehdat SC, King Hussein Stadium, Amman.
Replacing `"الوحدات"` with `"Wehdat"` or `"Al-Wehdat SC"` returns the same row.

---

## Query 3 — Players with and without a nickname (OPTIONAL)

This query returns every player, their club, their jersey number, and their
nickname if one exists. Players without a `:nickname` triple still appear;
`?nick` is unbound (NULL) for them.

```sparql
PREFIX :    <http://example.org/football/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?name ?club ?number ?nick
WHERE {
    ?player a :Player ;
            rdfs:label    ?name ;
            :playsFor     ?clubURI ;
            :jerseyNumber ?number .

    ?clubURI skos:prefLabel ?club .

    # Nickname is optional — players without one still appear
    OPTIONAL { ?player :nickname ?nick }
}
ORDER BY ?club ?number
```

**Expected result:** 9 rows total. 4 rows have `?nick` bound; 5 rows have
`?nick` unbound — demonstrating the open-world absence of the property.
