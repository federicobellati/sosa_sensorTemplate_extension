# Ocean SOSA Sensor Model Extension

## Overview

This repository contains an extension profile for using the W3C/OGC SOSA/SSN ontology to describe oceanographic sensors, sensor templates, sensor instances, sensor components, and their relationship to OGC SensorThings API.

The work started from a practical interoperability problem: SOSA/SSN provides a flexible and general ontology for sensors, observations, platforms, procedures, observable properties, and systems, but it does not prescribe one deterministic modelling pattern for distinguishing between a reusable sensor template/model and a concrete deployed sensor instance. This flexibility is useful, but in real-world oceanographic catalogues it can lead to divergent implementations and reduced interoperability.

The goal of this extension is therefore not to replace SOSA/SSN, but to define a conservative application profile that makes the model/instance distinction explicit while remaining compatible with SOSA/SSN and reasonably mappable to SensorThings API.

Relevant background standards and vocabularies include:

- W3C/OGC SOSA/SSN ontology.
- SSN System Capabilities module.
- OGC SensorThings API conceptual model.
- QUDT for units of measure and quantity values.
- Schema.org `ProductModel` for product/model-level metadata.
- PROV-O and Dublin Core Terms for provenance, source and versioning metadata.

## Motivation

In many oceanographic infrastructures, the same physical type of instrument may appear in many deployed instances.

For example:

```text
Sea-Bird SBE37 MicroCAT
    ├── SBE37 serial number 11872
    ├── SBE37 serial number 11873
    └── SBE37 serial number 11874
```

A catalogue must be able to represent both:

1. The reusable model/template: technical specifications, brochure, manufacturer, nominal observed properties, nominal accuracy, resolution, operating ranges, available configurations and components.
2. The concrete instance: serial number, inventory identifier, deployment, calibration history, hosting platform, produced observations.

SOSA/SSN has `sosa:Sensor`, but the ontology does not explicitly separate “sensor model” from “sensor instance”. In practice, different implementations use `sosa:Sensor` either for the physical deployed instrument or for the reusable sensor type. Both approaches can be made to work, but the ambiguity harms interoperability.

This extension chooses a clear modelling convention:

```text
ososa:SensorModel
    = reusable product model, template, commercial model or technical specification

sosa:Sensor
    = concrete sensor instance capable of making observations
```

## Design Decision: Why Introduce `SensorModel`

Two alternative approaches were evaluated.

### Approach A — Add `ososa:SensorModel`

In this approach:

```text
sosa:Sensor        = concrete observation-making sensor instance
ososa:SensorModel  = reusable template/model/specification
```

Example:

```turtle
ex:SBE37_SN11872
    a sosa:Sensor ;
    ososa:hasSensorModel ex:SeaBirdSBE37 ;
    sosa:isHostedBy ex:AcquaAltaPlatform .
```

This approach was selected because it preserves the usual SOSA/SSN interpretation of `sosa:Sensor` as the entity that can make observations.

### Approach B — Add `ososa:SensorInstance`

An alternative was to keep `sosa:Sensor` as the template and introduce:

```text
ososa:SensorInstance = concrete deployed sensor
```

This was rejected as the primary pattern because it would either:

- create a class separate from `sosa:Sensor`, making `sosa:madeBySensor` harder to use directly; or
- make `ososa:SensorInstance` a subclass of `sosa:Sensor`, which weakens the clean “template only” interpretation of `sosa:Sensor`.

The selected approach is therefore more interoperable with standard SOSA/SSN observation patterns.

## Core Modelling Pattern

The recommended pattern is:

```text
SensorModel
    ├── SensorModelComponent
    │       ├── nominal observed property
    │       ├── nominal unit
    │       ├── nominal accuracy
    │       ├── nominal resolution
    │       └── nominal operating range
    │
    └── documentation, manufacturer, version, profile, variants

Sensor instance
    ├── hasSensorModel → SensorModel
    ├── isHostedBy → Platform
    ├── observes → ObservableProperty
    └── made observations
```

