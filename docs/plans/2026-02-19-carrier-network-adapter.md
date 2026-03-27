# Carrier Network Adapter Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a pluggable carrier logistics network adapter pattern with a FedEx implementation, and an async ingestion driver + CLI script that imports 211K FedEx sample shipments through the full clearance pipeline.

**Architecture:** A `CarrierNetworkAdapter` ABC in `clearance_platform/adapters/carrier_network/` translates carrier-native manifest records into a platform-neutral `ManifestRecord`. The FedEx implementation maps AWB manifest fields. `CreateShipmentRequest` is extended with optional carrier-supplied fields so the ingestion requires a single API call. Carrier imports set `initial_status="booked"` so every shipment enters at the head of the analysis pipeline (preclearance → compliance → broker queue). UI-initiated creation continues to start at `"in_transit"` unchanged.

**Tech Stack:** Python dataclasses, ABC, asyncio, httpx, pandas (import script only). No new dependencies — pandas is already in the dev stack.

---

## Task 1: Base abstractions — `ManifestRecord` + `CarrierNetworkAdapter` ABC

**Files:**
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/__init__.py`
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/base.py`
- Create: `clearance-engine/backend/tests/unit/adapters/carrier_network/__init__.py`
- Create: `clearance-engine/backend/tests/unit/adapters/carrier_network/test_base.py`

**Step 1: Write the failing test**

```python
# tests/unit/adapters/carrier_network/test_base.py
from dataclasses import asdict
import pytest
from clearance_platform.adapters.carrier_network.base import (
    CarrierNetworkAdapter,
    ManifestRecord,
)


def test_manifest_record_required_fields():
    r = ManifestRecord(
        product_description="Widget",
        origin_country="CN",
        destination_country="US",
        declared_value=99.99,
        carrier_id="fedex",
        tracking_number="1234567890",
        carrier_reference="1234567890",
    )
    assert r.product_description == "Widget"
    assert r.hs_code is None
    assert r.cargo is None
    # Carrier imports default to booked — head of the analysis pipeline
    assert r.initial_status == "booked"


def test_manifest_record_optional_fields():
    r = ManifestRecord(
        product_description="Widget",
        origin_country="CN",
        destination_country="US",
        declared_value=99.99,
        carrier_id="fedex",
        tracking_number="1234567890",
        carrier_reference="1234567890",
        hs_code="6110.20.2015",
        transport_mode="air",
        cargo={"pieces": 3, "weight_kg": 1.5},
    )
    assert r.hs_code == "6110.20.2015"
    assert r.cargo["pieces"] == 3


def test_carrier_network_adapter_is_abstract():
    with pytest.raises(TypeError):
        CarrierNetworkAdapter()  # type: ignore[abstract]


def test_translate_batch_default_calls_translate():
    class _Stub(CarrierNetworkAdapter):
        carrier_id = "stub"
        def translate(self, raw: dict) -> ManifestRecord:
            return ManifestRecord(
                product_description=raw["desc"],
                origin_country="CN",
                destination_country="US",
                declared_value=1.0,
                carrier_id="stub",
                tracking_number=raw["awb"],
                carrier_reference=raw["awb"],
            )

    adapter = _Stub()
    records = [{"desc": "A", "awb": "001"}, {"desc": "B", "awb": "002"}]
    result = adapter.translate_batch(records)
    assert len(result) == 2
    assert result[0].tracking_number == "001"
    assert result[1].product_description == "B"
```

**Step 2: Run test to verify it fails**

```bash
cd clearance-engine/backend
python -m pytest tests/unit/adapters/carrier_network/test_base.py -v
```
Expected: `ModuleNotFoundError` or `ImportError`

**Step 3: Create the package init and base module**

```python
# clearance_platform/adapters/carrier_network/__init__.py
"""Pluggable carrier logistics network adapters.

Each sub-package integrates with a specific carrier's logistics network,
translating carrier-native manifest formats into platform ManifestRecord objects.

Distinct from ``adapters/fedex/`` which is the FedEx Dataworks data partnership
(HS classification API) — these adapters are for carrier logistics networks.

Current adapters:
- ``fedex`` — FedEx logistics carrier network
"""

from clearance_platform.adapters.carrier_network.registry import build as _build


def build_carrier_network_adapter(carrier_id: str):
    """Instantiate the registered adapter for the given carrier ID.

    Importing this function triggers lazy registration of known adapters.
    """
    if carrier_id == "fedex":
        from clearance_platform.adapters.carrier_network.fedex import adapter as _  # noqa: F401
    return _build(carrier_id)
```

```python
# clearance_platform/adapters/carrier_network/base.py
"""Carrier network adapter abstractions.

CarrierNetworkAdapterConfig — base settings class every adapter config extends.
CarrierNetworkAdapter — ABC all carrier adapters must implement.
ManifestRecord — platform-neutral shipment record produced by any adapter.
"""
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from pydantic_settings import BaseSettings, SettingsConfigDict


class CarrierNetworkAdapterConfig(BaseSettings):
    """Base configuration for carrier network adapters.

    Each carrier's config subclass extends this and reads its own env vars.
    ``source_id`` is written to ``Shipment.shipment_source`` so shipments
    can be filtered by origin (e.g. simulation actors exclude carrier imports).
    ``carrier_name`` is the human-readable carrier name used in UI and logs.
    """

    model_config = SettingsConfigDict(
        env_file=("../.env", ".env"),
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    source_id: str = "carrier_network"    # override in subclass
    carrier_name: str = "Unknown Carrier" # override in subclass


@dataclass
class ManifestRecord:
    """Platform-neutral representation of one shipment from a carrier manifest.

    Required fields map directly to ``CreateShipmentRequest``.
    Optional fields are applied when present, avoiding a second update-fields call.
    """

    # Required
    product_description: str
    origin_country: str       # ISO 2-letter
    destination_country: str  # ISO 2-letter
    declared_value: float
    carrier_id: str           # e.g. "fedex", "ups"
    tracking_number: str      # AWB / pro number / tracking ID
    carrier_reference: str    # deduplication key (usually == tracking_number)

    # Optional enrichment
    carrier: str | None = None
    transport_mode: str | None = None   # "air" | "ocean" | "ground"
    hs_code: str | None = None          # carrier-supplied hint to E1
    country_of_manufacture: str | None = None
    incoterms: str | None = None
    company_name: str | None = None
    ship_date: str | None = None        # ISO 8601 datetime string
    cargo: dict | None = None           # weight_kg, pieces, quantity, quantity_uom
    parties: dict | None = None         # shipper, consignee, importer dicts
    license_numbers: dict | None = None # import/export license refs

    # Pipeline entry point — carrier imports start at "booked" so they flow
    # through preclearance → compliance → broker queue.
    # UI-initiated creation uses "in_transit" (the service default).
    initial_status: str = "booked"

    # Source identifier — set from the adapter's config.source_id.
    # Written to the Shipment.shipment_source column so simulation actors
    # can filter to only their own shipments (shipment_source IS NULL).
    shipment_source: str = "carrier_network"


class CarrierNetworkAdapter(ABC):
    """Abstract base for all carrier logistics network adapters."""

    carrier_id: str  # must be overridden by subclass

    def __init__(self, config: CarrierNetworkAdapterConfig | None = None) -> None:
        self.config = config or CarrierNetworkAdapterConfig()

    @property
    def source_id(self) -> str:
        return self.config.source_id

    @property
    def carrier_name(self) -> str:
        return self.config.carrier_name

    @abstractmethod
    def translate(self, raw: dict) -> ManifestRecord:
        """Translate one carrier-native record into a ManifestRecord."""

    def translate_batch(self, records: list[dict]) -> list[ManifestRecord]:
        """Translate multiple records. Default: calls translate() per record."""
        return [self.translate(r) for r in records]
```

