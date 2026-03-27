# WP-2: Event Schema Validation -- Completion Report

**Branch:** `production-hardening-wp-2-event-schema`
**Status:** Complete
**Date:** 2026-02-27

## Summary

All 67 NATS JetStream domain events across 15 domains now have strict Pydantic
schema validation (`extra="forbid"`) with a typed event registry, typed
consumer deserialization, and publish-time validation.  This eliminates an
entire class of silent data-loss bugs where misspelled or extra fields were
silently dropped.

## Changes

### 1. BaseEvent strict mode (`shared/events/base.py`)

- Added `model_config = ConfigDict(extra="forbid")` -- any undeclared field
  on any event subclass now raises `ValidationError` at construction and
  deserialization time.
- Added `schema_version: int = 1` field with `field_validator` enforcing
  `>= 1`.  This enables future schema evolution without breaking consumers.

### 2. Event registry (`shared/events/registry.py`)

- New file: lazy-loaded registry mapping 67 `event_type` strings to their
  typed Pydantic model classes across all 15 domains.
- Lazy initialization via `_ensure_registry()` avoids circular imports
  (domain `events.py` files import `BaseEvent` from the same package).
- API:
  - `get_event_class(event_type)` -- raises `ValueError` for unknown types
  - `get_event_class_optional(event_type)` -- returns `None` for unknown types
  - `all_registered()` -- returns full mapping dict
  - `register_event(event_type, cls)` -- dynamic registration

### 3. Event bus validation (`shared/event_bus.py`)

- **Publish validation:** `publish()` now calls
  `event.__class__.model_validate(event.model_dump(mode="json"))` before
  serialization, catching schema violations at the source.
- **Typed deserialization:** `_msg_handler` in `subscribe_jetstream()` now
  resolves the `event_type` from message metadata, looks up the typed class
  via the registry, and calls `event_cls.model_validate(data)` instead of
  constructing a generic `BaseEvent`.  Consumers receive fully typed event
  objects.

### 4. Consumer migration (15 domains, ~38 handlers)

All consumer handler signatures updated from `BaseEvent` to specific typed
event classes.  Where possible, `event.model_dump(mode="json")` + dict
`payload.get()` patterns replaced with direct typed attribute access
(`event.shipment_id`, `event.hs_code`, etc.).

| Domain | Handlers | Typed Import |
|--------|----------|--------------|
| shipment_lifecycle | 5 | OrderShippedEvent, HUStatusChangedEvent, ClassificationProducedEvent, TariffCalculatedEvent, ComplianceScreenedEvent |
| cargo_handling_units | 4 | ShipmentCreatedEvent, ShipmentStatusChangedEvent, AdjudicationDecisionEvent, ConsolidationMovementEvent |
| compliance | 4 | ShipmentCreatedEvent, ClassificationProducedEvent, ProductCreatedEvent, ScreeningListUpdatedEvent |
| consolidation | 2 | HUCreatedEvent, HUStatusChangedEvent |
| customs_adjudication | 2 | DeclarationSubmittedEvent, ExportDeclarationSubmittedEvent |
| declaration_management | 4 | HUCreatedEvent, ClassificationProducedEvent, TariffCalculatedEvent, ComplianceScreenedEvent |
| document_management | 2 | ShipmentCreatedEvent, DeclarationSubmittedEvent |
| exception_management | 2 | AdjudicationDecisionEvent, HUStatusChangedEvent |
| financial_settlement | 4 | TariffCalculatedEvent, DeclarationSubmittedEvent, AdjudicationDecisionEvent, HUCageIntakeEvent |
| order_management | 1 | ShipmentStatusChangedEvent |
| party_management | 1 | OrderCreatedEvent |
| product_catalog | 1 | RegulatorySignalDetectedEvent |
| regulatory_intelligence | 1 | ComplianceScreenedEvent |
| supply_chain_disruptions | 1 | RegulatorySignalDetectedEvent |
| trade_intelligence | 4 | ProductCreatedEvent, ProductUpdatedEvent, ShipmentCreatedEvent, RegulatorySignalDetectedEvent |

A few handlers retain `event.model_dump(mode="json")` in the body where
downstream functions (e.g. `_run_selectivity(payload)`) expect dict input.
Handler signatures are still typed.

### 5. Test updates

- Updated `test_domain_structure.py::TestEventRegistry::test_register_and_lookup`
  to match new `get_event_class` behavior (raises `ValueError` instead of
  returning `None`).

### 6. New tests (`tests/unit/test_event_schema_validation.py`)

91 tests covering:

| Test class | Count | Validates |
|-----------|-------|-----------|
| TestBaseEventRejectsExtraFields | 3 | `extra="forbid"` at construction, `model_validate`, and on typed subclasses |
| TestSchemaVersionValidation | 4 | Rejects 0, negatives; accepts default=1 and explicit positive |
| TestTypedEventRequiredFields | 4 | Missing required fields on ShipmentCreated, ClassificationProduced, DeclarationSubmitted, AdjudicationDecision |
| TestTypedEventRoundTrip | 5 | Serialize/deserialize round-trip for 5 event types including JSON intermediate |
| TestEventRegistryResolution | 71 | 67 parameterized + registry size + subclass check + unknown type handling + spot checks |
| TestEventBusPublishValidation | 2 | Publish dispatches; typed event publish succeeds |
| TestEventBusTypedDeserialization | 1 | In-process dispatch delivers typed event instance |

All 91 tests pass.

## Risk Assessment

- **Low risk:** `extra="forbid"` only affects malformed payloads that were
  previously silently losing data.  Valid payloads are unaffected.
- **Compatibility:** The `get_event_class_optional()` function provides a
  graceful fallback path for unknown event types during rolling deploys.
  The event bus uses this to fall back to `BaseEvent` when the registry
  doesn't recognize a type.
- **Schema evolution:** The `schema_version` field enables future
  version-aware deserialization without a flag day.

## Files Modified

```
clearance_platform/shared/events/base.py
clearance_platform/shared/events/registry.py          (new)
clearance_platform/shared/events/__init__.py
clearance_platform/shared/event_bus.py
clearance_platform/domains/*/consumers.py              (15 files)
tests/unit/test_event_schema_validation.py             (new)
tests/unit/test_domain_structure.py                    (1 test updated)
```
