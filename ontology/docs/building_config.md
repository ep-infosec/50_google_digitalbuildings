# Model Instance Configuration (building configuration file)

**Prerequisite** Please review the [Learning Modules](../../README.md#learning-modules).
This readme assumes that the audience is familiar with the high level concepts
which comprise the building config.

In addition to a language and primitives for configuring an abstract building
model, the Digital Buildings project provides a YAML configuration syntax for
mapping concrete assets to the abstract model in a lightweight way. The intent
of providing this syntax is to make "manual" onboarding faster and easier
(because the resulting data file is machine readable and can be machine
validated) as well as provide a clean interface format for machine-assisted
onboarding to use.

NB: Some of the instructions here details that are specific to Google's
implementation of this stack on its own campuses (such as references to CloudIoT
registration). These should be relatively obvious to a critical reader, but if
they are confusing feel free to post an issue in the project.

*   For an explanation of types in the Digital Buildings abstract model see
    [model](model.md)
*   For a conceptual explanation of the ontology see [ontology](ontology.md)

- [Model Instance Configuration](#model-instance-configuration)
  * [Key Concepts](#key-concepts)
  * [Typical Data Elements](#typical-data-elements)
  * [Configuration Detail](#configuration-detail)
    + [Contents](#contents)
    + [Config Format](#config-format)
    + [Spaces](#spaces)
    + [Devices](#devices)
      - [Reporting Physical Devices](#reporting-physical-devices)
        * [Defining Translations](#defining-translations)
          + [Translation Shortcuts](#translation-shortcuts)
          <!--- + [Compliant Short forms](#compliant-short-forms) --->
        <!-- * [Metadata](#metadata) -->
      - [Virtual Devices](#virtual-devices)
      - [Device Relationships](#device-relationships)
    + [Zones and Control Groups](#zones-and-control-groups)
  * [Building Configuration Modes](#building-configuration-modes)
    + [INITIALIZE](#initialize)
    + [UPDATE](#update)
      - [General](#general-updates)
      - [Special Cases](#special-cases)
  * [Validation](#validation)
  * [Notes](#notes)

## Key Concepts

Creation of a concrete model is primarily concerned with defining entities,
their relationships to each other, and their relationships to real physical
things in the world.

Conceptually, we divide entities into three categories:

*   **Logical Entity**: An entity that maps 1:1 with a canonical concept in the
    model. For example, you might have an idealized concept of an Air Handler,
    with a corresponding entity type, that you may use for analysis. A logical
    air handler would use this type or a child of it.
*   **Reporting Entity**: A reporting entity is one that generates telemetry
    data. Reporting entities are important to differentiate because they may
    have translations to convert their native data payload to a standard form.
*   **Virtual Entity** A virtual entity does NOT directly generate telemetry
    data, but may still have a timeseries that is generated by linking data from
    other sources.

The following statements apply when considering entity categories:

*   Either a Reporting Entity or a Virtual Entity may be a Logical Entity.
*   Virtual Entities are typically also Logical Entities.
*   By convention, anything that receives telemetry is a Reporting Entity, even
    if it has data linked from other entities. In practice, however, we do not
    recommend mixing the two concepts (i.e. by linking additional data from
    other devices into a reporting device).

To completely map real data to the model we also need a few more concepts:

*   **Translation**: A key-value mapping between fields in a reporting device's
    payload and the standard fields in the device's assigned entity type.
*   **Link**: A mapping between a standard field in one device and another
    standard field in another device. Links map data from reporting devices to
    virtual devices.
*   **Connection**: Describes a relationship between two entities. See
    [Connections](ontology_config.md#connections)

These should all be defined in a complete configuration.

## Typical Data Elements

While there is basically infinite diversity in what can be defined for a
building, some elements are more or less expected in every model:

*   All the logical spaces in the building with their unique names (Building,
    Rooms, Floors).
*   Each logical piece of equipment (hereafter "device") and its associated
    Digital Buildings Type.
*   Each reporting device registered in Cloud IoT (or your endpoint of choice,
    if not working on a Google building).
*   Link mappings between the points of reporting devices and logical devices,
    if the two are not the same.
*   Translation mappings between device-native point names and the standard
    point names for each reporting device
*   Zones or Switch Groups, as applicable.
*   `FEEDS` relationships between chained equipment in HVAC or power systems, as
    well as between terminal units and Zones.
*   `CONTROLS` relationships between switch groups and fixtures, and between
    switches and switch groups.
*   `CONTAINS` relationships between Zones or Switch Groups and rooms.
*   `CONTAINS` relationships between all entities and floors (or more specific
    space, if known).
*   `CONTAINS` relationships between Buildings and floors, and floors and rooms.

## Configuration Detail

### Contents

Each configuration file should contain the contents for one building (or other
logical division of your universe). The reason for this is that the definitions
in the file are intentionally not keyed by a globally unique identifier. This is
done for a few of reasons:

1.  The human-readable entity code is, well, more human readable.
2.  Using human-readable codes, which are typically created at design time and
    known a-priori, allows for incremental population of the file at different
    stages of the commissioning or onboarding process, rather than having to
    wait until all the devices are cloud-registered.
3.  There is inherent value to forcing locally unique naming of spaces and
    equipment (because it makes it easier to find stuff).

### Config Format

The configuration format is focused around defining the entities in the model. A
generic entity with all possible top level fields looks like this[^5]:

**NOTE:** The new Building Config format switches entities being keyed by codes
to being keyed by guids and Ids are removed. To convert the old format to the
new format, run your config.yaml through the [guid generator](https://github.com/google/digitalbuildings/tree/master/tools/guid_generator).

#### Identifiers
*   **GUID:** A globally unique identifier for the entity. This field does not
    need to be included initially and can be generated with guid generator.
*   **Code:** The human readable identifier for the entity. This should
    be unique in document scope.
*   **cloud_device_id:** the cloud device numeric id from the cloud iot registry. A server-generated device numeric ID. The device numeric ID is automatically created by Cloud IoT Core; it's globally unique and not editable. To view a device numeric ID, go to the [Device ids page](https://cloud.google.com/iot/docs/concepts/devices#device_identifiers). This field is mandatory when a translation exists.

#### New Format
``` yaml
f7d82b75-ea41-49e2-bb5a-53228044eb4c # Entity keyed by a GUID
  type: NAMESPACE/A_DIGITAL_BUILDINGS_ENTITY_TYPE
  code: ENTITY-CODE
  connections:
    # Listed entities are sources on connections
    ANOTHER-ENTITY-GUID: FEEDS
    A-THIRD-ENTITY-GUID: CONTAINS
  links:
    A-FOURTH-ENTITY-GUID: # Source entity keyed by a GUID
      # target_device_field: source_device_field
      supply_air_damper_position_command: supply_air_damper_command_1
      zone_air_temperature: zone_air_temperature_sensor_1
  cloud_device_id: device-id-from-cloud-iot-registry
  translation:
    zone_air_temperature_sensor:
      present_value: "points.temp_1.present_value"
      units:
        key: "points.temp_1.units"
        values:
          degrees_celsius: "degC"
    supply_air_isolation_damper_command:
      present_value: "points.damper_1.present_value"
      states:
        OPEN: "1"
        CLOSED:
        - "2"
        - "3"
```

#### Old Format
``` yaml
ENTITY-CODE:
  guid: f7d82b75-ea41-49e2-bb5a-53228044eb4c # guid field is generated with guid generator.
  type: NAMESPACE/A_DIGITAL_BUILDINGS_ENTITY_TYPE
  connections:
    # Listed entities are sources on connections
    ANOTHER-ENTITY: FEEDS
    A-THIRD-ENTITY: CONTAINS
  links:
    A-FOURTH-ENTITY: # Source entity code
      # target_device_field: source_device_field
      supply_air_damper_position_command: supply_air_damper_command_1
      zone_air_temperature: zone_air_temperature_sensor_1
  cloud_device_id: device-id-from-cloud-iot-registry
  translation:
    zone_air_temperature_sensor:
      present_value: "points.temp_1.present_value"
      units:
        key: "points.temp_1.units"
        values:
          degrees_celsius: "degC"
    supply_air_isolation_damper_command:
      present_value: "points.damper_1.present_value"
      states:
        OPEN: "1"
        CLOSED:
        - "2"
        - "3"
```

*   **GUID:** A globally unique identifier for the entity. This field does not
    need to be included initially and can be generated with guid generator.
*   **Type:** A valid, fully qualified Digital Buildings entity type that
    represents this entity.
*   **Code:** The human readable identifier for the entity. This should
    be unique in document scope.
*   **cloud_device_id:** the cloud device numeric id from the cloud iot registry. A server-generated device numeric ID. The device numeric ID is automatically created by Cloud IoT Core; it's globally unique and not editable. To view a device numeric ID, go to the [Device ids page](https://cloud.google.com/iot/docs/concepts/devices#device_identifiers). This field is mandatory when a translation exists.
*   **Connections:** Used to specify connections from other entities (sources)
    pointing to this entity, with connection types. Entities are keys and cannot
    be repeated. Values are one or more connections, specified as a single
    value or a set.
*   **Links:** Used to specify mappings between standard fields of source
    entities to standard fields of this entity. First level key is another
    entity in the file (source). Second level key is a standard field of this
    (target) entity followed by a `:` and a standard field of the other (source)
    entity.
*   **Translation:** Used to specify how the fields of the devices native
    payload map to the standard fields of this entity's type. See
    [translation section](#translations) for more detail.

The Digital Buildings platform has recently transitioned to the new format,
which is identical to the above format except that entities were keyed by their
codenand contained a separate **id** and **guid** field.

Entities are now keyed by a globally unique identifier and **code** has been
moved to it's own separate field. **id** fields have been totally deprecated.

The old format is still valid if the **id** field is removed, and can be
converted to the new format using the [guid generator](https://github.com/google/digitalbuildings/tree/master/tools/guid_generator).

**NOTE:** Please continue using the old format, and use the [guid generator](https://github.com/google/digitalbuildings/tree/master/tools/guid_generator)
to convert a building configuration to the new format.

### Spaces

In nearly every model, entities should exist for Buildings, Floors, and Rooms.
The following example shows a building with one floor and one room:

``` yaml
# Building
UK-LON-S2:
  type: FACILITIES/BUILDING

# Floor
UK-LON-S2-1:
  type: FACILITIES/FLOOR
  connections:
    UK-LON-S2: CONTAINS

# Room
UK-LON-S2-1-1C3G:
  type: FACILITIES/ROOM
  connections:
    UK-LON-S2-1: CONTAINS

```

*   Buildings shall have `CONTAINS` connections to Floors
*   Floors should have `CONTAINS` connections to all Rooms
*   Floors should have `CONTAINS` connections to all Devices

In this example, entities are identified by their globally unique identifiers.
GUIDSs are widely used and highly recommended due to their exponentially low
collision rate.

Types for spaces are contained in the `FACILITIES` namespace of the ontology.

### Devices

#### Reporting Physical Devices

When sending data from a building via Cloud IoT, a reporting device is any
device with its own entry in Cloud Device Manager (CDM)[^1].

For clarity the human readable ID of the device in CDM and the entity code in
the config file should be the same[^6]. If the building also has a CAD drawing
or BIM model, good practice is for this code to also exist in the CAD or BIM
model.

Choose an entity type that has the correct fields for this type.

Define a translation as necessary...

##### Defining Translations

It is commonly (always?) the case that real devices have data payloads that
differ from our modeled entity types. In this case, it is necessary to define a
translation between the native payload and the standard form. This
standardization includes the _field names_, _field values_, _dimensional units_
and _multistate definitions_.

Below is a minimal example of what a translation configuration looks like in
long form:

``` yaml
FCU-123:
  ...
  translation:
    zone_air_temperature_sensor:
      present_value: "points.temp_1.present_value"
      units:
        key: "pointset.points.temp_1.units"
        values:
          degrees_celsius: "degC"
    supply_air_isolation_damper_command:
      present_value: "points.damper_1.present_value"
      states:
        OPEN: "1"
        CLOSED:
        - "2"
        - "3"
    zone_air_temperature_setpoint: MISSING
```

Inside the `translation` block, keys correspond to standard fields in the
ontology. Within each field block we provide information about the following:

*   `present_value`: defines the fully qualified path in the devices native JSON
    payload that contains the value of this field
*   `units`: when `present_value` is a dimensional number, this block defines
    information about the payload location that defines dimensional units for
    this field's `present_value`
    *   `key`: defines the fully qualified path to the section of the payload
        that calls out the units
    *   `values`: Maps standard Digital Buildings units to the values in the
        payload that represent those units
*   `states`: If `present_value` represents a multistate value, this block is
    used to map the native state values to standard ones. Each map key is a
    standard state, and its value is the corresponding native value, or a list
    of native values if more than one corresponds to the same standard state.
*   If a device lacks a field required for its type, the field should be marked
    `MISSING` as shown above.

###### Translation Shortcuts

In order to eliminate duplicate work, the format provides some shortcuts to
translation definitions:

1.  Substitute the `translation` block with `translate_like:ENTITY-CODE` to use
    a translation that is already defined on another entity. The [guid generator](https://github.com/google/digitalbuildings/tree/master/tools/guid_generator)
    does this for you.
2.  For devices that comply with
    [UDMI](https://github.com/faucetsdn/udmi)
    a short form can be used.

<!---
###### Compliant Short forms

**Forms in this section are as-yet unsupported**

Because UDMI strictly defines the path to points and units in the payload, as
well as name correspondence between `units` and `present_value` much of the
information in the translation definition can be omitted (The parser assumes
UDMI compliant payloads are the default).

``` yaml
FCU-123:
  ...
  translation:
    zone_air_temperature_sensor:
      present_value: "temp_1"
      unit_values:
        degrees_celsius: "degC"
    supply_air_isolation_damper_command:
      present_value: "damper_1"
      states:
        OPEN: "1"
        CLOSED: "2"
```

For any subsection that has a 1:1 translation to standard values, the section
can be completed in shorthand with `COMPLIANT`

The more compliant data the device has, the smaller the translation can be. The
same device with Digital Building ontology standard state and unit values looks
like:

``` yaml
FCU-123:
  ...
  translation:
    zone_air_temperature_sensor: "temp_1"
    supply_air_isolation_damper_command: "damper_1"
```

A device that is UDMI compliant and uses standard values from the ontology can
simply specify (not yet supported):

``` yaml
FCU-123:
  ...
  translation: COMPLIANT
```
--->
<!--
** metadata has not been implemented as of yet; this is an addition
   due to ABEL **
##### Metadata

Often it is useful to include metadata about devices in our model (and the
provided abstract model includes metadata points in its types).  To do this in
the building config, one can add a `metadata` block on an entity description:

``` yaml
FCU-123:
  ...
  metadata:
    discharge_air_flowrate_capacity:
      present_value:"300"
      units: "cubic_feet_per_minute"
    manufacturer_label: "Carrier"
```

For entries with dimensional units, `present_value` and `units` subentries must
be specified.  For strings or point types with implied units, the value may be
inlined.  Because metadata is user-specified, it is required that fields and
unit names all be in standard form.
-->

#### Virtual Devices

In many cases the logical device and the reporting device are the same thing
(i.e. the device that sends data is also the entity type you want in your
model). When the two are different (i.e. the logical device does not send its
own data) a virtual device is required.

Virtual entities are nearly always logical entities. Because logical entities
are the representations that the applications using your model will care about,
virtual devices should generally map to canonical types. In an integrated design
in construction stack, logical devices should be called out by code in the
CAD/BIM models and the same code should be used for them in this config.

A virtual entity example:

``` yaml
VAV-32:
  type: NAMESPACE/DEVICE_TYPE
  links:
    ANOTHER-ENTITY: # source device
      # target_device_field: source_device_field
      supply_air_damper_position_command: supply_air_damper_command_1
      ...
```

The key difference between virtual and reporting entities is that all the fields
of a virtual entity are derived via `links`. the link block lists all the source
entities and fields that contribute to this entity's data and provides a mapping
to this entity's local fields.

#### Device Relationships

In addition to telemetry points, many devices will have relationships to other
entities. System and spatial relationships are defined with the `connections`
block. Connection definitions work the same way for all entities, with
connections always defined on the target of the connection.

Expanding the VAV definition from the previous section, and adding some lights:

``` yaml
VAV-32:
  type: HVAC/VAV
  connections:
    UK-LON-6PS-1: CONTAINS
    AHU-123: FEEDS

LF-123:
  type: LIGHTING/LIGHTING_FIXTURE
  connections:
    UK-LON-6PS-1-1A2: CONTAINS
    LCG-234: HAS_PART
```

### Zones and Control Groups

In addition to spaces and devices, most buildings have other types of Logically
defined areas or groups that aren't strictly a "device". These entities are
simply virtual devices, following all the same rules, but often do not have
`links`.

Here are some examples:

``` yaml
# HVAC Zone
ZONE-123:
  type: HVAC/ZONE
  connections:
    UK-LON-6PS-1: CONTAINS
    VAV-123: FEEDS

# Lighting Control Group
LCG-234:
  type: LIGHTING/SWITCH_GROUP
  connections:
    UK-LON-6PS-1: CONTAINS
    SW-456: CONTROLS
```

## Building Configuration Modes
This document details the building configuration modes available in v1 Beta.
The current version, v1 Alpha, is available with limited functionality.

 * v1 Alpha:
  - **Interdependencies** among entities are **not allowed**
  - At most **2 different operations** throughout all entities in the same building
    configuration; **one of them being EXPORT**.
  - Instance Validator will reject a building config when more than 2 operations
    are specified building config and EXPORT is not present.
<!--
 * v1 Beta:
  - **Interdependencies** among entities are **allowed**
  - **No constraints for operations** on entities in a building configuration
-->
A building configuration file, with regard to data representation, can be
grouped into two mutually exclusive parts: 

1. a configuration metadata block that informs the program how to handle the 
   entities contained in the building configuration.

2. Entity block(s) derived from a physical building; they model physical and
   [virtual devices](#virutal-devices) within a building. For more information
   please refer to the [learning modules](../../README.md#learning-modules).

The configuration block, keyed by `CONFIG_METADATA`, currently allows for
a single field `operation` with values of `INITIALIZE` or `UPDATE`. This is
shown below for a general entity coded as `E` and an `UPDATE` configuration
operation.

``` yaml
CONFIG_METADATA:
  operation: UPDATE

E-GUID:
  type: ...
  code: E
  etag: ...
  cloud_device_id: ...
  translation: ...
  connections: ...
  links: ...
  operation: [ ADD | DELETE | UPDATE | EXPORT ]
  <update_mask: >

  .
  .
  .
```

There is a specification for `operation` in both the configuration metadata
block as well as the entity blocks contained within the building configuration
file. Although they share the same attribute name, their meanings differ. The
`operation` contained within the CONFIG_METADATA block specifies the program
workflow for the building configuration file as a whole. Whereas the attribute
`operation` of an entity details what is being done to that particular entity;
with an `update_mask` (if applicable).

An additional attribute `etag` has been included in each entity. The etag is
required for all entities under an `UPDATE` configuration and used to sync
proposed changes with the current state (onboarded buildings). Under this
scenario, in addition to standard offline validation checks, etags are compared
between the building configuration file and backend datastore. Successfully
validating the proposed update if all etags align; signaling that the update
building configuration is synced to the current state. The etags are generated
upon export and are not created by user; i.e. updates can only be applied to
a previously exported building configuration.

In the context of a building configuration file, the program is agnostic to the
location of the configuration metadata block. However, the standard practice is
to place this at the very top of the building configuration.

``` yaml
CONFIG_METADATA:
  operation: INITIALIZE
  
E1-GUID:
  type: ...
  code: E1
  ...
  
E2-GUID:
  type: ...
  code: E2
  ...
  
E3-GUID:
  type: ...
  code: E3
  ...
```

### INITIALIZE
Default operation for modeling a building yet to be onboarded. In this
mode of operation the program will treat the entire contents of the building
configuration as a new instantiation. The `operation` attribute for each entity
contained within the building config will default to `ADD`; unless otherwise
specified. Allowed entity operation values in this mode are of `ADD` or `EXPORT`
and is enforced by the instance validator. The building config must pass all
standard offline validation checks. If the `CONFIG_METADATA` block is missing
from the building configuration the program will assume this mode of operation.
An `INITIALIZE` configuration only allows `EXPORT` operations on previously onboarded entities. This is shown below with specified operations for entities E1, E2, and an export of building entity E3; note that there is no specification of operation for entity E2.

``` yaml
CONFIG_METADATA:
  operation: INITIALIZE
  
E1-GUID:
  type: ...
  code: E1
  operation: ADD
  ...
  
E2-GUID: # no operation specified, ADD assumed
  type: ...
  code: E2
  connections:
    E3-GUID: CONNECTS_TO  # reference connection to an onboarded building
  ...
  
E3-GUID: # export an onboarded building as its referenced by E2
  type: FACILITIES/BUILDING
  code: E3
  operation: EXPORT
  ...
```

### UPDATE
In this mode the configuration will be considered as a scoped view of an already
instantiated building; allowing users to perform operations of `ADD`,
`DELETE`, `UPDATE`, and `EXPORT` on entities. This follows the copy-and-replace
strategy:
  - Missing parts from the block will be removed.
  - New parts in the block will be added.
  - Existing parts in the block will be updated.

Under the `UPDATE` configuration:
  * Any entity without a specification for operation or update_mask will default
    to `EXPORT`. Therefore the following two entities are equivalent

    ``` yaml
    E-GUID: # This definition for E is equivalent to the one below
    type: ...
    code: E
    operation: EXPORT
    ...

    E-GUID:
    type: ...
    code: E
    ...
    ```

  * The `EXPORT` operation:
      - Results in no change to the instantiated entity; i.e. no-op
      - Must be used to include entities which are referenced by others
        undergoing update

      ``` yaml
      E2-GUID: # E0-GUID is updating its source to E2-GUID, it must be included
      type: ...
      code: E2

      E0-GUID:
      type: ...
      code: E3
      operation: UPDATE
      update_mask:
        - connections
      connections:
        E2-GUID: CONNECTS_TO
      ...
      ```

  * If the update_mask is specified, the operation for that entity is assumed
    `UPDATE`.

    ``` yaml
    E-GUID: # These two entity definitions for E are equivalent
    type: ...
    code: E
    operation: UPDATE
    update_mask:
      - ...
    ...

    E-GUID:
    type: ...
    code: E
    update_mask:
      - ...
    ...
    ```

This mode, with an entity operation of `UPDATE`, introduces another attribute
to the entity blocks keyed by `update_mask`. This is shown in the example for
an entity coded as E4, showing all possible entity attributes, where
`update_mask` is given with fields of `translation` and `cloud_device_id`.

``` yaml
CONFIG_METADATA:
  operation: UPDATE

E4-GUID:
  type: ...
  code: E4
  etag: ...
  cloud_device_id: ...
  translation: ...
  connections: ...
  links: ...
  operation: UPDATE # This can be omitted when update_mask is specified
  update_mask:
    - translation
    - cloud_device_id
```

There are various types of updates that can be performed. All of this updates
are performed according to the fields provided in the `update_mask`. They are
grouped in the following categories:

1. [type](#type-update)
2. [code](#code-update)
3. [translation](#translation-update)
4. [connections](connections-update)
5. [links](#links-update)
<!-- 6. [cloud_device_id](#cloud-device-id-update)
 -->
#### Type update
This update typically occurs alongside an update to the `translation` or `links`
. If the update to another type contains the same fields, as the previous type,
this can be done independently. The entity type can be changed to any valid
identifier but cannot be cleared. This is shown in the description for [Code update](#code-update).

#### Code update
This update may be completed independently; however, it typically follows a type
update. In the following example: the code is updated from `FAN-123` to 
`PMP-123` which follows a change of type from HVAC/FAN_SS to HVAC/PMP_SS. Since
the two types share the same fields there is no need to make updates to the
translations.

  * Instantiated Entity:
    ``` yaml
    GUID-123:
      type: HVAC/FAN_SS
      code: FAN-123
      ...
    ```

  * Update:
    ``` yaml
    GUID-123: 
      type: HVAC/PMP_SS
      code: PMP-123
      ...
      update_mask:
        - type
        - code
    ```

<!-- Incognito Mode pending outcome of guid requirement spike
#### Cloud Device ID update
This update requires the presence of the fully specified translation field for
validation purposes; regardless if it is being updated. The cloud device id
corresponds to the numeric id in the cloud registry. Often, when a device is
replaced after a malfunction for example, only its numeric id is updated and the
other parameters remain the same. Therefore, this field can be updated as
follows:

``` yaml
ENTITY-123-GUID:
  type: ...
  code: ENTITY-123
  etag: ...
  cloud_device_id: NEW-CLOUD-DEVICE-ID
  update_mask: 
    - cloud_device_id
  translations:
    ...
```
-->

#### Translation update
In general, a type update requires an update of the translation. These updates
are completed on the standard field by the following: replacement and mapping,
addition, or removal. For example an entity coded as `ENTITY-CODE` with
the required fields `field_a`, `field_b`, and `field_c`:

``` yaml
ENTITY-GUID:
  type: SOME_TYPE_NAMESPACE/SOME_TYPE_A
  code: ENTITY-CODE
  etag: ...
  cloud_device_id: NEW-CLOUD-DEVICE-ID
  translation:
    field_a: 
      present_value: points.analog-value_1.present_value
      units: …
    field_b: 
      present_value: points.analog-value_2.present_value
      units: …
    field_c: 
      present_value: points.analog-value_3.present_value
      units: …
```

To change the type from SOME_TYPE_A to SOME_TYPE_B in the same namespace
requiring fields: `field_b`, `field_c`, and `field_d`. This can be done as
follows:

``` yaml
ENTITY-GUID:
  type: SOME_TYPE_NAMESPACE/SOME_TYPE_B
  code: ...
  etag: ...
  cloud_device_id: ...
  translation:
    field_b: 
      present_value: points.analog-value_2.present_value
      units: …
    field_c: 
      present_value: points.analog-value_3.present_value
      units: …
    field_d: 
      present_value: points.analog-value_4.present_value
      units: …
  update_mask:
    - type
    - translation
```

Note: a type update typically requires a code update. This has been omitted from
the example for brevity. Any field that is omitted is deleted, not present in
the instantiated entity is added, or present is ignored.

#### Connections update
The entity's connections can be updated to reflect a new connection type,
source, or can be cleared. An update to both the source entity and connection
type together is allowed. For example: an entity coded as `LF-123` with a
connection to source entity coded as `LCG-123`:

``` yaml
LF-123-GUID
  type: LIGHTING/LIGHTING_FIXTURE
  code: LF-123
  etag: ...
  cloud_device_id: ...
  connections:
    LCG-123-GUID: HAS_PART
```

The connection type can be updated from `HAS_PART` to `CONNECTS_TO` as:

``` yaml
LF-123-GUID
  type: LIGHTING/LIGHTING_FIXTURE
  code: LF-123
  etag: ...
  cloud_device_id: ...
  connections:
    LCG-123-GUID: CONNECTS_TO
  update_mask:
    - connections
```

The connection source can be updated from `LCG-123-GUID` to `LCG-234-GUID` as:

``` yaml
LF-123-GUID
  type: LIGHTING/LIGHTING_FIXTURE
  code: LF-123
  etag: ...
  cloud_device_id: ...
  connections:
    LCG-234-GUID: HAS_PART
  update_mask:
    - connections
```

Or all connection can be cleared by specifying connections in the update_mask
and no specification for connections:

``` yaml
LF-123-GUID
  type: LIGHTING/LIGHTING_FIXTURE
  code: LF-123
  etag: ...
  cloud_device_id: ...
  update_mask:
    - connections
```

#### Links update
The links exist on virtual entities and are tied to physical entities and more
precisely to the standard fields of a given translation [Virtual Devices](#virtual-devices).
Therefore, when an entity type is updated with a change applied at the
translation level, the links must be updated. The links update follows the same
concept as the translation depicted earlier. For example: a virtual entity coded
as `VIRTUAL-ENTITY-123` of `TYPE-C` is linked to another coded as `ENTITY-123` with
type `TYPE-A` and fields: `field_a`, `field_b`, `field_c`.

``` yaml
ENTITY-A-123-GUID:
  type: SOME_TYPE_NAMESPACE/TYPE-A
  code: ENTITY-A-123
  etag: ...
  cloud_device_id: ...
  translation:
    field_a: 
      present_value: points.analog-value_1.present_value
      units: ...
    field_b: 
      present_value: points.analog-value_2.present_value
      units: ...
    field_c: 
      present_value: points.analog-value_3.present_value
      units: ...   

VIRTUAL-ENTITY-C-123-GUID:
  type: SOME_TYPE_NAMESPACE/TYPE-C
  code: VIRTUAL-ENTITY-C-123
  etag: ...
  cloud_device_id: ...
  links:
  entity-123:
    field_a: field_a_1
    field_b: field_b_1
    field_c: field_c_1
```

It is required to change the type of ENTITY-123 from TYPE-A to TYPE-B with
fields: field_b, field_c, and field_d; necessitating an update also to its code
and links. In addition VIRTUAL-ENTITY-123, which depends on ENTITY-123, must be
updated. This can be achieved by the following:

``` yaml
ENTITY-A-123-GUID:
  type: SOME_TYPE_NAMESPACE/TYPE-B
  code: ENTITY-B-123
  etag: ...
  cloud_device_id: ...
  translation:
    field_b: 
      present_value: points.analog-value_1.present_value
      units: ...
    field_c: 
      present_value: points.analog-value_2.present_value
      units: ...
    field_d: 
      present_value: points.analog-value_3.present_value
      units: ...
  update_mask:
    - type
    - code
    - translation

VIRTUAL-ENTITY-C-123-GUID:
  type: SOME_TYPE_NAMESPACE/TYPE-D
  code: VIRTUAL-ENTITY-D-123
  links:
    ENTITY-A-123-GUID:
      field_b: field_b_1
      field_c: field_c_1
      field_d: field_d_1
  update_mask:
    - type
    - links
```

## Validation
The building config can be machine validated for consistency and adherence to
the rules defined in the data model. This tool is available [here](https://github.com/google/digitalbuildings/tree/master/tools/validators/instance_validator).

<!-- Footnotes themselves at the bottom. -->

## Notes

[^1]: Practically speaking, the entry typically corresponds to the controller,
    which may or may not map to exactly one complete piece of equipment. \
[^2]:
     It is anticipated that non 1:1 mapping will only apply to HVAC equipment. \
[^3]:
     A fully compliant UDMI payload uses standard names for units, formatted
     with points in the payload using the fully qualified path
     `points.<name>.present_value` and units in metadata with the path `
     pointset.points.<name>.units`.  A UDMI compliant field will use the exact
     text (case insensitive) for multi-states, and all states defined for the
     field will be used. \
[^5]: A user would almost never define all the sections in a single entity, but
    all are shown below for illustration purposes. \
[^6]: An unstated assumption here is that the CDM registry scope and the scope
    of the config file are the same. This guarantees that a unique ID can
    be found in the CDM registry for each entity.