**Step 4: Run test to verify it passes**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_base.py -v
```
Expected: 4 PASSED

**Step 5: Commit**

```bash
git add clearance_platform/adapters/carrier_network/__init__.py \
        clearance_platform/adapters/carrier_network/base.py \
        tests/unit/adapters/carrier_network/__init__.py \
        tests/unit/adapters/carrier_network/test_base.py
git commit -m "feat: add CarrierNetworkAdapter ABC and ManifestRecord"
```

---

## Task 2: Registry + factory

**Files:**
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/registry.py`
- Modify: `clearance-engine/backend/tests/unit/adapters/carrier_network/test_base.py` (add registry tests)

**Step 1: Write the failing tests**

Append to `test_base.py`:

```python
from clearance_platform.adapters.carrier_network.registry import register, build


def test_register_and_build():
    class _TestAdapter(CarrierNetworkAdapter):
        carrier_id = "test_carrier"
        def translate(self, raw: dict) -> ManifestRecord:
            raise NotImplementedError

    register("test_carrier", _TestAdapter)
    adapter = build("test_carrier")
    assert isinstance(adapter, _TestAdapter)


def test_build_unknown_carrier_raises():
    with pytest.raises(KeyError, match="unknown_xyz"):
        build("unknown_xyz")
```

**Step 2: Run to verify they fail**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_base.py::test_register_and_build -v
```
Expected: `ImportError`

**Step 3: Create registry**

```python
# clearance_platform/adapters/carrier_network/registry.py
"""Carrier ID → adapter class registry."""
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from clearance_platform.adapters.carrier_network.base import CarrierNetworkAdapter

_REGISTRY: dict[str, type] = {}


def register(carrier_id: str, adapter_class: type) -> None:
    """Register an adapter class for a carrier ID. Called by each adapter module."""
    _REGISTRY[carrier_id] = adapter_class


def build(carrier_id: str) -> "CarrierNetworkAdapter":
    """Instantiate the adapter for the given carrier ID.

    Raises:
        KeyError: If no adapter is registered for carrier_id.
    """
    if carrier_id not in _REGISTRY:
        raise KeyError(
            f"No carrier network adapter registered for '{carrier_id}'. "
            f"Available: {list(_REGISTRY.keys())}"
        )
    return _REGISTRY[carrier_id]()
```

**Step 4: Run tests to verify they pass**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_base.py -v
```
Expected: all PASSED

**Step 5: Commit**

```bash
git add clearance_platform/adapters/carrier_network/registry.py \
        tests/unit/adapters/carrier_network/test_base.py
git commit -m "feat: add carrier network adapter registry"
```

---

## Task 3: FedEx field map

**Files:**
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/fedex/__init__.py`
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/fedex/field_map.py`
- Create: `clearance-engine/backend/tests/unit/adapters/carrier_network/test_fedex_field_map.py`

**Step 1: Write the failing tests**

```python
# tests/unit/adapters/carrier_network/test_fedex_field_map.py
import pytest
from clearance_platform.adapters.carrier_network.fedex.field_map import (
    build_cargo,
    build_license_numbers,
    build_parties,
    map_service_type_to_mode,
    normalise_awb,
    normalise_hs_code,
)


class TestNormaliseHsCode:
    def test_dotted_10_digit(self):
        assert normalise_hs_code("6110202015") == "6110.20.2015"

    def test_already_dotted(self):
        assert normalise_hs_code("6110.20.2015") == "6110.20.2015"

    def test_fewer_than_6_digits_returns_none(self):
        assert normalise_hs_code("611") is None

    def test_none_returns_none(self):
        assert normalise_hs_code(None) is None

    def test_float_input(self):
        # pandas reads some HS codes as floats
        assert normalise_hs_code("85444210005.0") == "8544.42.1000"

    def test_nan_float_returns_none(self):
        assert normalise_hs_code(float("nan")) is None


class TestNormaliseAwb:
    def test_strips_dot_zero(self):
        assert normalise_awb("165023616658.0") == "165023616658"

    def test_none_returns_none(self):
        assert normalise_awb(None) is None

    def test_plain_string(self):
        assert normalise_awb("165023616658") == "165023616658"


class TestMapServiceTypeToMode:
    def test_known_service_type(self):
        assert map_service_type_to_mode("FedEx International Priority") == "air"

    def test_unknown_defaults_to_air(self):
        assert map_service_type_to_mode("Unknown Service XYZ") == "air"

    def test_none_defaults_to_air(self):
        assert map_service_type_to_mode(None) == "air"


class TestBuildCargo:
    def test_full_record(self):
        raw = {
            "Commodity Weight": 2.5,
            "Commodity Weight UoM": "KG",
            "Commodity Quantity": 3.0,
            "Commodity Quantity UoM": "EA",
            "Number of Pieces": 1.0,
        }
        cargo = build_cargo(raw)
        assert cargo == {"weight_kg": 2.5, "quantity": 3.0, "quantity_uom": "EA", "pieces": 1}

    def test_all_null_returns_none(self):
        raw = {
            "Commodity Weight": float("nan"),
            "Commodity Quantity": None,
            "Number of Pieces": float("nan"),
        }
        assert build_cargo(raw) is None

    def test_lbs_converted_to_kg(self):
        raw = {"Commodity Weight": 10.0, "Commodity Weight UoM": "LB"}
        cargo = build_cargo(raw)
        assert cargo is not None
        assert abs(cargo["weight_kg"] - 4.54) < 0.01


class TestBuildParties:
    def test_shipper_extracted(self):
        raw = {
            "Shipper Company": "ACME Corp",
            "Shipper Name": "John Doe",
            "Shipper Address": "123 Main St",
            "Shipper Email": float("nan"),
        }
        parties = build_parties(raw)
        assert parties is not None
        assert parties["shipper"]["company"] == "ACME Corp"
        assert "email" not in parties["shipper"]

    def test_all_null_returns_none(self):
        raw = {"Shipper Company": None, "Consignee Company": float("nan")}
        assert build_parties(raw) is None


class TestBuildLicenseNumbers:
    def test_import_license(self):
        raw = {"import_license_number": "IMP-123", "export_license_number": None}
        result = build_license_numbers(raw)
        assert result == {"import_license_number": "IMP-123"}

    def test_all_null_returns_none(self):
        raw = {"import_license_number": None, "export_license_number": float("nan")}
        assert build_license_numbers(raw) is None
```