In RDF terms:

```turtle
ex:SeaBirdSBE37
    a ososa:SensorModel ;
    rdfs:label "Sea-Bird SBE37 MicroCAT" ;
    ssn:hasSubSystem ex:SBE37_TemperatureComponent ;
    ssn:hasSubSystem ex:SBE37_ConductivityComponent .

ex:SBE37_SN11872
    a sosa:Sensor ;
    ososa:hasSensorModel ex:SeaBirdSBE37 ;
    ososa:serialNumber "11872" ;
    sosa:isHostedBy ex:AcquaAltaPlatform ;
    sosa:observes ex:SeaWaterTemperature .
```

## Sensor Model Components

During the design discussion, we considered introducing a new class such as `MeasurementChannel`.

However, SOSA/SSN already provides a general system-composition pattern through `ssn:System` and `ssn:hasSubSystem`. Therefore, instead of inventing a completely parallel structure, this extension introduces:

```text
ososa:SensorModelComponent
```

as a model-level component or nominal subsystem.

A sensor model component can represent, for example:

- temperature measurement component;
- conductivity measurement component;
- pressure measurement component;
- dissolved oxygen component;
- power component;
- communication component;
- processing component;
- mechanical/housing component.

The recommended hierarchy is:

```text
ososa:SensorModelComponent
    rdfs:subClassOf ssn:System

ososa:SensingComponent
    rdfs:subClassOf ososa:SensorModelComponent

ososa:ProcessingComponent
    rdfs:subClassOf ososa:SensorModelComponent

ososa:CommunicationComponent
    rdfs:subClassOf ososa:SensorModelComponent

ososa:PowerComponent
    rdfs:subClassOf ososa:SensorModelComponent
```

This allows a model such as a Sea-Bird SBE37 to be represented as a composed nominal system:

```turtle
ex:SeaBirdSBE37
    a ososa:SensorModel ;
    ssn:hasSubSystem ex:SBE37_TemperatureComponent ;
    ssn:hasSubSystem ex:SBE37_ConductivityComponent ;
    ssn:hasSubSystem ex:SBE37_PressureComponent .
```

## Capabilities, Accuracy, Resolution and Ranges

Nominal capabilities should be associated preferably with the relevant model component, not only with the whole sensor model.

For example, temperature accuracy and pressure accuracy are different characteristics and should not be attached ambiguously to the whole instrument.

Recommended pattern:

```turtle
ex:SBE37_TemperatureComponent
    a ososa:SensingComponent ;
    ososa:designedToObserve ex:SeaWaterTemperature ;
    ososa:hasDefaultUnit unit:DEG_C ;
    ososa:hasModelCapability ex:SBE37_TemperatureAccuracy ;
    ososa:hasModelCapability ex:SBE37_TemperatureResolution ;
    ososa:hasModelOperatingRange ex:SBE37_TemperatureOperatingRange .
```

Capabilities can reuse the SSN System Capabilities vocabulary:

```turtle
ex:SBE37_TemperatureAccuracy
    a ssn-system:Accuracy ;
    qudt:quantityValue [
        a qudt:QuantityValue ;
        qudt:numericValue "0.002"^^xsd:decimal ;
        qudt:unit unit:DEG_C
    ] .
```

The key distinction is:

```text
ososa:hasModelCapability
    = nominal capability declared at model/template level

ssn-system:hasSystemCapability
    = capability of an actual SSN system
```

This avoids incorrectly treating a product model as if it were itself the operational observation-making sensor.

## Why Not Use `sosa:observes` Directly on `SensorModel`

Although a sensor model may be designed to observe temperature, salinity, pressure or oxygen, the model itself does not make observations.

For this reason, this extension introduces:

```text
ososa:designedToObserve
```

instead of using:

```text
sosa:observes
```

directly on `ososa:SensorModel`.

Recommended pattern:

