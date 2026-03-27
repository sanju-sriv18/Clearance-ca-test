# Shipment Lifecycle, Tracking Events, and Logistics Operations

> Clearance Intelligence Engine -- Operational Knowledge Base

> Last Updated: February 2026

---

## 1. Ocean Freight Shipment Lifecycle

### 1.1 Booking Confirmation
The ocean freight process begins when a shipper or freight forwarder requests a quotation from a carrier or NVOCC (Non-Vessel Operating Common Carrier). Critical details are confirmed: port of loading (POL), port of discharge (POD), cargo description (commodity, weight, dimensions), container type (20' / 40' / 40'HC / reefer), Incoterms (FOB, CIF, DDP, etc.), and target ship date. Upon acceptance, a **booking confirmation** is issued with a unique booking number. The carrier issues a **Shipping Order (SO)** which authorizes the empty container pickup. For FCL (Full Container Load), the shipper gets a dedicated container. For LCL (Less than Container Load), cargo is consolidated at a Container Freight Station (CFS) with other shippers' goods.

### 1.2 Container Pickup and Stuffing at Origin Warehouse
The empty container is picked up from a container yard or depot and delivered to the shipper's warehouse or factory. Cargo is loaded ("stuffed") into the container. A packing list is created documenting contents, weights, and piece counts. For LCL shipments, cargo is delivered loose to the CFS, where a consolidator loads it into shared containers. The container is sealed with a uniquely numbered security seal. Verified Gross Mass (VGM) must be declared per SOLAS regulations before the container can be loaded onto a vessel.

### 1.3 Drayage to Port of Loading
The loaded container is transported via truck (drayage) from the warehouse/CFS to the port terminal or inland intermodal facility. The container passes through the terminal gate ("gate-in" event), where the seal number, container number, and condition are verified. The container is placed in the terminal yard to await vessel loading. For intermodal moves, the container may travel by rail to a port-adjacent rail ramp before drayage to the terminal.

### 1.4 Export Customs Clearance at Origin
Prior to vessel departure, the goods must clear export customs at the origin country. An **Export Declaration** (in the US, this is the Electronic Export Information / EEI filed via AES) is submitted to the origin country's customs authority. Required documents include the commercial invoice, packing list, certificate of origin, and any product-specific certificates (phytosanitary, fumigation, etc.). Export licenses may be required for controlled goods. Once cleared, the goods receive export release authorization.

### 1.5 Documentation Preparation
Key documents prepared at this stage include:
- **Bill of Lading (B/L)** -- the contract of carriage and document of title. Can be a Master B/L (issued by the carrier) or a House B/L (issued by the forwarder). Types: Original (negotiable), Sea Waybill (non-negotiable), Express Release.
- **Commercial Invoice** -- details the transaction value, seller/buyer, payment terms.
- **Packing List** -- specifies contents, weights, dimensions per package.
- **Certificate of Origin** -- certifies where goods were manufactured; may be required for preferential tariff treatment under FTAs.
- **Shipping Instructions (SI)** -- submitted to the carrier before the SI cut-off deadline (typically 24-72 hours before vessel ETD) to avoid late fees.

### 1.6 Vessel Loading and Departure (ETD)
The container is loaded onto the vessel at the port terminal using ship-to-shore (STS) gantry cranes. Loading follows a pre-planned **bay plan** / stowage plan that accounts for container weight, hazmat class, destination port sequence, and reefer power requirements. The carrier issues the **on-board Bill of Lading** confirming the container is loaded. The vessel departs the port (ETD -- Estimated Time of Departure becomes ATD -- Actual Time of Departure). Cargo cut-off times typically range from 24-72 hours before vessel departure.

### 1.7 Ocean Transit (Tracking via AIS and Carrier APIs)
During ocean transit, the vessel is tracked via:
- **AIS (Automatic Identification System)**: A VHF radio-based transponder system required by IMO/SOLAS on all vessels over 300 GT. Ships broadcast identification, position, course, speed, and heading. AIS data is captured by terrestrial antenna networks and satellite receivers. Platforms like MarineTraffic, VesselFinder, and VesselTracker provide real-time vessel position maps. As of 2021, over 1.6 million ships are equipped with AIS.
- **Carrier APIs and Portals**: Carriers provide container-level tracking through their websites and APIs. Events include loaded, departed, in transit, arrived, discharged, etc. The **DCSA Track & Trace** standard (see Section 4) provides a unified event structure across the nine largest carriers.
- **Predictive ETA Systems**: Platforms like Flexport, project44, DHL Smart ETA, and Portcast use machine learning models trained on historical and real-time data to predict arrival times. Flexport's model achieved 75% on-time performance accuracy vs. 53% carrier schedule reliability on the Transpacific Eastbound lane.

Typical port-to-port transit times (2025):
| Trade Lane | Typical Days |
|---|---|
| Asia to U.S. West Coast | 15-25 |
| Asia to U.S. East Coast | 25-35 |
| Asia to Northern Europe | 28-40 |
| Transatlantic (US/Europe) | 7-12 |
| Intra-Asia | 3-14 |

### 1.8 Transshipment at Intermediate Ports
Many shipping routes require transshipment, where the container is discharged from one vessel and loaded onto another at an intermediate hub port. Major transshipment hubs include Singapore, Colombo, Tanjung Pelepas, Algeciras, and Panama. Transshipment adds 2-7 days to transit time. Risk of delays increases at each transshipment point due to vessel schedule misalignment, terminal congestion, or missed connections. The DCSA standard tracks transshipment events with facility type code "POTE" at the transshipment port.

### 1.9 Arrival at Destination Port (ETA to ATA)
The carrier publishes an ETA (Estimated Time of Arrival) that may be updated multiple times during transit. Upon arrival at the port of discharge, the vessel anchors or proceeds directly to berth. ATA (Actual Time of Arrival) is recorded when the vessel is berthed with first line ashore. Pre-arrival filing requirements must be met (see Section 1.10). The vessel discharge operation begins (see Section 3.1).

### 1.10 Manifest Filing and Customs Processing
In the United States, the **Importer Security Filing (ISF / "10+2")** must be filed at least 24 hours before the vessel departs the foreign port. The ISF includes 10 data elements from the importer and 2 from the carrier. Failure to file on time incurs penalties of $5,000-$10,000 per violation. The carrier files the **Vessel Manifest** with CBP via the Automated Manifest System (AMS) at least 24 hours before arrival at a U.S. port. CBP's Automated Targeting System (ATS) performs risk assessment on the manifest data and may issue "do not load" orders, exam holds, or release instructions.

### 1.11 Container Unloading and Terminal Storage
The vessel is discharged by STS gantry cranes. Containers are placed in the terminal yard, organized by carrier, size, weight, and destination. **Free time** begins -- the grace period (typically 3-7 days at the terminal, varying by terminal operator) during which no storage charges apply. After free time expires, **demurrage** charges accrue for every day the container remains in the terminal. Demurrage rates are tiered and escalate over time, typically starting at $75-150/day and increasing to $300+/day after extended periods. Per the FMC and the Ocean Shipping Reform Act (OSRA) of 2022, demurrage charges must be reasonable and related to actual cargo movement.

### 1.12 Import Customs Clearance
The customs broker files a **Customs Entry** (CBP Form 3461) within 15 calendar days of arrival. CBP's ACE (Automated Commercial Environment) system processes the entry and returns one of these statuses:
- **Released**: Cleared for pickup. All holds removed.
- **Admissible**: Bill match occurred, no holds, but variable release window not yet reached. Automatically converts to Released.
- **Hold (Intensive Exam)**: Physical examination required at a CES facility.
- **Document Review Hold**: Document submission required before release.
- **Manifest Hold**: Hold on the associated bill of lading.

An **Entry Summary** (CBP Form 7501) must be filed within 10 working days of release, with estimated duties deposited. **Liquidation** -- CBP's final duty calculation -- occurs within 314 days (up to 4 years with extensions). Liquidation outcomes: No Change, Increase (bill issued), or Decrease (refund issued). Liquidations process weekly on Fridays. Importers have 180 days post-liquidation to file a **Protest** (19 USC 1514).

### 1.13 Container Release and Delivery Order
Once CBP releases the shipment, the carrier or terminal issues a **Delivery Order (D/O)**, authorizing the trucking company to pick up the container. The original Bill of Lading (or bank release for L/C transactions) must be surrendered to obtain the D/O. The consignee's trucker presents the D/O and a valid interchange agreement at the terminal. The container goes through the terminal gate ("gate-out" event).

### 1.14 Drayage to Final Destination and Deconsolidation
The released container is transported by truck to the consignee's warehouse or an off-dock facility. For FCL, the container is delivered directly and stripped at the consignee's facility. For LCL, the container goes to a CFS for deconsolidation -- individual shipments are separated and made available for pickup or local delivery. The empty container must be returned to a designated depot within the carrier's **free time for detention** (typically 4-7 days). After free time expires, **detention** charges accrue (typically $125-175/day per container).

---

## 2. Air Freight Shipment Lifecycle

### 2.1 Booking and Acceptance
The process starts with a freight forwarder or shipper requesting an air freight booking. Key parameters: origin/destination airports, commodity, weight, dimensions, and any special handling requirements (dangerous goods, temperature control, live animals). Cargo is classified as either general cargo or special cargo. The forwarder books space on a specific flight or as a general allocation. An **Air Waybill (AWB)** is prepared -- a non-negotiable transport document that serves as receipt, contract of carriage, and customs declaration document. AWBs can be Master (MAWB, issued by the airline) or House (HAWB, issued by the forwarder).

### 2.2 Airport of Origin Processing
Cargo is delivered to the freight forwarder's warehouse near the origin airport, typically under customs bond. The forwarder performs:
- Acceptance check: verifying cargo matches documentation (weight, piece count, marking).
- Security screening: all air cargo undergoes mandatory X-ray scanning and/or physical inspection per ICAO/TSA regulations. High-risk shipments receive enhanced screening.
- Build-up: cargo is consolidated onto **ULDs (Unit Load Devices)** -- standardized pallets or containers designed to fit aircraft holds. Types include LD3 (lower deck), LD7 (lower deck wide-body), and PMC (pallet with net).
- Labeling: AWB labels, handling labels (e.g., "This Way Up," "Fragile"), and security screening confirmation labels.

### 2.3 Export Clearance
Export customs clearance is completed before the flight. In the US, the EEI (Electronic Export Information) is filed via AES for shipments valued over $2,500 or requiring an export license. The customs authority reviews and releases the shipment for export.

### 2.4 Flight Departure
Cargo is loaded into the aircraft: in the belly hold (lower deck) of passenger aircraft, or in both main deck and lower deck of dedicated freighter aircraft. The flight departs the origin airport.

### 2.5 Transit Through Hub Airports
Many air freight routes involve transit through major cargo hubs (e.g., Dubai, Hong Kong, Amsterdam, Memphis, Louisville, Anchorage). At the hub, cargo may be:
- Transferred between aircraft (direct transit, typically 4-24 hours)
- Broken down and reconsolidated for different destinations
- Held in a bonded warehouse pending the next connecting flight

### 2.6 Arrival at Destination Airport
The aircraft arrives at the destination gateway airport. Cargo is unloaded and moved to the airline's cargo terminal or a ground handling agent's warehouse. ULDs are broken down and individual shipments are sorted.

### 2.7 Customs Processing
Import customs clearance for air freight is significantly faster than ocean freight. In the US, air cargo entries are typically processed within 1-3 days. Pre-clearance can allow same-day release. If customs clearance is completed before 9 AM on the day of arrival, the shipment can be loaded onto a truck for same-day delivery. Formal entry is required for shipments valued over $2,500. Section 321 de minimis entry applies for shipments valued at $800 or less (though this threshold is under regulatory review as of 2025).

### 2.8 Delivery to Consignee
The freight forwarder picks up the cleared cargo from the airline terminal. Last-mile delivery is performed by truck to the consignee's address. Proof of delivery (POD) is obtained.

### 2.9 Key Differences from Ocean Freight
| Aspect | Ocean Freight | Air Freight |
|---|---|---|
| Transit time | 14-45 days | 1-5 days (door-to-door) |
| Cost per kg | $0.10-0.50/kg | $2.00-8.00/kg |
| Document of title | Bill of Lading (negotiable) | Air Waybill (non-negotiable) |
| Cargo unit | Container (TEU/FEU) | ULD (pallet/container) |
| Customs speed | 2-7 days typical | 1-3 days typical |
| Weight capacity | Virtually unlimited | Limited by aircraft payload |
| Security | Seal verification | Mandatory X-ray screening |
| Tracking granularity | Container-level events | Flight-level + piece-level |
| ISF requirement | Yes (24h pre-departure) | No ISF required |
| Manifest filing | 24h before arrival | 4 hours before arrival (air AMS) |

---

## 3. Port Operations and Terminal Processes

### 3.1 Vessel Discharge Operations
When a vessel berths at the terminal, STS (Ship-to-Shore) gantry cranes discharge containers according to the discharge plan. Modern terminals can discharge 25-40 containers per hour per crane. Containers are transferred to the yard by straddle carriers, reach stackers, or automated guided vehicles (AGVs). Import containers are stacked in the yard by carrier, size, and destination. The terminal generates a discharge report/tally confirming the number and condition of containers discharged.

### 3.2 Terminal Storage: Free Time, Demurrage, and Detention
**Free Time** is the carrier- or terminal-granted grace period during which containers can sit at the terminal or with the consignee without incurring charges. Typical ranges:
- Terminal free time (import): 3-7 days from discharge
- Carrier free time (detention): 4-7 days from gate-out

**Demurrage** applies when a container remains inside the terminal beyond the free time. Charges are per container per day and escalate:
- Days 1-3 over free time: ~$75-125/day
- Days 4-7: ~$150-200/day
- Days 8+: ~$200-350+/day

**Detention** applies when a container is outside the terminal beyond the allowed return time. The consignee must return the empty container to a designated depot. Typical charges: $125-175/day per container.

Per the **Federal Maritime Commission (FMC)** and OSRA 2022, demurrage and detention charges must serve an incentive function -- encouraging efficient cargo movement, not penalizing shippers for delays outside their control (e.g., terminal closures, exam holds).

### 3.3 Container Freight Station (CFS) for LCL
A **CFS** is a warehouse facility (usually bonded) where LCL cargo is consolidated or deconsolidated. At origin, multiple shippers' cargo is consolidated into shared containers. At destination, the container is delivered to the CFS where it is stripped and individual shipments are made available for pickup. CFS facilities are licensed by CBP and must maintain a custodial bond. CFS operations add 5-7 days to total LCL transit time compared to FCL.

### 3.4 Bonded Warehouse Facilities
A **CBP Bonded Warehouse** (19 USC 1555, 19 CFR 19) is a facility authorized for the storage of imported merchandise without payment of duties. Key characteristics:
- Goods can be stored for up to 5 years
- Duties are deferred until goods enter U.S. commerce
- If goods are re-exported, no duties are paid
- Goods can be manipulated (repacked, relabeled) but not manufactured
- Entry type: Warehouse Entry (type 21/22)
- Bonded warehouse classes: 1 (government-owned), 2 (private, importer-owned), 3 (public), 4 (bonded yard), 5 (bonded bins), 6 (manufacturing), 7 (smelting/refining)

### 3.5 Foreign Trade Zones (FTZ)
An **FTZ** is a designated area in or near a U.S. port of entry where goods are considered outside U.S. customs territory for tariff purposes. Authorized by the Foreign-Trade Zones Board (chaired by the Secretary of Commerce) and supervised locally by CBP.

**Admission Types (Zone Status)**:
| Status | Code | Description |
|---|---|---|
| **Privileged Foreign (PF)** | PF | Duty rate and classification are "frozen" at the time of admission. Advantageous when components have a lower duty rate than the finished product. Required for AD/CVD goods. |
| **Nonprivileged Foreign (NPF)** | NPF | Duties assessed based on the condition and classification at the time of entry into U.S. commerce (withdrawal). Advantageous when the finished product has a lower duty rate than components (inverted tariff). |
| **Zone-Restricted** | ZR | Goods restricted to the zone for export or destruction only. Cannot be entered into U.S. commerce without FTZ Board approval. |
| **Domestic** | D | U.S.-origin goods or previously duty-paid imports. Can enter/exit the zone without CBP permits. |

**Key FTZ Benefits**:
- Duty deferral until goods enter U.S. commerce
- Duty elimination on re-exports, waste, and scrap
- Inverted tariff savings (NPF status)
- Weekly entry filing (single entry for all goods withdrawn in a 7-day period)
- No time limits on storage
- AD/CVD duty deferral management

Goods are admitted on **CBP Form 214** and withdrawn through standard CBP entry or in-bond procedures.

### 3.6 Exam Stations / Centralized Examination Stations (CES)
A **CES** is a private facility authorized by CBP for conducting detailed physical cargo inspections that cannot be performed at the port terminal. CES operations:

1. **Transfer**: The container is moved from the terminal to the CES (at broker's direction).
2. **Staging**: Cargo is positioned for CBP officer inspection. Exam types include:
   - **VACIS / NII (Non-Intrusive Inspection)**: Large-scale X-ray or gamma-ray imaging. 1-3 days.
   - **Tailgate Exam**: Visual inspection of container contents without removing cargo. 1-3 days.
   - **Intensive Exam**: Full physical inspection -- cargo is unloaded, cartons opened, contents examined. 7-30 days.
3. **Release**: Once CBP clears the goods, the cargo is returned to the supply chain.

CES facilities must: maintain CBP-approved security, post a custodial bond, follow pre-approved fee schedules, renew their license every 3 years, and provide adequate staffing and equipment. Exam costs are borne by the importer and typically range from $300-$1,000+ depending on exam type and container size.

---

## 4. Tracking Events and Milestones (for Software Modeling)

### 4.1 DCSA Standard Event Codes (Industry Standard for Container Shipping)
The **Digital Container Shipping Association (DCSA)** -- founded in 2019 by MSC, Maersk, CMA CGM, Hapag-Lloyd, ONE, Evergreen, Yang Ming, HMM, and ZIM -- has established the primary industry standard for container tracking events. The DCSA Track & Trace standard defines events across three journey types:

**Equipment Event Type Codes** (container-level physical events):
| Code | Meaning |
|---|---|
| LOAD | Container loaded onto transport (vessel, rail, truck) |
| DISC | Container discharged from transport |
| GTIN | Gate In -- container enters a facility |
| GTOT | Gate Out -- container leaves a facility |
| STUF | Stuffing -- cargo loaded into container |
| STRP | Stripping -- cargo unloaded from container |
| PICK | Pick-up -- container collected |
| DROP | Drop-off -- container deposited |

**Transport Event Type Codes** (vessel/conveyance-level events):
| Code | Meaning |
|---|---|
| ARRI | Arrival -- transport arrives at location (vessel berthed with first line ashore; rail stationary at platform; truck at loading dock) |
| DEPA | Departure -- transport departs from location |

**Event Classifier Codes**:
| Code | Meaning |
|---|---|
| PLN | Planned (long-term schedule) |
| EST | Estimated (updated forecast) |
| ACT | Actual (confirmed occurrence) |

**Empty Indicator Codes**:
| Code | Meaning |
|---|---|
| EMPTY | Container is empty |
| LADEN | Container has cargo (also: FULL) |

**Facility Type Codes**:
| Code | Meaning |
|---|---|
| POTE | Port Terminal |
| DEPO | Container Depot |
| INTE | Inland Terminal |
| COYA | Container Yard |
| OFFD | Off-Dock facility |
| BOCR | Border Crossing |

**Shipment Location Type Codes**:
| Code | Meaning |
|---|---|
| POL | Port of Loading |
| POD | Port of Discharge |
| PRE | Place of Receipt (pre-carriage origin) |
| PDE | Place of Delivery (on-carriage destination) |
| TSP | Transshipment Port |

**Five Shipment Phases** in the DCSA model:
1. Pre-shipment (booking, documentation)
2. Pre-ocean (inland transport to port, export clearance)
3. Ocean (loading, transit, transshipment, discharge)
4. Post-ocean (import clearance, terminal storage)
5. Post-shipment (inland delivery, empty return)

### 4.2 SMDG Code Lists
**SMDG (Ship Message Design Group)** maintains code lists used in electronic data interchange for shipping:
- **Terminal Code List (TCL)**: 1,213+ terminals in 715 ports across 161 countries
- **Liner Code List**: Identifies shipping lines and services
- **Delay Reason Code List**: Standardized delay classification
- **PCACTIVITY Code**: Planned vessel schedule events
- **Stowage Code List**: Loading/discharge instructions
- **DG Attributes Code List**: Dangerous goods properties
- **TERMACT**: Terminal activity codes for performance reporting

SMDG codes are referenced by the DCSA Information Model for terminal codes, carrier codes, and delay reason codes.

### 4.3 UN/EDIFACT Message Types for Shipping
The UN/EDIFACT standard provides structured EDI messages for the shipping industry:
- **IFTMIN** (Instruction Message): Shipping instructions for cargo movement -- routing, handling, special conditions. ANSI X12 equivalent: EDI 304.
- **IFTSTA** (Status Report): Real-time cargo status updates -- in transit, delayed, arrived, delivered.
- **IFTMAN** (Arrival Notice): Notification of cargo arrival.
- **IFTMCS** (Instruction Contract Status): Contract/booking status.
- **COPARN** (Container Pre-Announcement): Notice of container pickup/return.
- **BAPLIE** (Bay Plan): Vessel stowage plan with container positions.
- **CUSCAR** (Customs Cargo Report): Cargo declaration to customs.
- **CUSDEC** (Customs Declaration): Import/export customs entry.

### 4.4 Container Tracking Event Sequence (Typical Import FCL)
For software modeling, a typical import FCL shipment generates these events in order:

1. `STUF` (Stuffing) -- Cargo loaded into container at origin warehouse
2. `GTIN` (Gate In) at origin terminal -- Container enters port terminal
3. `LOAD` (Loaded) on vessel at POL -- Container loaded onto vessel
4. `DEPA` (Departure) from POL -- Vessel departs origin port
5. `ARRI` (Arrival) at TSP -- Vessel arrives at transshipment port (if applicable)
6. `DISC` (Discharged) at TSP -- Container discharged for transshipment
7. `LOAD` (Loaded) on connecting vessel at TSP -- Container loaded onto next vessel
8. `DEPA` (Departure) from TSP -- Vessel departs transshipment port
9. `ARRI` (Arrival) at POD -- Vessel arrives at destination port
10. `DISC` (Discharged) at POD -- Container discharged from vessel
11. `GTOT` (Gate Out) from terminal -- Container picked up by trucker
12. `STRP` (Stripping) -- Cargo unloaded from container at destination
13. `GTIN` (Gate In) at depot -- Empty container returned to depot

### 4.5 Customs Events (US CBP ACE System)
| Event | Description |
|---|---|
| ISF Filed | Importer Security Filing submitted (24h pre-departure) |
| AMS Filed | Automated Manifest System filing by carrier |
| Entry Filed | CBP Form 3461 submitted |
| Admissible | Bill match confirmed, no holds, awaiting release window |
| Released | Entry cleared, all holds removed |
| Hold - Intensive Exam | Physical examination required |
| Hold - Document Review | Document submission required |
| Hold - Manifest | Bill-level hold |
| Exam Ordered | CBP orders container to CES |
| Exam Complete | CBP releases after examination |
| Entry Summary Filed | CBP Form 7501 with duties deposited |
| Liquidated | Final duty assessment (weekly, Fridays) |
| Deemed Liquidated | Auto-liquidated at declared amounts (if CBP misses deadline) |
| Suspension | Liquidation suspended pending investigation |
| Extension | Liquidation deadline extended |
| Protest Filed | Importer challenges liquidation (180-day window) |

### 4.6 Last-Mile Delivery Events
| Event | Description |
|---|---|
| Dispatched | Shipment assigned to delivery vehicle |
| Out for Delivery | Vehicle en route to consignee |
| Attempted Delivery | Delivery attempted but unsuccessful |
| Delivered | Proof of delivery obtained (signature, photo) |
| Returned to Sender | Shipment returned due to delivery failure |

---

## 5. Shipment Statuses for Software

### 5.1 Real-World Shipment Statuses (Complete Lifecycle)
For a clearance intelligence platform, the following status model captures the full shipment lifecycle:

**Pre-Shipment Phase:**
- `DRAFT` -- Shipment record created, not yet confirmed
- `BOOKED` -- Carrier booking confirmed
- `PICKUP_SCHEDULED` -- Container/cargo pickup arranged
- `PICKED_UP` -- Cargo collected from shipper
- `AT_ORIGIN_CFS` -- Cargo at origin consolidation facility (LCL)
- `STUFFED` -- Cargo loaded into container

**Export Phase:**
- `GATE_IN_ORIGIN` -- Container at origin port terminal
- `EXPORT_CLEARANCE_PENDING` -- Export customs filing in progress
- `EXPORT_CLEARED` -- Export customs approved
- `LOADED_ON_VESSEL` -- Container on board vessel
- `DEPARTED_ORIGIN` -- Vessel sailed from POL

**Transit Phase:**
- `IN_TRANSIT` -- Vessel at sea, en route
- `AT_TRANSSHIPMENT` -- At intermediate port for vessel change
- `TRANSSHIPPED` -- Loaded onto connecting vessel
- `IN_TRANSIT_LEG_2` -- Second ocean leg (if transshipment)

**Arrival/Import Phase:**
- `ARRIVED_DESTINATION` -- Vessel arrived at POD
- `DISCHARGED` -- Container offloaded from vessel
- `ISF_FILED` -- Importer Security Filing submitted
- `ENTRY_FILED` -- Customs entry submitted
- `CUSTOMS_HOLD` -- Entry held for exam or document review
- `EXAM_ORDERED` -- Container sent to CES for physical exam
- `EXAM_IN_PROGRESS` -- CBP examination underway
- `CUSTOMS_RELEASED` -- CBP release granted
- `AVAILABLE_FOR_PICKUP` -- Delivery order issued, container ready

**Delivery Phase:**
- `GATE_OUT_DESTINATION` -- Container departed terminal
- `IN_TRANSIT_INLAND` -- Drayage/rail to final destination
- `AT_DESTINATION_CFS` -- At deconsolidation facility (LCL)
- `DELIVERED` -- Cargo received by consignee
- `EMPTY_RETURNED` -- Container returned to depot

**Post-Clearance Phase:**
- `ENTRY_SUMMARY_FILED` -- Form 7501 filed with duties
- `LIQUIDATED` -- CBP final duty assessment complete
- `CLOSED` -- All obligations fulfilled

### 5.2 Mapping Shipment Status to Customs Clearance Status
| Shipment Status | Customs Status | CBP ACE Status |
|---|---|---|
| ARRIVED_DESTINATION | Pre-entry | -- |
| ENTRY_FILED | Pending release | Admissible |
| CUSTOMS_RELEASED | Cleared | Released |
| CUSTOMS_HOLD | Held | Hold (various types) |
| EXAM_ORDERED | Under exam | Intensive Exam Hold |
| ENTRY_SUMMARY_FILED | Duties deposited | Summary on file |
| LIQUIDATED | Final assessment | Liquidated/Deemed Liquidated |

### 5.3 Typical Transit Times by Mode
| Mode | Typical Door-to-Door | Port/Airport-to-Port/Airport |
|---|---|---|
| Ocean FCL (Asia-USWC) | 20-35 days | 15-25 days |
| Ocean FCL (Asia-USEC) | 30-45 days | 25-35 days |
| Ocean LCL | Add 5-10 days to FCL | Add CFS time both ends |
| Air Freight (standard) | 5-10 days | 1-3 days |
| Air Freight (express) | 2-5 days | 1-2 days |
| Ground (US domestic) | 1-7 days | N/A |
| Rail (intermodal, US) | 3-7 days | N/A |
| Rail (China-Europe) | 15-25 days | 10-20 days |

### 5.4 How Delays Compound
Delays in the shipment lifecycle create cascading effects:

**Port Congestion Cascades:**
- Vessel queuing at anchor: 1-7+ days waiting for berth
- Slower discharge due to yard congestion
- Longer terminal dwell times consuming free time
- Demurrage charges begin accruing
- Truck chassis shortages at congested ports
- Rail congestion from backed-up intermodal moves

**Customs Exam Cascades:**
- Exam hold delays: VACIS 1-3 days, tailgate 1-3 days, intensive 7-30 days
- Container transfer to CES: 1-2 days
- During exam, demurrage clock continues at the terminal (if container not yet gate-out) or detention clock runs at CES
- Additional CES handling fees: $300-1,000+
- Production schedules disrupted downstream

**Document Issue Cascades:**
- Missing/incorrect ISF: $5,000-10,000 penalty, potential hold
- Missing original B/L: cannot obtain delivery order, container sits
- Wrong HTS classification: hold for review, potential seizure
- Missing PGA documents (FDA, USDA, EPA, CPSC): hold until resolved
- Each document issue can add 3-14 days

**Compounding Example (Realistic Worst Case):**
- Port congestion adds 3 days at anchor
- Discharge delayed 2 days (yard full)
- CBP orders intensive exam: 2-day transfer to CES + 14-day exam
- Document issue discovered during exam: 5 more days
- Total unplanned delay: 26 days
- Demurrage/detention/CES costs: $5,000-15,000+
- Downstream: missed retail delivery window, stockout, lost sales

---

## 6. Multi-Modal and Special Handling

### 6.1 In-Bond Movements (IT, IE, T&E)
The **in-bond** process allows imported merchandise to move between U.S. ports of entry or to be exported without appraisement or duty payment, under a customs bond (CBP Form 301).

**Type 61 -- Immediate Transportation (IT):**
- Moves merchandise from one U.S. port to another U.S. port for entry
- Example: Container arrives at Port of Long Beach but entry will be filed at Chicago (inland port)
- Used heavily for intermodal rail movements from coastal ports to inland destinations
- Merchandise must arrive at destination within 30 days of arrival at origination port

**Type 62 -- Transportation and Exportation (T&E):**
- Moves merchandise through the U.S. for export at another port
- Example: Canadian goods arriving at a U.S. port, transported across the U.S. to the Mexican border for export
- Arrival must be reported within 2 days; export must be reported within 20 days of arrival at destination

**Type 63 -- Immediate Exportation (IE):**
- Goods arriving at a U.S. port that are immediately exported from that same port
- No entry into U.S. commerce
- Example: Transshipment cargo that arrives and departs from the same U.S. port

**In-Bond Status Lifecycle:**
1. `ON_FILE` -- In-bond created, goods not yet in the US
2. `ENROUTE` -- Goods entered the US, in transit
3. `ARRIVED` -- Reached destination port, arrival reported to CBP (completes Type 61)
4. `EXPORTED` -- Goods exported (completes Types 62 and 63)
5. `CONCLUDED` -- Succeeded by a new in-bond, liability transferred

All in-bond movements must be filed electronically in ACE since July 2019 (no paper CBP Form 7512). Arrivals and exports must be reported within 2 days of the event.

### 6.2 Oversize/Overweight Cargo
Cargo exceeding standard container dimensions or weight limits requires special handling:

**Standard Container Limits:**
- 20' container: ~21,770 kg (48,000 lbs) max payload
- 40' container: ~26,780 kg (59,040 lbs) max payload
- Max height for standard container: 8'6" (8.5 ft)
- Max height for high cube: 9'6" (9.5 ft)

**Oversize Options:**
- **Open Top containers**: No roof, cargo can protrude above container height
- **Flat Rack containers**: No sides or roof, for wide/tall cargo
- **Platform containers**: Flat base only, for heavy machinery
- **Breakbulk shipping**: Cargo loaded directly onto the vessel (not containerized)
- **RoRo (Roll-on/Roll-off)**: Wheeled cargo driven onto specialized vessels

**Overweight Considerations:**
- Road weight limits vary by state/jurisdiction (typically 80,000 lbs gross vehicle weight in the US)
- Overweight permits required for moves exceeding legal limits
- Route planning must account for bridge weight restrictions
- Some terminals charge overweight surcharges

### 6.3 Hazardous Materials (HAZMAT / IMDG)
The **IMDG Code** (International Maritime Dangerous Goods Code), developed by the IMO in 1965, governs the safe transport of dangerous goods by sea. Compliance is mandatory under SOLAS.

**Nine Hazard Classes:**
| Class | Description |
|---|---|
| 1 | Explosives |
| 2 | Gases (flammable, non-flammable, toxic) |
| 3 | Flammable Liquids |
| 4 | Flammable Solids, spontaneous combustion, water-reactive |
| 5 | Oxidizers and Organic Peroxides |
| 6 | Toxic and Infectious Substances |
| 7 | Radioactive Materials |
| 8 | Corrosives |
| 9 | Miscellaneous Dangerous Goods |

**Identification Requirements:**
- Four-digit UN Number for each hazardous material
- Proper Shipping Name (PSN) from the IMDG Code (trade names not accepted)
- Packing Group (PG): I (high danger), II (medium), III (low)
- Diamond-shaped hazard class labels

**Documentation:**
- Dangerous Goods Declaration (DGD)
- Multimodal Dangerous Goods Form (for intermodal moves)
- Container/Vehicle Packing Certificate
- Emergency Response Plan (EmS -- Emergency Schedule)

**Stowage and Segregation:**
- IMDG specifies which classes can/cannot be stowed together
- Segregation categories: "away from," "separated from," "separated by a complete compartment"
- Height and stacking restrictions for hazmat containers on vessel
- Reefer containers carrying flammable gases require explosion-proof electrical fittings

**U.S. Regulatory Framework:**
- U.S. DOT/PHMSA Hazardous Materials Regulations (HMR, 49 CFR 100-185) apply to domestic transport
- IMDG Code applies to international vessel transport
- A shipper may comply with IMDG even for domestic transport per 49 CFR 171.22 and 171.25

### 6.4 Temperature-Controlled Shipments (Reefer Containers)
**Reefer containers** are refrigerated shipping containers with built-in cooling/heating units that maintain a set temperature range.

**Specifications:**
- Standard sizes: 20' and 40' (40' most common)
- Temperature range: typically -30C to +30C (-22F to +86F)
- Power supply: 440V three-phase at terminals; diesel genset during road transport
- Atmosphere control: some units offer Controlled Atmosphere (CA) or Modified Atmosphere (MA) for perishables

**Common Commodities:**
- Perishable foods (meat, seafood, produce, dairy)
- Pharmaceuticals and vaccines (cold chain)
- Chemicals requiring temperature stability
- Chocolate, wine, and other temperature-sensitive goods

**Operational Requirements:**
- Pre-trip inspection (PTI) before loading
- Continuous power connection at terminals (reefer plugs)
- Monitoring: temperature data loggers, remote monitoring via IoT sensors
- Ventilation settings for respiring cargo (fruits, vegetables)
- Reefer containers incur surcharges: $500-3,000+ above dry container rates

**Risks:**
- Power failure during transit (genset malfunction, plug disconnection)
- Temperature excursions from incorrect settings
- Pre-cooling failures
- Condensation and moisture damage

### 6.5 Project Cargo
**Project cargo** refers to large, heavy, high-value, or complex shipments typically associated with industrial, infrastructure, or energy projects. Examples: turbines, generators, refinery modules, bridges, mining equipment.

**Characteristics:**
- Typically exceeds standard container dimensions and weight
- Requires custom engineering for lifting, transport, and securing
- Often involves multiple transport modes (vessel, barge, rail, road)
- Requires detailed route surveys for overland segments
- Specialized vessels: heavy-lift ships, semi-submersible vessels, multipurpose vessels
- Project-specific insurance and risk assessment
- Long planning cycles (months to years)
- Permits required: oversize/overweight road permits, crane permits, police escorts

**Shipping Methods:**
- **Breakbulk**: Individual pieces loaded by crane onto conventional cargo vessels
- **Heavy-lift vessels**: Ships with onboard cranes rated 100-3,000+ tonnes
- **Semi-submersible**: Vessel submerges to float very large structures onto the deck
- **RoRo**: Wheeled or tracked equipment driven onto the vessel
- **Barge**: Used for inland waterway transport or coastal feeder moves

---

## Key Sources

- [Maersk - Ocean Freight Transit Times Guide](https://www.maersk.com/logistics-explained/transportation-and-freight/2023/09/27/sea-freight-guide)
- [DHL - Air Cargo Shipping Process](https://www.dhl.com/us-en/home/global-forwarding/freight-forwarding-education-center/air-cargo-shipping-process.html)
- [IATA - Air Cargo Handling](https://www.iata.org/en/publications/newsletters/iata-knowledge-hub/what-to-know-about-air-cargo-handling/)
- [DCSA - Track & Trace Standard](https://dcsa.org/standards/track-and-trace)
- [Vizion API - DCSA Event Code Types](https://docs.vizionapi.com/docs/dcsa-event-code-types)
- [Vizion API - DCSA Milestones](https://docs.vizionapi.com/docs/dcsa-milestones)
- [SMDG - Code Lists](https://smdg.org/working-groups/codes/)
- [U.S. Trade.gov - About FTZs](https://www.trade.gov/about-ftzs)
- [CBP - Foreign Trade Zones](https://www.cbp.gov/border-security/ports-entry/cargo-security/cargo-control/foreign-trade-zones/about)
- [CBP - In-Bond Regulatory Changes FAQ](https://www.cbp.gov/border-security/ports-entry/cargo-control/bond/bond-regulatory-changes-faqs)
- [19 CFR 18.1 - In-Bond Rules](https://www.law.cornell.edu/cfr/text/19/18.1)
- [CBP - Liquidation in ACE](https://www.cbp.gov/trade/automated/news/liquidation)
- [CBP - Entry Summary Processes](https://www.cbp.gov/trade/programs-administration/entry-summary)
- [GHY - US Customs Entry Process Lifecycle](https://www.ghy.com/trade-talk/us-customs-entry-process-lifecycle/)
- [FMC - Detention and Demurrage](https://www.fmc.gov/detention-and-demurrage/)
- [Maersk - Demurrage and Detention](https://www.maersk.com/logistics-explained/transportation-and-freight/2023/08/28/what-is-demurrage-detention-in-shipping-for-buyers)
- [Global CFS - Centralized Examination Stations](https://globalcfs.com/centralized-examination-station/)
- [CHEMTREC - IMDG for Beginners](https://www.chemtrec.com/resources/blog/imdg-beginners-what-you-need-know-about-shipping-dangerous-goods-sea)
- [CustomsCity - CBP In-Bond Processing](https://customscity.com/cbp-ace-in-bond-processing/)
- [Flexport - Ocean Transit Time Prediction](https://www.flexport.com/blog/predicting-accurate-and-reliable-ocean-freight-transit-times/)
- [project44 - Ocean Visibility](https://www.project44.com/platform/visibility/ocean/)
- [Portcast - Global Ocean Transit Time Trends (2025)](https://www.portcast.io/blog/global-ocean-transit-time-trends-report-april-september-2025)
- [Crowley - FTZ Guide](https://www.crowley.com/logistics/resources/ftz-guide/)
- [NAFTZ - FTZ Basics & Benefits](https://www.naftz.org/basics-benefits/)
- [19 CFR Part 146 - Foreign Trade Zones](https://www.ecfr.gov/current/title-19/chapter-I/part-146)
- [DCSA Information Model (GitHub)](https://github.com/dcsaorg/DCSA-Information-Model)
- [Windward - shipmentLocationTypeCode](https://developer.windward.ai/reference/shipmentlocationtypecode)
- [USCG - AIS Overview](https://navcen.uscg.gov/automatic-identification-system-overview)
- [UNECE - EDIFACT IFTSTA Message](https://service.unece.org/trade/untdid/d21a/trmd/iftsta_c.htm)