**Step 2: Run to verify they fail**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_field_map.py -v
```
Expected: `ImportError`

**Step 3: Create the field map module**

```python
# clearance_platform/adapters/carrier_network/fedex/__init__.py
"""FedEx carrier logistics network adapter."""
```

```python
# clearance_platform/adapters/carrier_network/fedex/field_map.py
"""FedEx manifest field → ManifestRecord field translations."""
from __future__ import annotations

import math

SERVICE_TYPE_TO_MODE: dict[str, str] = {
    "FedEx International Priority": "air",
    "FedEx International Economy": "air",
    "FedEx Intl Connect Plus": "air",
    "FedEx Intl Priority Express": "air",
    "FedEx International First": "air",
    "International Priority": "air",
    "International Priority Freight": "air",
    "International Economy Freight": "air",
    "TNT Intl Parcel End of Day": "air",
    "FedEx IP DirectDistribution": "air",
}


def map_service_type_to_mode(service_type: str | None) -> str:
    if not service_type:
        return "air"
    return SERVICE_TYPE_TO_MODE.get(service_type, "air")


def normalise_hs_code(raw) -> str | None:
    if raw is None:
        return None
    if isinstance(raw, float) and math.isnan(raw):
        return None
    digits = "".join(c for c in str(raw).split(".")[0] if c.isdigit())
    if len(digits) < 6:
        return None
    if len(digits) >= 10:
        return f"{digits[:4]}.{digits[4:6]}.{digits[6:10]}"
    if len(digits) >= 8:
        return f"{digits[:4]}.{digits[4:6]}.{digits[6:8]}"
    return f"{digits[:4]}.{digits[4:6]}"


def normalise_awb(raw) -> str | None:
    if raw is None:
        return None
    if isinstance(raw, float) and math.isnan(raw):
        return None
    s = str(raw).strip()
    if s.endswith(".0"):
        s = s[:-2]
    return s if s else None


def build_cargo(record: dict) -> dict | None:
    weight = record.get("Commodity Weight")
    quantity = record.get("Commodity Quantity")
    pieces = record.get("Number of Pieces")
    uom = record.get("Commodity Weight UoM", "KG")
    qty_uom = record.get("Commodity Quantity UoM")

    cargo: dict = {}
    if _num(pieces) is not None:
        cargo["pieces"] = int(_num(pieces))  # type: ignore[arg-type]
    if _num(weight) is not None:
        w = float(_num(weight))  # type: ignore[arg-type]
        cargo["weight_kg"] = w if str(uom).strip().upper() in ("KG", "K") else round(w * 0.453592, 2)
    if _num(quantity) is not None:
        cargo["quantity"] = float(_num(quantity))  # type: ignore[arg-type]
        if _present(qty_uom):
            cargo["quantity_uom"] = str(qty_uom)
    return cargo or None


def build_parties(record: dict) -> dict | None:
    parties: dict = {}
    for prefix in ("Shipper", "Consignee", "Importer"):
        party = _extract_party(record, prefix)
        if party:
            parties[prefix.lower()] = party
    return parties or None


def build_license_numbers(record: dict) -> dict | None:
    result: dict = {}
    if _present(record.get("import_license_number")):
        result["import_license_number"] = str(record["import_license_number"])
    if _present(record.get("export_license_number")):
        result["export_license_number"] = str(record["export_license_number"])
    if _present(record.get("export_control_class_nbr")):
        result["export_control_class"] = str(record["export_control_class_nbr"])
    return result or None


def _extract_party(record: dict, prefix: str) -> dict | None:
    party: dict = {}
    for suffix in ("Company", "Name", "Address", "Email", "Phone", "Account"):
        val = record.get(f"{prefix} {suffix}")
        if _present(val):
            party[suffix.lower()] = str(val)
    if prefix == "Importer" and _present(record.get("Importer Tax ID")):
        party["tax_id"] = str(record["Importer Tax ID"])
    return party or None


def _present(val) -> bool:
    if val is None:
        return False
    if isinstance(val, float) and math.isnan(val):
        return False
    return bool(str(val).strip())


def _num(val) -> float | None:
    if val is None:
        return None
    if isinstance(val, float) and math.isnan(val):
        return None
    try:
        return float(val)
    except (TypeError, ValueError):
        return None
```

**Step 4: Run tests to verify they pass**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_field_map.py -v
```
Expected: all PASSED

**Step 5: Commit**

```bash
git add clearance_platform/adapters/carrier_network/fedex/ \
        tests/unit/adapters/carrier_network/test_fedex_field_map.py
git commit -m "feat: add FedEx carrier network field map"
```

---

## Task 3.5: FedEx carrier network config

**Files:**
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/fedex/config.py`
- Modify: `clearance-engine/backend/tests/unit/adapters/carrier_network/test_fedex_field_map.py` (add config tests)

**Step 1: Write the failing tests**

Append to `test_fedex_field_map.py`:

```python
def test_fedex_carrier_config_defaults():
    from clearance_platform.adapters.carrier_network.fedex.config import FedExCarrierNetworkConfig
    cfg = FedExCarrierNetworkConfig()
    assert cfg.source_id == "fedex_carrier_network"
    assert cfg.carrier_name == "FedEx"

def test_fedex_carrier_config_overridable(monkeypatch):
    monkeypatch.setenv("FEDEX_CARRIER_SOURCE_ID", "fedex_custom")
    monkeypatch.setenv("FEDEX_CARRIER_NAME", "FedEx Express")
    from clearance_platform.adapters.carrier_network.fedex.config import FedExCarrierNetworkConfig
    cfg = FedExCarrierNetworkConfig()
    assert cfg.source_id == "fedex_custom"
    assert cfg.carrier_name == "FedEx Express"
```

**Step 2: Run to verify they fail**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_field_map.py::test_fedex_carrier_config_defaults -v
```
Expected: `ImportError`

**Step 3: Create the config**

```python
# clearance_platform/adapters/carrier_network/fedex/config.py
"""FedEx carrier network adapter configuration.

Reads from environment variables. Safe defaults mean the adapter works
out-of-the-box without any env config for local development and testing.

Environment variables
---------------------
FEDEX_CARRIER_SOURCE_ID
    Source identifier written to Shipment.shipment_source.
    Defaults to ``"fedex_carrier_network"``.
    Used to filter shipments by origin (e.g. exclude from simulation actors).
FEDEX_CARRIER_NAME
    Human-readable carrier name used in logs and UI.
    Defaults to ``"FedEx"``.
"""
from __future__ import annotations

from pydantic_settings import SettingsConfigDict
from clearance_platform.adapters.carrier_network.base import CarrierNetworkAdapterConfig


class FedExCarrierNetworkConfig(CarrierNetworkAdapterConfig):
    """FedEx logistics carrier network adapter settings."""

    model_config = SettingsConfigDict(
        env_file=("../.env", ".env"),
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    FEDEX_CARRIER_SOURCE_ID: str = "fedex_carrier_network"
    FEDEX_CARRIER_NAME: str = "FedEx"

    @property
    def source_id(self) -> str:  # type: ignore[override]
        return self.FEDEX_CARRIER_SOURCE_ID

    @property
    def carrier_name(self) -> str:  # type: ignore[override]
        return self.FEDEX_CARRIER_NAME
```