```turtle
ex:SeaBirdSBE37
    a ososa:SensorModel ;
    ososa:designedToObserve ex:SeaWaterTemperature .

ex:SBE37_SN11872
    a sosa:Sensor ;
    ososa:hasSensorModel ex:SeaBirdSBE37 ;
    sosa:observes ex:SeaWaterTemperature .
```

This keeps the semantics clear:

```text
SensorModel designedToObserve a property.
Sensor observes a property.
Observation observedProperty a property.
```

## Compatibility with SensorThings API

A key design requirement was compatibility with OGC SensorThings API.

SensorThings API provides entities such as:

```text
Thing
Sensor
ObservedProperty
Datastream
Observation
Location
HistoricalLocation
```

In many SensorThings implementations, `Thing` is used to represent a platform, such as a buoy or station. However, `Thing` is generic and may also represent the physical sensor instance itself.

For this profile, the preferred mapping is:

```text
SensorThings Thing
    ≈ concrete physical asset / deployed sensor instance

SensorThings Sensor
    ≈ reusable sensor model, sensor description or measurement procedure
```

This is especially useful for oceanographic asset management, because the physical sensor instance has a serial number, calibration events, maintenance records and installation history.

## Mapping Table: SOSA/SSN + Extension to SensorThings API

| Concept | SOSA/SSN + Extension | SensorThings API | Notes |
|---|---|---|---|
| Sensor model / template | `ososa:SensorModel` | `Sensor` | The SensorThings `Sensor` is treated as the reusable sensor description, model, or procedure. |
| Physical sensor instance | `sosa:Sensor` | `Thing` | Preferred mapping when the catalogue focuses on physical instruments and asset management. |
| Platform | `sosa:Platform` | `Thing` or custom relation from sensor `Thing` | If `Thing` is used for the physical sensor, the hosting platform requires an explicit relation or extension. |
| Hosting relation | `sosa:isHostedBy` | Custom property or navigation relation | SensorThings does not provide a direct equivalent of `sosa:isHostedBy`. |
| Observable property | `sosa:ObservableProperty` | `ObservedProperty` | Should use controlled vocabularies such as CF Standard Names or NERC NVS where possible. |
| Observation | `sosa:Observation` | `Observation` | Direct conceptual mapping. |
| Datastream | Collection or logical grouping of observations | `Datastream` | SensorThings has an explicit `Datastream` entity; SOSA/SSN normally requires an application pattern. |
| Result value | `sosa:hasSimpleResult` or `sosa:hasResult` | `Observation.result` | Units should be explicitly declared. |
| Unit of measurement | QUDT `qudt:Unit` | `Datastream.unitOfMeasurement` | SensorThings uses a simple unit object; RDF profile should prefer QUDT. |
| Sensor component / channel | `ososa:SensorModelComponent`, `ososa:SensingComponent`, `ssn:hasSubSystem` | Usually not native; may be represented in Sensor metadata or custom properties | Useful for multi-parameter instruments. |
| Accuracy, resolution, precision | `ososa:hasModelCapability` + `ssn-system:*` | Usually custom Sensor metadata | Best represented in RDF using SSN-System and QUDT. |
| Operating range | `ososa:hasModelOperatingRange` + `ssn-system:OperatingRange` | Usually custom Sensor metadata | Should be attached to the relevant component/channel. |
| Brochure / datasheet | `schema:subjectOf`, `foaf:page`, `dcterms:references` | Sensor metadata or external link | Documentation belongs naturally to the model/template. |
| Manufacturer | `schema:manufacturer` | Sensor metadata | The manufacturer describes the model, not necessarily the instance owner. |
| Serial number | `ososa:serialNumber` on `sosa:Sensor` | Thing properties | Instance-level property. |
| Inventory identifier | `ososa:inventoryIdentifier` on `sosa:Sensor` | Thing properties | Institution-specific asset identifier. |
| Deployment | `ssn:Deployment` or deployment-specific extension | HistoricalLocation and/or custom properties | No perfect one-to-one mapping. |
| Provenance / source | `dcterms:source`, `prov:wasDerivedFrom` | External metadata | Useful for tracking datasheets, catalogues and releases. |

## Repository and Serialization Choices

The ontology source should be maintained in Turtle:

```text
ontology/ocean-sosa.ttl
```

Turtle is the preferred source format because it is readable, version-control friendly and easy to review in Git.

Optional generated serializations may be provided for tool compatibility:

```text
ontology/ocean-sosa.rdf
ontology/ocean-sosa.jsonld
ontology/ocean-sosa.nt
```

The `.rdf` file should normally contain RDF/XML, not Turtle. Therefore, the Turtle source should not simply be renamed as `.rdf`.

Recommended repository structure:

```text
ocean-sosa/
├── ontology/
│   ├── ocean-sosa.ttl
│   ├── ocean-sosa.rdf
│   └── ocean-sosa.jsonld
├── examples/
│   ├── seabird-sbe37-example.ttl
│   └── sensor-instance-example.ttl
├── shacl/
│   └── ocean-sosa-shapes.ttl
├── docs/
│   └── index.md
├── README.md
└── LICENSE
```

The `.ttl` file should be treated as the authoritative source. Other serializations should preferably be generated during release.

## Ontology IRI, Versioning and Provenance

The ontology should have a stable namespace IRI, preferably not tied directly to a GitHub `blob/main` URL.

Recommended pattern:

```text
Ontology IRI:
https://w3id.org/ocean-sosa/

Version IRI:
https://w3id.org/ocean-sosa/0.1.0/
```

The GitHub repository should be referenced as the source repository, not necessarily as the conceptual namespace.

Example ontology metadata:

```turtle
@prefix ososa:   <https://w3id.org/ocean-sosa/> .
@prefix owl:     <http://www.w3.org/2002/07/owl#> .
@prefix dcterms: <http://purl.org/dc/terms/> .
@prefix vann:    <http://purl.org/vocab/vann/> .
@prefix foaf:    <http://xmlns.com/foaf/0.1/> .
@prefix xsd:     <http://www.w3.org/2001/XMLSchema#> .

ososa:
    a owl:Ontology ;
    dcterms:title "Ocean SOSA Sensor Model Extension"@en ;
    dcterms:description "Extension of SOSA/SSN introducing SensorModel and SensorModelComponent for oceanographic sensor catalogues."@en ;
    owl:versionIRI <https://w3id.org/ocean-sosa/0.1.0/> ;
    owl:versionInfo "0.1.0" ;
    dcterms:created "2026-06-20"^^xsd:date ;
    dcterms:license <https://creativecommons.org/licenses/by/4.0/> ;
    dcterms:source <https://github.com/YOUR-ORG/ocean-sosa> ;
    foaf:homepage <https://YOUR-ORG.github.io/ocean-sosa/> ;
    vann:preferredNamespacePrefix "ososa" ;
    vann:preferredNamespaceUri "https://w3id.org/ocean-sosa/" .
```

Datasets or sensor descriptions using this profile may declare conformance as follows:

```turtle
ex:SBE37_SN11872
    a sosa:Sensor ;
    ososa:hasSensorModel ex:SeaBirdSBE37 ;
    dcterms:conformsTo <https://w3id.org/ocean-sosa/0.1.0/> .
```

If another ontology or application profile depends on this extension, it may import it:

```turtle
<https://example.org/my-sensor-catalogue-ontology>
    a owl:Ontology ;
    owl:imports <https://w3id.org/ocean-sosa/> .
```

## Example: Sea-Bird SBE37 Pattern