**Step 4: Run tests to verify they pass**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_field_map.py -v
```
Expected: all PASSED

**Step 5: Commit**

```bash
git add clearance_platform/adapters/carrier_network/fedex/config.py \
        tests/unit/adapters/carrier_network/test_fedex_field_map.py
git commit -m "feat: add FedExCarrierNetworkConfig with source_id and carrier_name"
```

---

## Task 4: FedEx carrier network adapter

**Files:**
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/fedex/adapter.py`
- Create: `clearance-engine/backend/tests/unit/adapters/carrier_network/test_fedex_adapter.py`

**Step 1: Write the failing tests**

```python
# tests/unit/adapters/carrier_network/test_fedex_adapter.py
import math
import pytest
from clearance_platform.adapters.carrier_network.fedex.adapter import FedExCarrierNetworkAdapter


MINIMAL_RAW = {
    "AWB Number": "165023616658",
    "Commodity Description": "FLEXIBLE CABLE",
    "Origin Country": "TW",
    "Customs Value": 1.0,
    "Service Type": "FedEx International Priority",
}


@pytest.fixture
def adapter():
    return FedExCarrierNetworkAdapter()


class TestTranslate:
    def test_minimal_record(self, adapter):
        r = adapter.translate(MINIMAL_RAW)
        assert r.tracking_number == "165023616658"
        assert r.carrier_reference == "165023616658"
        assert r.product_description == "FLEXIBLE CABLE"
        assert r.origin_country == "TW"
        assert r.destination_country == "US"
        assert r.declared_value == 1.0
        assert r.carrier_id == "fedex"
        assert r.transport_mode == "air"

    def test_hs_code_normalised(self, adapter):
        raw = {**MINIMAL_RAW, "HS Code": "85444210005"}
        r = adapter.translate(raw)
        assert r.hs_code == "8544.42.1000"

    def test_missing_hs_code_is_none(self, adapter):
        raw = {**MINIMAL_RAW, "HS Code": float("nan")}
        r = adapter.translate(raw)
        assert r.hs_code is None

    def test_awb_float_normalised(self, adapter):
        raw = {**MINIMAL_RAW, "AWB Number": 165023616658.0}
        r = adapter.translate(raw)
        assert r.tracking_number == "165023616658"

    def test_sheet_name_used_as_company(self, adapter):
        raw = {**MINIMAL_RAW, "_sheet_name": "Farfetch"}
        r = adapter.translate(raw)
        assert r.company_name == "Farfetch"

    def test_consignee_company_preferred_over_sheet_name(self, adapter):
        raw = {**MINIMAL_RAW, "Consignee Company": "GARMIN INT'L", "_sheet_name": "Sheet1"}
        r = adapter.translate(raw)
        assert r.company_name == "GARMIN INT'L"

    def test_zero_declared_value_becomes_one(self, adapter):
        raw = {**MINIMAL_RAW, "Customs Value": 0.0}
        r = adapter.translate(raw)
        assert r.declared_value == 1.0

    def test_missing_awb_raises(self, adapter):
        raw = {**MINIMAL_RAW, "AWB Number": None}
        with pytest.raises(ValueError, match="AWB Number"):
            adapter.translate(raw)

    def test_missing_description_raises(self, adapter):
        raw = {**MINIMAL_RAW, "Commodity Description": ""}
        with pytest.raises(ValueError, match="Commodity Description"):
            adapter.translate(raw)

    def test_parties_extracted(self, adapter):
        raw = {
            **MINIMAL_RAW,
            "Shipper Company": "GARMIN CORP",
            "Shipper Name": "JASON CHIEN",
        }
        r = adapter.translate(raw)
        assert r.parties is not None
        assert r.parties["shipper"]["company"] == "GARMIN CORP"

    def test_ship_date_iso_string(self, adapter):
        from datetime import datetime
        raw = {**MINIMAL_RAW, "Ship Date": datetime(2026, 2, 11)}
        r = adapter.translate(raw)
        assert r.ship_date == "2026-02-11T00:00:00"

    def test_carrier_id_registered(self):
        from clearance_platform.adapters.carrier_network.registry import build
        from clearance_platform.adapters.carrier_network.fedex import adapter as _  # noqa
        inst = build("fedex")
        assert inst.carrier_id == "fedex"
```

**Step 2: Run to verify they fail**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_adapter.py -v
```
Expected: `ImportError`

**Step 3: Create the adapter**

```python
# clearance_platform/adapters/carrier_network/fedex/adapter.py
"""FedEx carrier logistics network adapter."""
from __future__ import annotations

from clearance_platform.adapters.carrier_network.base import CarrierNetworkAdapter, CarrierNetworkAdapterConfig, ManifestRecord
from clearance_platform.adapters.carrier_network.fedex.config import FedExCarrierNetworkConfig
from clearance_platform.adapters.carrier_network.fedex.field_map import (
    build_cargo,
    build_license_numbers,
    build_parties,
    map_service_type_to_mode,
    normalise_awb,
    normalise_hs_code,
    _present,
)
from clearance_platform.adapters.carrier_network.registry import register


class FedExCarrierNetworkAdapter(CarrierNetworkAdapter):
    """Translates FedEx manifest records (CIE / CIP format) to ManifestRecord."""

    carrier_id = "fedex"

    def __init__(self, config: CarrierNetworkAdapterConfig | None = None) -> None:
        super().__init__(config or FedExCarrierNetworkConfig())

    def translate(self, raw: dict) -> ManifestRecord:
        awb = normalise_awb(raw.get("AWB Number"))
        if not awb:
            raise ValueError("AWB Number is required")

        desc = str(raw.get("Commodity Description") or "").strip()
        if not desc:
            raise ValueError("Commodity Description is required")

        origin = str(raw.get("Origin Country") or "").strip().upper()
        if not origin:
            raise ValueError("Origin Country is required")

        raw_value = raw.get("Customs Value")
        try:
            declared_value = float(raw_value) if raw_value is not None else 0.0
        except (TypeError, ValueError):
            declared_value = 0.0
        if declared_value <= 0:
            declared_value = 1.0  # de minimis placeholder normalisation

        service_type = raw.get("Service Type")

        # Resolve company name: consignee company preferred, fall back to sheet tag
        company: str | None = None
        if _present(raw.get("Consignee Company")):
            company = str(raw["Consignee Company"]).strip()
        elif _present(raw.get("_sheet_name")):
            company = str(raw["_sheet_name"]).strip()

        ship_date = raw.get("Ship Date")
        if hasattr(ship_date, "isoformat"):
            ship_date = ship_date.isoformat()

        com = str(raw.get("Country of Manufacture") or "").strip().upper() or None
        incoterms = str(raw.get("Terms of Sale") or "").strip() or None

        return ManifestRecord(
            product_description=desc,
            origin_country=origin,
            destination_country="US",
            declared_value=declared_value,
            carrier_id="fedex",
            tracking_number=awb,
            carrier_reference=awb,
            carrier=str(service_type).strip() if service_type else self.carrier_name,
            transport_mode=map_service_type_to_mode(str(service_type) if service_type else None),
            hs_code=normalise_hs_code(raw.get("HS Code")),
            country_of_manufacture=com,
            incoterms=incoterms,
            company_name=company,
            ship_date=ship_date,
            cargo=build_cargo(raw),
            parties=build_parties(raw),
            license_numbers=build_license_numbers(raw),
            shipment_source=self.source_id,
        )


# Self-register on import
register("fedex", FedExCarrierNetworkAdapter)
```

**Step 4: Run tests to verify they pass**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_adapter.py -v
```
Expected: all PASSED

**Step 5: Commit**

```bash
git add clearance_platform/adapters/carrier_network/fedex/adapter.py \
        tests/unit/adapters/carrier_network/test_fedex_adapter.py
git commit -m "feat: add FedExCarrierNetworkAdapter"
```

---

## Task 5: Extend `CreateShipmentRequest` with optional carrier fields

**Files:**
- Modify: `clearance-engine/backend/clearance_platform/shared/schemas/shipments.py:207-213`
- Modify: `clearance-engine/backend/app/api/schemas/shipments.py:207-213` (keep in sync)

**Step 1: Write the failing test**

Append to `tests/unit/adapters/carrier_network/test_fedex_adapter.py`:

```python
def test_manifest_record_to_create_shipment_request_fields():
    """ManifestRecord optional fields must exist on CreateShipmentRequest."""
    from clearance_platform.shared.schemas.shipments import CreateShipmentRequest
    r = CreateShipmentRequest(
        product_description="Widget",
        origin_country="CN",
        destination_country="US",
        declared_value=99.0,
        tracking_number="AWB123",
        carrier="FedEx International Priority",
        transport_mode="air",
        hs_code="6110.20.2015",
        country_of_manufacture="CN",
        incoterms="DAP",
        company_name="Farfetch",
        ship_date="2026-02-11T00:00:00",
        cargo={"pieces": 2, "weight_kg": 0.5},
        parties={"shipper": {"company": "ACME"}},
        license_numbers=None,
        carrier_id="fedex",
        initial_status="booked",
    )
    assert r.tracking_number == "AWB123"
    assert r.hs_code == "6110.20.2015"
    assert r.carrier_id == "fedex"
    assert r.initial_status == "booked"

def test_create_shipment_request_default_status_is_none():
    """UI-initiated creation leaves initial_status unset — service uses in_transit."""
    from clearance_platform.shared.schemas.shipments import CreateShipmentRequest
    r = CreateShipmentRequest(
        product_description="Widget",
        origin_country="CN",
        destination_country="US",
        declared_value=99.0,
    )
    assert r.initial_status is None
```

**Step 2: Run to verify it fails**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_adapter.py::test_manifest_record_to_create_shipment_request_fields -v
```
Expected: `ValidationError` (unexpected fields)

**Step 3: Extend the schema in both files**

In `clearance_platform/shared/schemas/shipments.py`, replace the `CreateShipmentRequest` class:

```python
class CreateShipmentRequest(BaseModel):
    """Payload for creating a new shipment.

    The four required fields are sufficient for UI-initiated creation.
    The optional carrier_* fields are populated by carrier network adapters
    to avoid a second update-fields call — all are backwards compatible.
    """

    product_description: str = Field(..., min_length=1, max_length=2000)
    origin_country: str = Field(..., min_length=2, max_length=3)
    destination_country: str = Field(..., min_length=2, max_length=3)
    declared_value: float = Field(..., gt=0, description="Declared value of goods.")

    # Carrier-supplied optional fields
    carrier_id: str | None = None
    tracking_number: str | None = None
    carrier: str | None = None
    transport_mode: str | None = None
    hs_code: str | None = None
    country_of_manufacture: str | None = None
    incoterms: str | None = None
    company_name: str | None = None
    ship_date: str | None = None
    cargo: dict | None = None
    parties: dict | None = None
    license_numbers: dict | None = None
    # When set, overrides the service default of "in_transit".
    # Carrier imports set "booked" to enter the preclearance pipeline.
    initial_status: str | None = None
    # Written to Shipment.shipment_source. NULL means simulation-generated.
    # Simulation actors query WHERE shipment_source IS NULL to exclude imports.
    shipment_source: str | None = None
```

Apply the identical change to `app/api/schemas/shipments.py`.

**Step 4: Run test to verify it passes**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_fedex_adapter.py::test_manifest_record_to_create_shipment_request_fields -v
```
Expected: PASSED

**Step 5: Run the full unit suite to confirm no regressions**

```bash
python -m pytest tests/unit/ -v --tb=short
```
Expected: all previously passing tests still PASS

**Step 6: Commit**

```bash
git add clearance_platform/shared/schemas/shipments.py \
        app/api/schemas/shipments.py \
        tests/unit/adapters/carrier_network/test_fedex_adapter.py
git commit -m "feat: extend CreateShipmentRequest with optional carrier fields"
```

---

## Task 5.5: Add `shipment_source` column to `Shipment` model + migration

**Files:**
- Modify: `clearance-engine/backend/clearance_platform/domains/shipment_lifecycle/models.py`
- Create: `clearance-engine/backend/alembic/versions/XXX_add_shipment_source.py`

**Why this matters:** `shipment_source` is indexed so simulation actors can efficiently
query `WHERE shipment_source IS NULL` (simulation-generated) vs carrier imports. No
existing code breaks — the column is nullable with no server default, so all current
rows read as NULL (= simulation-generated).

**Step 1: Add the column to the model**

In `models.py`, add after the `entity_state` field and before `__table_args__`:

```python
    # Identifies the origin of this shipment record.
    # NULL = simulation-generated (the historical default).
    # Set by carrier network adapters via CreateShipmentRequest.shipment_source.
    # Simulation actors filter WHERE shipment_source IS NULL to exclude imports.
    shipment_source: Mapped[Optional[str]] = mapped_column(
        String(100), nullable=True, index=True
    )
```

Also add `"ix_shipments_shipment_source"` to `__table_args__`:

```python
        Index("ix_shipments_shipment_source", "shipment_source"),
```

**Step 2: Generate the Alembic migration**

```bash
cd clearance-engine/backend
alembic revision --autogenerate -m "add_shipment_source_to_shipments"
```

Review the generated file — it should contain exactly:

```python
def upgrade() -> None:
    op.add_column(
        "shipments",
        sa.Column("shipment_source", sa.String(100), nullable=True),
        schema="shipment",
    )
    op.create_index(
        "ix_shipments_shipment_source",
        "shipments",
        ["shipment_source"],
        schema="shipment",
    )

def downgrade() -> None:
    op.drop_index("ix_shipments_shipment_source", table_name="shipments", schema="shipment")
    op.drop_column("shipments", "shipment_source", schema="shipment")
```

**Step 3: Apply the migration**

```bash
alembic upgrade head
```

Expected: `Running upgrade ... -> <new_rev>, add_shipment_source_to_shipments`

**Step 4: Verify**

```bash
python -c "
from clearance_platform.domains.shipment_lifecycle.models import Shipment
cols = [c.name for c in Shipment.__table__.columns]
assert 'shipment_source' in cols, f'Missing! Got: {cols}'
print('OK:', cols)
"
```