```turtle
@prefix ex:         <https://example.org/resource/> .
@prefix ososa:      <https://w3id.org/ocean-sosa/> .
@prefix sosa:       <http://www.w3.org/ns/sosa/> .
@prefix ssn:        <http://www.w3.org/ns/ssn/> .
@prefix ssn-system: <http://www.w3.org/ns/ssn/systems/> .
@prefix qudt:       <http://qudt.org/schema/qudt/> .
@prefix unit:       <http://qudt.org/vocab/unit/> .
@prefix schema:     <https://schema.org/> .
@prefix xsd:        <http://www.w3.org/2001/XMLSchema#> .

ex:SeaBirdSBE37
    a ososa:SensorModel ;
    rdfs:label "Sea-Bird SBE37 MicroCAT"@en ;
    schema:manufacturer ex:SeaBirdScientific ;
    schema:model "SBE37" ;
    ssn:hasSubSystem ex:SBE37_TemperatureComponent ;
    ssn:hasSubSystem ex:SBE37_ConductivityComponent ;
    ssn:hasSubSystem ex:SBE37_PressureComponent .

ex:SBE37_TemperatureComponent
    a ososa:SensingComponent ;
    rdfs:label "SBE37 nominal temperature sensing component"@en ;
    ososa:designedToObserve ex:SeaWaterTemperature ;
    ososa:hasDefaultUnit unit:DEG_C ;
    ososa:hasModelCapability ex:SBE37_TemperatureAccuracy .

ex:SBE37_TemperatureAccuracy
    a ssn-system:Accuracy ;
    rdfs:label "SBE37 nominal temperature accuracy"@en ;
    qudt:quantityValue [
        a qudt:QuantityValue ;
        qudt:numericValue "0.002"^^xsd:decimal ;
        qudt:unit unit:DEG_C
    ] .

ex:AcquaAltaPlatform
    a sosa:Platform ;
    rdfs:label "Acqua Alta oceanographic platform"@en .

ex:SBE37_SN11872
    a sosa:Sensor ;
    rdfs:label "Sea-Bird SBE37 serial number 11872"@en ;
    ososa:hasSensorModel ex:SeaBirdSBE37 ;
    ososa:serialNumber "11872" ;
    sosa:isHostedBy ex:AcquaAltaPlatform ;
    sosa:observes ex:SeaWaterTemperature .
```

## Recommended Modelling Rules

The following rules are recommended for consistency:

1. Use `ososa:SensorModel` for reusable sensor templates, product models, commercial models or technical specifications.
2. Use `sosa:Sensor` for concrete sensor instances that can make observations.
3. Link sensor instances to their model using `ososa:hasSensorModel`.
4. Use `sosa:isHostedBy` to link physical sensor instances to platforms.
5. Use `ososa:designedToObserve` at model/component level.
6. Use `sosa:observes` at concrete sensor instance level.
7. Use `ssn:hasSubSystem` to describe model components or nominal subsystems.
8. Attach accuracy, resolution, range and unit metadata to the most specific relevant component.
9. Use QUDT for units and quantitative values.
10. Use controlled vocabularies such as CF Standard Names or NERC NVS for observable properties where possible.
11. Keep examples separate from the ontology core.
12. Use SHACL shapes to enforce local profile constraints such as required model links, serial numbers or allowed property cardinalities.

## Future Work

Possible future additions include:

- SHACL validation profile.
- JSON-LD context for web/API use.
- Formal SensorThings API mapping examples.
- Alignment with NERC Vocabulary Server concepts.
- Alignment with CF Standard Names.
- Deployment and calibration profile.
- Versioned release workflow.
- GitHub Pages documentation.
- w3id.org persistent identifier setup.

## Summary

This extension provides a practical and semantically conservative way to model oceanographic sensor catalogues using SOSA/SSN.

The central decision is to keep `sosa:Sensor` for concrete observation-making sensor instances and introduce `ososa:SensorModel` for reusable sensor templates. Internal model structure is represented using `ososa:SensorModelComponent` and `ssn:hasSubSystem`, while nominal capabilities are expressed using SSN-System and QUDT.

This pattern improves interoperability because it makes explicit the distinction between:

```text
model / template
instance / asset
platform / host
component / channel
observation / result
```

and it provides a stable bridge toward SensorThings API implementations.