**Step 5: Commit**

```bash
git add clearance_platform/domains/shipment_lifecycle/models.py \
        alembic/versions/
git commit -m "feat: add shipment_source column to shipments table"
```

---

## Task 6: Update `create_shipment` service to apply carrier fields

**Files:**
- Modify: `clearance-engine/backend/clearance_platform/domains/shipment_lifecycle/service.py:726-895`

**What to change:** When carrier-supplied optional fields are present on `payload`, apply them directly instead of using the auto-derived defaults. Specifically:
- `tracking_number` → use instead of `_tracking_number("CL")`
- `carrier` → use instead of `"Pending Assignment"`
- `company_name` → use instead of `OUR_COMPANY`
- `initial_status` → use instead of `"in_transit"`. Carrier imports pass `"booked"` so shipments enter the preclearance pipeline. UI creation leaves this `None` → service defaults to `"in_transit"`.
- `shipment_source` → write directly. NULL for simulation/UI-created (existing behaviour unchanged); carrier imports pass their `config.source_id` (e.g. `"fedex_carrier_network"`).
- `transport_mode`, `country_of_manufacture`, `incoterms` → write to references JSONB
- `hs_code` → if provided, skip E1 and use directly as `submitted_hs_code` (E1 can run later via re-analysis).
- `cargo`, `parties`, `license_numbers` → write to `references` JSONB
- `ship_date` → use as the initial event timestamp if provided

**Step 1: Write the failing test**

```python
# tests/unit/adapters/carrier_network/test_service_carrier_fields.py
"""Verify create_shipment uses carrier-supplied fields when present."""
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from clearance_platform.shared.schemas.shipments import CreateShipmentRequest


@pytest.mark.asyncio
async def test_create_shipment_uses_carrier_tracking_number():
    """When tracking_number is set, it is used verbatim."""
    from clearance_platform.domains.shipment_lifecycle.service import ShipmentLifecycleService

    payload = CreateShipmentRequest(
        product_description="FLEXIBLE CABLE",
        origin_country="TW",
        destination_country="US",
        declared_value=1.0,
        tracking_number="165023616658",
        carrier="FedEx International Priority",
        company_name="Farfetch",
        transport_mode="air",
        initial_status="booked",
    )

    mock_session = AsyncMock()
    mock_session.add = MagicMock()
    mock_session.commit = AsyncMock()
    mock_session.refresh = AsyncMock()

    captured = {}

    def _capture_add(obj):
        captured["shipment"] = obj

    mock_session.add.side_effect = _capture_add

    svc = ShipmentLifecycleService.__new__(ShipmentLifecycleService)

    with patch(
        "clearance_platform.domains.shipment_lifecycle.service.classify",
        new=AsyncMock(return_value=MagicMock(hs_code="6110.20.2015")),
    ), patch(
        "clearance_platform.domains.shipment_lifecycle.service.calculate_tariff",
        new=AsyncMock(return_value=MagicMock(total_duty=5.0)),
    ):
        with pytest.raises(Exception):
            # Will fail at session.refresh (mock) — that's fine, we just need
            # to inspect the Shipment object that was passed to session.add
            await svc.create_shipment(mock_session, payload)

    shipment = captured.get("shipment")
    assert shipment is not None
    assert shipment.tracking_number == "165023616658"
    assert shipment.carrier == "FedEx International Priority"
    assert shipment.company_name == "Farfetch"
    assert shipment.transport_mode == "air"
    # Carrier import must start at booked — entry point for analysis pipeline
    assert shipment.status == "booked"
```

**Step 2: Run to verify it fails**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_service_carrier_fields.py -v
```
Expected: FAIL (tracking_number is `CL-...` not the AWB)

**Step 3: Modify `create_shipment` in service.py**

In `create_shipment`, replace:

```python
        shipment_id = uuid.uuid4()
        tracking = _tracking_number("CL")
        now = _now_iso()
```

With:

```python
        shipment_id = uuid.uuid4()
        tracking = payload.tracking_number or _tracking_number("CL")
        now = payload.ship_date or _now_iso()
```

Replace the HS code resolution block:

```python
        # Attempt real classification and tariff calculation
        hs_code = "0000.00.0000"
        predicted_duty = round(payload.declared_value * 0.05, 2)

        try:
            from clearance_platform.shared.engines.e1_classification import classify
            classification_result = await classify(...)
            if classification_result.hs_code ...:
                hs_code = classification_result.hs_code
        except Exception as exc:
            ...
```

With:

```python
        # HS code: use carrier-supplied code if present, otherwise run E1
        hs_code = "0000.00.0000"
        predicted_duty = round(payload.declared_value * 0.05, 2)

        if payload.hs_code:
            hs_code = payload.hs_code
        else:
            try:
                from clearance_platform.shared.engines.e1_classification import classify
                classification_result = await classify(
                    product_description=payload.product_description,
                    origin_country=payload.origin_country,
                    destination_country=payload.destination_country,
                )
                if classification_result.hs_code and classification_result.hs_code != "0000.00.0000":
                    hs_code = classification_result.hs_code
            except Exception as exc:
                logger.warning("shipment_create_classify_error", error=str(exc))
```

Replace the `Shipment(...)` constructor call to use carrier-supplied fields:

```python
        # Build references from carrier-supplied fields
        carrier_refs: dict = {}
        if payload.country_of_manufacture:
            carrier_refs["country_of_manufacture"] = payload.country_of_manufacture
        if payload.incoterms:
            carrier_refs["incoterms"] = payload.incoterms
        if payload.parties:
            carrier_refs["parties"] = payload.parties
        if payload.license_numbers:
            carrier_refs["license_numbers"] = payload.license_numbers
        if payload.carrier_id:
            carrier_refs["carrier_id"] = payload.carrier_id

        obj = Shipment(
            id=shipment_id,
            product=payload.product_description,
            product_id=None,
            origin=payload.origin_country,
            destination=payload.destination_country,
            status=payload.initial_status or "in_transit",
            carrier=payload.carrier or "Pending Assignment",
            tracking_number=tracking,
            transport_mode=payload.transport_mode,
            declared_value=payload.declared_value,
            events=[initial_event],
            waypoints=waypoints,
            codes=codes,
            financials=financials,
            cargo_detail=payload.cargo,
            references=carrier_refs if carrier_refs else None,
            analysis=None,
            company_name=payload.company_name or OUR_COMPANY,
            shipment_source=payload.shipment_source,  # None for sim/UI, set for carrier imports
        )
```

**Step 4: Run test to verify it passes**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_service_carrier_fields.py -v
```
Expected: PASSED

**Step 5: Run full unit suite**

```bash
python -m pytest tests/unit/ -v --tb=short
```
Expected: all PASSED

**Step 6: Commit**

```bash
git add clearance_platform/domains/shipment_lifecycle/service.py \
        tests/unit/adapters/carrier_network/test_service_carrier_fields.py
git commit -m "feat: create_shipment applies carrier-supplied fields directly"
```

---

## Task 7: Async ingestion driver

**Files:**
- Create: `clearance-engine/backend/clearance_platform/adapters/carrier_network/importer.py`
- Create: `clearance-engine/backend/tests/unit/adapters/carrier_network/test_importer.py`

**Step 1: Write the failing tests**

```python
# tests/unit/adapters/carrier_network/test_importer.py
import asyncio
import pytest
import httpx
from clearance_platform.adapters.carrier_network.base import CarrierNetworkAdapter, ManifestRecord
from clearance_platform.adapters.carrier_network.importer import import_records


def _make_record(awb: str, value: float = 10.0) -> ManifestRecord:
    return ManifestRecord(
        product_description="Widget",
        origin_country="CN",
        destination_country="US",
        declared_value=value,
        carrier_id="fedex",
        tracking_number=awb,
        carrier_reference=awb,
    )


class _StubAdapter(CarrierNetworkAdapter):
    carrier_id = "stub"

    def __init__(self, fail_on: list[str] | None = None):
        self.fail_on = fail_on or []

    def translate(self, raw: dict) -> ManifestRecord:
        awb = raw["awb"]
        if awb in self.fail_on:
            raise ValueError(f"Bad record: {awb}")
        return _make_record(awb, raw.get("value", 10.0))


@pytest.mark.asyncio
async def test_successful_import():
    transport = httpx.MockTransport(
        lambda req: httpx.Response(201, json={"id": "abc-123", "tracking_number": "AWB001"})
    )
    raw = [{"awb": "AWB001"}, {"awb": "AWB002"}]
    result = await import_records(
        adapter=_StubAdapter(),
        raw_records=raw,
        api_base_url="http://test",
        transport=transport,
    )
    assert result["success"] == 2
    assert result["errors"] == 0
    assert result["skipped_duplicates"] == 0


@pytest.mark.asyncio
async def test_duplicate_skipped():
    transport = httpx.MockTransport(
        lambda req: httpx.Response(201, json={"id": "abc"})
    )
    raw = [{"awb": "AWB001"}, {"awb": "AWB002"}]
    result = await import_records(
        adapter=_StubAdapter(),
        raw_records=raw,
        api_base_url="http://test",
        transport=transport,
        existing_references={"AWB001"},
    )
    assert result["success"] == 1
    assert result["skipped_duplicates"] == 1


@pytest.mark.asyncio
async def test_translation_error_counted():
    transport = httpx.MockTransport(
        lambda req: httpx.Response(201, json={"id": "abc"})
    )
    raw = [{"awb": "BAD"}, {"awb": "AWB001"}]
    result = await import_records(
        adapter=_StubAdapter(fail_on=["BAD"]),
        raw_records=raw,
        api_base_url="http://test",
        transport=transport,
    )
    assert result["success"] == 1
    assert result["errors"] == 1


@pytest.mark.asyncio
async def test_api_error_counted():
    def _handler(req: httpx.Request) -> httpx.Response:
        if b"AWB002" in req.content:
            return httpx.Response(500, json={"detail": "server error"})
        return httpx.Response(201, json={"id": "abc"})

    transport = httpx.MockTransport(_handler)
    raw = [{"awb": "AWB001"}, {"awb": "AWB002"}]
    result = await import_records(
        adapter=_StubAdapter(),
        raw_records=raw,
        api_base_url="http://test",
        transport=transport,
    )
    assert result["success"] == 1
    assert result["errors"] == 1
```

**Step 2: Run to verify they fail**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_importer.py -v
```
Expected: `ImportError`

**Step 3: Create the importer**

```python
# clearance_platform/adapters/carrier_network/importer.py
"""Carrier-agnostic async ingestion driver.

Takes any CarrierNetworkAdapter, translates raw records to ManifestRecord,
and posts each to POST /api/shipments concurrently.
"""
from __future__ import annotations

import asyncio
import structlog
from typing import Any

import httpx

from clearance_platform.adapters.carrier_network.base import CarrierNetworkAdapter, ManifestRecord

logger = structlog.get_logger(__name__)


async def import_records(
    adapter: CarrierNetworkAdapter,
    raw_records: list[dict],
    api_base_url: str,
    *,
    concurrency: int = 20,
    existing_references: set[str] | None = None,
    transport: Any = None,  # injectable for testing (httpx.MockTransport)
) -> dict:
    """Translate and import raw carrier records into the platform.

    Args:
        adapter: CarrierNetworkAdapter to use for translation.
        raw_records: Raw dicts from carrier data source.
        api_base_url: Base URL of the platform API.
        concurrency: Maximum concurrent HTTP requests.
        existing_references: Set of carrier_references already in platform (dedup).
        transport: Optional httpx transport override (for testing).

    Returns:
        Summary dict with success, skipped_duplicates, errors counts.
    """
    existing = existing_references or set()
    errors: list[dict] = []
    skipped = 0

    # Translate all records upfront
    translated: list[ManifestRecord] = []
    for raw in raw_records:
        try:
            record = adapter.translate(raw)
            if record.carrier_reference in existing:
                skipped += 1
                continue
            translated.append(record)
        except Exception as exc:
            errors.append({"error": str(exc), "raw": str(raw)[:120]})

    success = 0
    semaphore = asyncio.Semaphore(concurrency)

    client_kwargs: dict[str, Any] = {"base_url": api_base_url, "timeout": 30.0}
    if transport is not None:
        client_kwargs["transport"] = transport

    async with httpx.AsyncClient(**client_kwargs) as client:
        async def _post_one(record: ManifestRecord) -> bool:
            async with semaphore:
                payload = _to_payload(record)
                try:
                    resp = await client.post("/api/shipments", json=payload)
                    resp.raise_for_status()
                    logger.debug(
                        "carrier_import_success",
                        tracking=record.tracking_number,
                        carrier_id=record.carrier_id,
                    )
                    return True
                except Exception as exc:
                    errors.append({"error": str(exc), "tracking": record.tracking_number})
                    logger.warning(
                        "carrier_import_failed",
                        tracking=record.tracking_number,
                        error=str(exc),
                    )
                    return False

        results = await asyncio.gather(*[_post_one(r) for r in translated])
        success = sum(1 for r in results if r)

    summary = {
        "total": len(raw_records),
        "success": success,
        "skipped_duplicates": skipped,
        "errors": len(errors),
        "error_details": errors[:20],
    }
    logger.info("carrier_import_complete", **{k: v for k, v in summary.items() if k != "error_details"})
    return summary


def _to_payload(record: ManifestRecord) -> dict:
    payload: dict = {
        "product_description": record.product_description,
        "origin_country": record.origin_country,
        "destination_country": record.destination_country,
        "declared_value": record.declared_value,
    }
    for f in (
        "carrier_id", "tracking_number", "carrier", "transport_mode",
        "hs_code", "country_of_manufacture", "incoterms", "company_name",
        "ship_date", "cargo", "parties", "license_numbers",
    ):
        val = getattr(record, f, None)
        if val is not None:
            payload[f] = val
    return payload
```

**Step 4: Run tests to verify they pass**

```bash
python -m pytest tests/unit/adapters/carrier_network/test_importer.py -v
```
Expected: all PASSED

**Step 5: Run full unit suite**

```bash
python -m pytest tests/unit/ -v --tb=short
```
Expected: all PASSED

**Step 6: Commit**

```bash
git add clearance_platform/adapters/carrier_network/importer.py \
        tests/unit/adapters/carrier_network/test_importer.py
git commit -m "feat: add async carrier network ingestion driver"
```

---

## Task 8: CLI import script

**Files:**
- Create: `clearance-engine/backend/scripts/import_fedex_manifest.py`

No tests for this task — it's a thin CLI wrapper over the already-tested importer.

**Step 1: Create the script**

```python
#!/usr/bin/env python3
"""Import FedEx manifest data from Excel files into the clearance platform.

Usage:
    cd clearance-engine/backend
    python scripts/import_fedex_manifest.py \\
        --url http://localhost:4000 \\
        --file "../../data/CIE POC Customers.xlsx" \\
        --file "../../data/CIP POC 50K Under 800.xlsx" \\
        [--concurrency 20] \\
        [--dry-run]

The script loads all sheets from each Excel file, tags rows with their
sheet name (used as company_name fallback for CIE customer sheets),
then drives the FedEx carrier network adapter and async importer.
"""
from __future__ import annotations

import argparse
import asyncio
import sys
from pathlib import Path

# Allow running from the scripts/ directory
sys.path.insert(0, str(Path(__file__).parent.parent))

import pandas as pd

from clearance_platform.adapters.carrier_network import build_carrier_network_adapter
from clearance_platform.adapters.carrier_network.importer import import_records


def load_excel(path: Path) -> list[dict]:
    """Load all sheets from an Excel file. Tags each row with _sheet_name."""
    print(f"  Reading {path.name}...", flush=True)
    all_sheets = pd.read_excel(path, sheet_name=None)
    records: list[dict] = []
    for sheet_name, df in all_sheets.items():
        df["_sheet_name"] = sheet_name.strip()
        records.extend(df.to_dict(orient="records"))
    return records


async def main(args: argparse.Namespace) -> None:
    adapter = build_carrier_network_adapter("fedex")

    all_records: list[dict] = []
    for file_path in args.file:
        path = Path(file_path)
        if not path.exists():
            print(f"ERROR: File not found: {path}", file=sys.stderr)
            sys.exit(1)
        records = load_excel(path)
        print(f"  {len(records):>10,} records from {path.name}")
        all_records.extend(records)

    print(f"\n  Total records to process: {len(all_records):,}")

    if args.dry_run:
        ok, bad = 0, 0
        for raw in all_records:
            try:
                adapter.translate(raw)
                ok += 1
            except Exception as e:
                bad += 1
                if bad <= 10:
                    print(f"  [translate error] {e}")
        print(f"\nDry-run complete: {ok:,} translatable, {bad:,} errors")
        return

    print(f"\nImporting → {args.url}  (concurrency={args.concurrency})")
    result = await import_records(
        adapter=adapter,
        raw_records=all_records,
        api_base_url=args.url,
        concurrency=args.concurrency,
    )

    print(f"\n{'─'*40}")
    print(f"  Total processed : {result['total']:>10,}")
    print(f"  Imported        : {result['success']:>10,}")
    print(f"  Skipped (dupes) : {result['skipped_duplicates']:>10,}")
    print(f"  Errors          : {result['errors']:>10,}")
    if result["error_details"]:
        print("\n  First errors:")
        for e in result["error_details"][:5]:
            print(f"    {e}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Import FedEx manifest data into clearance platform")
    parser.add_argument("--url", default="http://localhost:4000", help="Platform API base URL")
    parser.add_argument("--file", action="append", required=True, metavar="XLSX",
                        help="Excel file path (repeat for multiple files)")
    parser.add_argument("--concurrency", type=int, default=20,
                        help="Max concurrent API calls (default: 20)")
    parser.add_argument("--dry-run", action="store_true",
                        help="Translate records only, do not call the API")
    asyncio.run(main(parser.parse_args()))
```

**Step 2: Verify dry-run works against real data**

```bash
cd clearance-engine/backend
python scripts/import_fedex_manifest.py \
    --file "../../data/CIE POC Customers.xlsx" \
    --file "../../data/CIP POC 50K Under 800.xlsx" \
    --dry-run
```

Expected output:
```
  Reading CIE POC Customers.xlsx...
      61,652 records from CIE POC Customers.xlsx
  Reading CIP POC 50K Under 800.xlsx...
     150,003 records from CIP POC 50K Under 800.xlsx

  Total records to process: 211,655
Dry-run complete: ~211,000+ translatable, <500 errors
```

**Step 3: Commit**

```bash
git add scripts/import_fedex_manifest.py
git commit -m "feat: add FedEx manifest import CLI script"
```

---

## Task 9: Update `adapters/__init__.py` docstring

**Files:**
- Modify: `clearance-engine/backend/clearance_platform/adapters/__init__.py`

**Step 1: Update the docstring**

```python
"""Carrier-specific integration adapters.

Each sub-package integrates with an external carrier or data partner.
Adapters translate between external protocols and platform domain events,
following the "adapter, not modification" principle described in
``docs/fedex-integration-architecture.md``.

Two adapter families exist — they are independent and serve different purposes:

**Data partnership adapters** (``adapters/fedex/``)
    Integrations with data intelligence providers. The FedEx Dataworks HS API
    adapter provides classification enrichment for the E1 engine.

**Carrier network adapters** (``adapters/carrier_network/``)
    Pluggable logistics carrier integrations. Translate carrier manifest formats
    into platform ``ManifestRecord`` objects for shipment ingestion.
    Carriers: fedex (implemented), ups/dhl (future).

See ``docs/carrier-network-adapter-design.md`` for the carrier network design.
"""
```

**Step 2: Commit**

```bash
git add clearance_platform/adapters/__init__.py
git commit -m "docs: clarify two adapter families in package docstring"
```

---

## Task 10: Final verification

**Step 1: Run the full backend test suite**

```bash
cd clearance-engine/backend
python -m pytest tests/ -v --tb=short -q
```
Expected: all previously passing tests still PASS, new carrier network tests PASS

**Step 2: Dry-run the import script**

```bash
python scripts/import_fedex_manifest.py \
    --file "../../data/CIE POC Customers.xlsx" \
    --file "../../data/CIP POC 50K Under 800.xlsx" \
    --dry-run
```
Expected: translation success rate >98%

**Step 3: Live import against local dev stack**

Start the dev stack first (`make dev` or `docker compose up`), then:

```bash
python scripts/import_fedex_manifest.py \
    --url http://localhost:4000 \
    --file "../../data/CIE POC Customers.xlsx" \
    --file "../../data/CIP POC 50K Under 800.xlsx" \
    --concurrency 20
```

Expected: shipments appear in broker queue at `http://localhost:4001`

**Step 4: Final commit**

```bash
git add -A
git commit -m "feat: carrier network adapter — FedEx implementation complete

Pluggable CarrierNetworkAdapter ABC + ManifestRecord, FedEx carrier
network adapter with full field mapping, async ingestion driver, and
CLI script. Imports 211K FedEx POC shipments through E1→E2→preclearance
→broker queue pipeline in a single API call per shipment.

Separate from adapters/fedex/ (FedEx Dataworks HS API — data partnership).
"
```
