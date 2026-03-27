# International Shipment Lifecycle: Comprehensive Reference

> **FedEx Global Clearance Knowledge Base**
> Last Updated: February 2026

---

## Executive Summary

This document provides an exhaustive, field-level reference for the end-to-end lifecycle of an international shipment at a major logistics company. It covers every stage from booking through post-delivery, the business processes and decisions at each stage, the actors and their data requirements, the full customs clearance process with regulatory specifics, the data entities and event-sourcing patterns that model the lifecycle, and the financial and tracking dimensions. This is intended as the definitive domain reference for the Clearance Intelligence Engine platform.

---

## PART 1: END-TO-END SHIPMENT LIFECYCLE STAGES

### 1.1 Lifecycle Overview

An international shipment passes through the following macro-stages. Each stage contains sub-stages with discrete triggers, validations, and branching paths.

```
BOOKING --> PICKUP/ACCEPTANCE --> ORIGIN PROCESSING --> EXPORT CLEARANCE
    --> IN-TRANSIT --> DESTINATION ARRIVAL --> IMPORT CLEARANCE
    --> RELEASE/HOLD --> DESTINATION PROCESSING --> LAST-MILE DELIVERY
    --> PROOF OF DELIVERY --> POST-DELIVERY (BILLING, RECONCILIATION, AUDIT)
```

The lifecycle is NOT strictly linear. Branching paths include:

- **Exception/Hold branch**: At import clearance, shipments may enter holds, exams, detentions, or seizures.
- **Re-routing branch**: At any transit point, shipments may be re-routed due to address corrections, security events, weather, or carrier operational decisions.
- **Return branch**: After delivery (or upon refusal), shipments may enter a return lifecycle that is itself a complete shipment lifecycle in reverse.
- **Abandonment branch**: Shipments that cannot clear customs and are not claimed enter general order, bonded warehouse storage, and eventual disposal or auction.
- **Partial delivery branch**: Consolidated shipments may be partially released while some line items remain held.

### 1.2 Stage 1: Booking / Shipment Request

**Definition**: The initiation of a shipment by a shipper through a carrier's booking system, API, or physical drop-off.

**Sub-stages**:
1. **Booking request**: Shipper provides shipment details (origin, destination, commodity description, weight, dimensions, declared value, service level).
2. **Rate quote / landed cost estimate**: Carrier provides transport cost; for DDP shipments, a duty and tax estimate is included.
3. **Booking confirmation**: Carrier accepts the shipment and assigns a tracking number (air waybill for air freight, bill of lading for ocean, PRO number for ground).
4. **Label/documentation generation**: Shipping labels, commercial invoices, and waybill documents are generated.

**Trigger to next stage**: Shipper tenders the physical package to the carrier (pickup, drop-off, or manifest acceptance).

**Key data captured**:
- Shipper name, address, phone, account number
- Consignee name, address, phone
- Commodity description (free text and/or product catalog reference)
- Declared value and currency
- Weight (gross) and dimensions
- Number of pieces / packages
- Service type (Express, Economy, Freight)
- Incoterms (EXW, FOB, CIF, DDP, DAP, etc.)
- Special handling requirements (hazmat, perishable, temperature-controlled)
- Desired pickup date and time window

**Validations performed**:
- Address validation (origin and destination)
- Service availability for origin-destination pair
- Weight/dimension limits for selected service
- Prohibited/restricted commodity screening (carrier-specific)
- Embargoed country check
- Credit/payment method verification
- Shipper account standing

**Branching paths**:
- If the commodity is prohibited by the carrier (e.g., live animals, hazardous materials without proper packaging), the booking is rejected.
- If the destination is under embargo or sanction, the booking is rejected.
- If the shipper's credit is insufficient or account is suspended, the booking is held pending resolution.

### 1.3 Stage 2: Pickup / Acceptance

**Definition**: Physical custody transfer of the goods from the shipper to the carrier.

**Sub-stages**:
1. **Pickup dispatch**: Carrier dispatches a courier or schedules a pickup.
2. **Pickup scan**: Courier scans the package barcode, confirming physical receipt.
3. **Piece count and weight verification**: Actual weight and piece count are compared to the booking data.
4. **Acceptance or rejection**: Carrier accepts the shipment into its network or rejects it (packaging deficient, exceeds weight limit, prohibited contents detected at pickup).

**Trigger to next stage**: The shipment receives a "Picked Up" or "Shipment Received" scan event.

**Key data produced**:
- Pickup timestamp (local time and UTC)
- Courier ID
- Actual weight and dimensions (if measured)
- Piece count verification
- Pickup location (GPS coordinates for mobile scanners)
- Acceptance/rejection status

**Validations**:
- Physical inspection for packaging integrity
- Dangerous goods labeling check (if applicable)
- Piece count matches manifest
- Weight within declared tolerance (carriers typically allow +/- 0.5 kg for express)

### 1.4 Stage 3: Origin Processing

**Definition**: The shipment is processed at the carrier's origin facility (hub, station, or gateway) for sortation, consolidation, and preparation for export.

**Sub-stages**:
1. **Arrival at origin facility**: Shipment arrives at the carrier's sort facility.
2. **Induction scan**: Barcode scanned into the facility management system.
3. **Sortation**: Shipment is sorted by destination, service level, and routing.
4. **Consolidation**: Individual shipments are consolidated into containers, unit load devices (ULDs for air), or pallets.
5. **Export documentation preparation**: Commercial invoices, packing lists, shipper's export declarations (SED/AES), and any required export licenses are compiled.
6. **Export clearance initiation**: For countries requiring export clearance, the carrier or a customs broker files the export declaration.

**Trigger to next stage**: Shipment is manifested onto an outbound transport unit (flight, vessel, truck) and departs the origin country.

**Key data produced**:
- Facility arrival and departure timestamps
- Container/ULD assignment
- Manifest number
- Flight number / vessel name / truck ID
- Estimated departure time (ETD) and actual departure time (ATD)

### 1.5 Stage 4: Export Clearance

**Definition**: The process of obtaining permission from the origin country's customs authority to export the goods.

**Sub-stages**:
1. **Export declaration filing**: The Automated Export System (AES) filing in the US via the Electronic Export Information (EEI) for shipments exceeding $2,500 or requiring an export license. In the EU, the Export Control System (ECS) processes export declarations.
2. **Export license check**: For controlled goods (dual-use items, military items, items subject to EAR/ITAR in the US, EU Dual-Use Regulation), the export license status is verified.
3. **Denied party screening (export side)**: The shipper, consignee, and any intermediaries are screened against restricted party lists (OFAC SDN, BIS Entity List, BIS Denied Persons List, BIS Unverified List, EU Sanctions, UN Sanctions).
4. **Export release**: The origin country's customs authority releases the goods for export. In many countries, export clearance is relatively fast for non-controlled goods.

**Key regulatory requirements**:
- **US**: AES/EEI filing via AESDirect or ABI for exports over $2,500 or requiring a license. Schedule B classification (6-digit HS-based). Export Control Classification Number (ECCN) for dual-use items. ITAR USML category for defense articles.
- **EU**: Export declaration filed in the exporting member state's customs system. Dual-use items require an export authorization per EU Regulation 2021/821.
- **China**: Export declaration through Single Window. Export licenses required for certain technologies and rare earths.

**Trigger to next stage**: Export clearance is granted and the shipment physically departs the origin country.

**Branching paths**:
- If an export license is required but not obtained, the shipment is held at origin.
- If a denied party match is found, the shipment is stopped and reported to authorities (BIS, OFAC, or equivalent).
- If the goods are subject to export controls and the end-use or end-user raises concerns, the shipment may be detained pending investigation.

### 1.6 Stage 5: In-Transit

**Definition**: The shipment is moving between the origin country and the destination country, potentially through intermediate transit points (hubs, transshipment ports).

**Sub-stages**:
1. **Departure from origin**: The transport unit (aircraft, vessel, truck) departs with the shipment.
2. **Transit hub processing** (if applicable): For hub-and-spoke networks (FedEx Memphis, UPS Louisville, DHL Leipzig), shipments arrive at the hub, are unloaded, re-sorted, and loaded onto outbound transport to the destination.
3. **Advance electronic data submission**: While goods are in transit, the carrier or broker files advance data with the destination country's customs authority:
   - **US**: Air Cargo Advance Screening (ACAS) data; formal entry filing in ACE up to 5 days before arrival.
   - **EU**: ICS2 Entry Summary Declaration (ENS) before goods depart for the EU (express: before departure; air cargo: before loading).
   - **UK**: Safety and security declarations via CHIEF/CDS.
   - **Canada**: Advance Commercial Information (ACI) / eManifest.
   - **Japan**: Advance Filing Rules (AFR).
4. **Pre-clearance processing**: If advance data is complete and accurate, the destination customs authority may process and release the entry before goods arrive. This is the pre-clearance goal.
5. **ISF filing (US ocean only)**: For ocean shipments entering the US, the Importer Security Filing (ISF, also called "10+2") must be filed at least 24 hours before vessel loading at the foreign port of departure. ISF contains 10 data elements from the importer and 2 from the carrier.

**Key ISF 10+2 data elements (US ocean imports)**:
1. Seller name and address
2. Buyer name and address
3. Importer of record number
4. Consignee number
5. Manufacturer (or supplier) name and address
6. Ship-to party name and address
7. Country of origin (of the goods)
8. Commodity HTSUS number (6-digit minimum)
9. Container stuffing location
10. Consolidator name and address
- Plus 2 carrier elements: vessel stow plan and container status messages

**Trigger to next stage**: The shipment arrives at the destination country (aircraft landing, vessel berthing, truck crossing the border).

**Tracking milestones generated during transit**:
- Departed [Origin City]
- Arrived at [Hub]
- Departed [Hub]
- In Transit to [Destination]
- Customs information received (when advance filing is submitted)

### 1.7 Stage 6: Destination Arrival

**Definition**: The shipment physically arrives in the destination country and is presented to or available for customs processing.

**Sub-stages**:
1. **Aircraft/vessel/truck arrival**: The transport unit arrives at the destination port of entry.
2. **Unloading**: Goods are unloaded from the transport unit at the carrier's facility, bonded warehouse, or port terminal.
3. **Break-bulk / deconsolidation**: Consolidated shipments are broken down into individual shipment units.
4. **Arrival scan**: Each individual shipment is scanned at the destination facility, recording its presence.
5. **Manifest reconciliation**: The physical count of shipments is reconciled against the electronic manifest.
6. **Arrival notification to customs**: The carrier notifies the destination customs authority of the arrival of goods (may be automatic via manifest filing or require a separate arrival notification).

**Trigger to next stage**: Shipments are scanned at the destination facility and customs processing begins.

**Key data produced**:
- Actual arrival date and time
- Port of entry
- Facility identifier
- Manifest reconciliation status (matched, short, over)

### 1.8 Stage 7: Import Clearance

This is the most complex and consequential stage of the lifecycle. It is detailed extensively in Section 4 of this document. At a high level:

**Sub-stages**:
1. **Entry preparation**: Compile all required data (HTSUS classification, value, origin, PGA data, party information).
2. **Entry filing**: Submit the customs entry electronically to the destination customs authority (ACE in the US, national customs systems in the EU, CDS in the UK).
3. **Automated risk assessment**: The customs authority's targeting system evaluates the entry against risk rules.
4. **PGA routing**: Entry data is routed to applicable Partner Government Agencies for review.
5. **Decision**: Release, hold, exam, detention, or seizure.

**Trigger to next stage**: The customs authority issues a release decision.

**Branching paths (detailed in Stage 8)**:
- **Clean release**: No holds, no exams. Goods are released for delivery. (~70-95% of shipments depending on trade lane and data quality.)
- **Document hold**: Customs requests additional documentation. Goods remain in carrier custody pending resolution.
- **Exam hold**: Customs orders physical examination (tailgate, intensive, VACIS/NII). Goods must be staged for exam.
- **PGA hold**: A Partner Government Agency (FDA, USDA, CPSC, EPA, FCC, TTB, F&W) holds the shipment pending review.
- **UFLPA detention**: Goods are detained under the rebuttable presumption of forced labor.
- **Seizure**: Goods are seized for violation of law (counterfeiting, prohibited goods, narcotics).
- **Conditional release**: Goods released pending lab results, additional documentation, or bond adjustment.

### 1.9 Stage 8: Release / Hold Resolution

**Definition**: The period between the customs authority's initial decision and the final disposition of the shipment.

**For released shipments**:
1. Release message received from customs authority.
2. Shipment is cleared in the carrier's system.
3. Duty payment is processed (or bonded).
4. Shipment moves to destination processing for delivery.

**For held shipments** (the exception branch):

1. **Hold notification**: The carrier receives a hold message from the customs authority (via ABI/ACE in the US). The hold specifies the reason and the action required.

2. **Caging**: The physical shipment is moved to the cage -- the bonded, secure area within the carrier's facility where held shipments are stored.

3. **Stakeholder notification**: The carrier notifies the importer of record and/or customs broker of the hold. Communication channels vary: ABI messaging, email, phone, portal notification.

4. **Information gathering**: The broker or importer gathers the required information. This may involve contacting:
   - The shipper/supplier (for product specifications, certificates of origin, manufacturing data)
   - Third-party testing labs (for product safety testing, composition analysis)
   - Government agencies (for permits, registrations, prior notices)
   - Legal counsel (for UFLPA detention responses, seizure protests)

5. **Response submission**: The broker submits the response to the customs authority or PGA through the appropriate channel (ACE, PGA-specific systems, or direct communication with CBP import specialists or PGA officials).

6. **Review and decision**: The customs authority or PGA reviews the response and makes a decision:
   - **Release**: Goods are released. Proceed to delivery.
   - **Continued hold**: Additional information required. Cycle repeats.
   - **Exam ordered**: Physical examination scheduled.
   - **Seizure/exclusion**: Goods are seized or ordered excluded from the US.
   - **Voluntary abandonment**: Importer abandons the goods rather than pay duties or continue fighting the hold.

7. **Exam execution** (if ordered):
   - CBP schedules the exam.
   - Carrier stages the shipment at the exam location (carrier facility, Centralized Examination Station/CES, or designated exam site).
   - Exam is conducted (tailgate, intensive, or non-intrusive inspection).
   - Results are processed and entered into ACE.
   - Release, continued hold, or seizure decision follows.

8. **General order** (if unresolved): After 15 days in carrier custody without entry, goods are transferred to a general order warehouse. After 6 months in general order, goods may be sold at auction or destroyed.

**Trigger to next stage**: Release message is received and duty payment is settled.

**Key metrics for hold resolution**:
| Hold Type | Average Resolution Time | Range |
|---|---|---|
| Document request | 2-5 business days | 1-15 days |
| Tailgate exam | 3-7 business days | 2-14 days |
| Intensive exam | 5-10 business days | 3-21 days |
| PGA hold (FDA) | 5-15 business days | 2-45 days |
| PGA hold (USDA) | 3-10 business days | 1-30 days |
| UFLPA detention | 30-90+ business days | 14-180+ days |

### 1.10 Stage 9: Destination Processing

**Definition**: After customs release, the shipment is processed at the carrier's destination facility for final delivery.

**Sub-stages**:
1. **Release confirmation**: The carrier's system updates the shipment status to "Cleared Customs" or "Released for Delivery."
2. **Sort to delivery route**: The shipment is sorted to the appropriate delivery route, station, or last-mile carrier.
3. **Duty/tax collection** (if applicable): For DDU (Delivered Duty Unpaid) or DAP (Delivered At Place) shipments, the carrier collects duties and taxes from the consignee upon delivery. This requires generating a duty invoice and having collection capability at the point of delivery.
4. **Transfer to last-mile carrier** (if applicable): For some destinations, the shipment is transferred from the express carrier's network to a local last-mile delivery partner.
5. **Out for delivery**: The shipment is loaded onto the delivery vehicle.

**Trigger to next stage**: The shipment is out for delivery to the consignee.

### 1.11 Stage 10: Last-Mile Delivery

**Definition**: The physical delivery of the shipment to the consignee's address.

**Sub-stages**:
1. **Delivery attempt**: Courier attempts delivery at the consignee's address.
2. **Delivery or exception**:
   - **Successful delivery**: Package is handed to the consignee or left at the address (per instructions). Proof of delivery (POD) is captured.
   - **Failed delivery attempt**: Consignee not available, address incorrect, refused delivery, access restricted. A delivery exception event is generated.
3. **Delivery reattempt or hold at location**: For failed attempts, the carrier may reattempt the next business day or hold the package at a nearby location for pickup.
4. **Refusal**: The consignee refuses the package (often due to unexpected duty charges on DDU shipments). The shipment enters the return branch.

**Trigger to next stage**: POD captured or final delivery exception disposition reached.

**Proof of delivery data**:
- Delivery timestamp
- Recipient name (printed and/or signature)
- Delivery location (door, reception, locker, neighbor)
- Photo evidence (increasingly standard for express carriers)
- GPS coordinates of delivery
- Delivery driver ID

### 1.12 Stage 11: Post-Delivery

**Definition**: All activities that occur after physical delivery, including billing, audit, reconciliation, and potential returns.

**Sub-stages**:
1. **Invoice generation**: Carrier generates invoices for transport charges, brokerage fees, duty advances, and any ancillary charges (storage, exam fees, special handling).
2. **Duty reconciliation**: For DDP shipments where the carrier advanced duties on behalf of the importer, the carrier invoices the importer for duty reimbursement.
3. **Entry liquidation** (US-specific): CBP liquidates entries approximately 314 days after entry date. The liquidated duty amount may differ from the initial deposit if CBP adjusts classification, value, or rate.
4. **Post-entry amendments**: If errors are discovered after filing, the broker files a Post-Entry Amendment (PEA) or Post-Summary Correction (PSC) in ACE.
5. **Protest**: If the importer disagrees with CBP's liquidation, they may file a protest within 180 days of liquidation.
6. **Drawback claims**: If imported goods are subsequently exported (in original or manufactured form), the importer may file a duty drawback claim to recover up to 99% of duties paid.
7. **Reconciliation entries**: For entries flagged for reconciliation (common for transfer pricing adjustments, retroactive FTA qualification), the final reconciliation entry is filed with corrected data.
8. **Audit response**: If CBP's Regulatory Audit Division initiates a Focused Assessment (FA) or Quick Response Audit (QRA), the importer must provide records and cooperate with the audit.
9. **Returns processing**: If the consignee returns the goods, a return shipment lifecycle begins (new booking, pickup, export clearance from destination country, import clearance at origin country, potential duty refund).

**Trigger to completion**: All financial obligations are settled, entry is liquidated, and the audit window has closed (typically 5 years from entry date for US imports).

---

## PART 2: BUSINESS PROCESSES AT EACH STAGE

### 2.1 Booking Stage Processes

| Process | Operations | Decisions | Validations | Regulatory Requirements |
|---|---|---|---|---|
| Rate calculation | Calculate transport cost based on origin, destination, weight, dimensions, service level | Which service level to offer; whether to accept the shipment | Weight/dimension within limits; hazmat class if applicable | Carrier operating authority for the route |
| Commodity screening | Screen commodity description against carrier prohibited items list | Accept or reject | Description matches known prohibited categories | IATA DGR for air; IMO IMDG Code for ocean |
| Embargo check | Verify destination country is not under comprehensive embargo | Accept or reject | Country code against embargo list | OFAC comprehensive sanctions; EU restrictive measures |
| Landed cost estimation | For DDP, estimate duties, taxes, and fees | Whether to provide a binding quote or estimate | HS code validity; origin country; Incoterms | Tariff schedule accuracy; applicable trade programs |
| Documentation requirements | Determine what documents the shipper must provide | Which documents to request based on commodity and destination | Document completeness | Destination country import requirements |

### 2.2 Origin Processing Processes

| Process | Operations | Decisions | Validations | Regulatory Requirements |
|---|---|---|---|---|
| Sortation | Physical routing of packages through sort facility | Route assignment based on destination and service | Scan verification at each sort point | None (internal operations) |
| Consolidation | Group shipments by destination into containers/ULDs | Optimal container utilization; weight distribution | Container weight limits; balanced loading | IATA ULD regulations; ICAO security requirements |
| Export data compilation | Gather shipper declarations, invoices, certificates | Whether data is sufficient for export filing | Data completeness against destination requirements | AES/EEI filing requirements; export control regulations |
| Dangerous goods acceptance | Verify DG packaging, labeling, and documentation | Accept or reject DG shipment | IATA DGR compliance (air); IMDG compliance (ocean) | IATA DGR 66th Edition (2025); 49 CFR (US); ADR (EU road) |

### 2.3 Import Clearance Processes

This is the most process-intensive stage. The detailed process breakdown is provided in Part 4 (Customs Clearance Specifics). Summary processes:

| Process | Operations | Decisions | Validations | Regulatory Requirements |
|---|---|---|---|---|
| Classification | Assign correct HTSUS/HS code to the commodity | Select from 17,000+ tariff lines (US); determine essential character for multi-material goods | Code validity; GRI application correctness | WCO HS Convention; HTSUS; CBP rulings |
| Valuation | Determine customs value of the goods | Apply correct valuation method (transaction value, deductive, computed, fallback) | Value reasonableness; related-party pricing documentation | WTO Valuation Agreement; 19 USC 1401a |
| Origin determination | Determine country of origin | Substantial transformation test; marking rules vs. preferential origin rules | Origin documentation; manufacturer ID | 19 CFR 134 (marking); FTA-specific rules of origin |
| Entry filing | Submit entry data to customs authority | Entry type (formal, informal, TIB, FTZ, warehouse); timing of filing | All required data fields populated; bond sufficiency | 19 CFR 141-143; ACE filing requirements |
| Duty calculation | Calculate total duties, taxes, and fees | Applicable tariff programs; FTA eligibility; exclusion claims | Rate accuracy; program applicability; stacking order | HTSUS General Notes; Presidential proclamations; FR notices |
| Compliance screening | Screen parties and goods against regulatory requirements | Flag or block shipments with compliance issues | DPS results; PGA requirements; UFLPA exposure | OFAC regulations; BIS EAR; UFLPA; PGA-specific statutes |
| PGA data submission | Submit required data to Partner Government Agencies | Which PGAs are applicable based on HS code and product type | PGA-specific data completeness | FDA FSMA; USDA 7 CFR; EPA TSCA; CPSC CPSA; FCC Part 15/18 |

### 2.4 Post-Delivery Processes

| Process | Operations | Decisions | Validations | Regulatory Requirements |
|---|---|---|---|---|
| Billing/invoicing | Generate invoices for transport, brokerage, duty advance | Payment terms; billing party (shipper vs. consignee vs. third party) | Charge accuracy; authorized billing party | Contract terms; customs bond obligations |
| Liquidation monitoring | Track entry liquidation status in ACE | Whether to protest adverse liquidation | Liquidation amounts vs. initial deposits | 19 USC 1514 (protest); 19 USC 1504 (deemed liquidation) |
| Drawback filing | Compile import-export pairs for drawback claims | Which entries qualify; manufacturing vs. unused merchandise drawback | Import/export linkage documentation; 5-year filing window | 19 USC 1313; 19 CFR 190 |
| Reconciliation | File reconciliation entries with final data | Which entries to reconcile; final values/classifications | Data accuracy; filing window compliance | 19 CFR 181 (USMCA reconciliation); 19 CFR 401 |
| Record retention | Maintain import records for required period | Retention format (electronic vs. paper); storage location | Record completeness; accessibility | 19 CFR 163 (5-year retention for US imports; 10 years for drawback) |

---

## PART 3: ACTORS / ROLES

### 3.1 Complete Actor Matrix

| Actor | Role Description | Stages Active | Data Consumed | Data Produced |
|---|---|---|---|---|
| **Shipper / Exporter** | The party that originates the shipment. Responsible for providing accurate product data, commercial invoices, and origin documentation. | Booking, Pickup, Origin, Export Clearance | Rate quotes, pickup schedules, label data | Product descriptions, commercial invoices, packing lists, certificates of origin, export declarations |
| **Consignee / Buyer** | The intended recipient of the goods. May also be the importer of record. | Booking (DDP), Delivery, Post-Delivery | Tracking updates, delivery notifications, duty invoices | Delivery confirmation, POD signature, duty payment |
| **Importer of Record (IOR)** | The US entity legally responsible for the import entry. Bears liability for classification accuracy, value declaration, duty payment, and compliance. May be the consignee, a customs broker importing on own behalf, or a third party (e.g., a sourcing agent). | Import Clearance, Post-Delivery | Entry data, duty assessments, hold notifications, liquidation notices | Bond information, power of attorney, compliance evidence, protest filings |
| **Customs Broker (Licensed)** | A licensed professional authorized to transact customs business on behalf of importers. In the US, licensed by CBP after passing the Customs Broker License Exam. Files entries, manages holds, advises on classification and compliance. | Import Clearance, Hold Resolution, Post-Delivery | Entry data from carrier/importer, CBP messages (ABI), PGA notifications | Entry filings, duty calculations, hold responses, post-entry amendments, drawback claims |
| **Express Carrier** | The logistics company providing transport, handling, and often brokerage services. (FedEx, UPS, DHL.) Operates hubs, sort facilities, aircraft, and delivery vehicles. Also acts as carrier/broker for customs purposes. | ALL stages | All upstream and downstream data | Tracking events, manifests, carrier certifications, facility scans, delivery records |
| **Freight Forwarder** | An intermediary that arranges transportation on behalf of shippers, particularly for ocean and air freight. May also provide customs brokerage. Acts as an NVOCC (Non-Vessel Operating Common Carrier) for ocean freight. | Booking, Origin, Transit, Destination | Shipper requirements, carrier rates and schedules | Bills of lading (as NVOCC), booking confirmations, arrival notices |
| **Ocean Carrier / Airline** | The vessel operator (Maersk, MSC, CMA CGM) or airline (cargo capacity on passenger or freighter aircraft). Transports goods between ports. | Transit | Booking data, container/cargo details | Vessel/flight manifests, AIS tracking data, arrival notifications |
| **CBP (US Customs and Border Protection)** | The primary US customs authority. Processes entries, assesses risk, orders exams, collects duties, enforces trade law. Operates through ports of entry and Centers of Excellence and Expertise (CEEs). | Import Clearance, Hold Resolution, Post-Delivery | Entry data (via ACE), risk intelligence, trade flow patterns | Release/hold decisions, exam orders, liquidation notices, penalty actions, binding rulings |
| **CBP Import Specialist** | A CBP officer specializing in specific commodity areas (aligned with CEEs: Electronics, Pharmaceuticals, Petroleum, Automotive, etc.). Reviews entries for classification accuracy, value reasonableness, and program applicability. | Import Clearance, Hold Resolution | Entry data, product samples, broker responses, ruling precedents | Classification decisions, hold messages, requests for information (RFIs) |
| **CBP Officer (Port)** | A uniformed CBP officer at the port of entry. Conducts physical examinations, inspects cargo, and enforces border security. | Hold Resolution (Exams) | Exam orders, shipment manifests, risk intelligence | Exam results, seizure reports, chain-of-custody documentation |
| **Partner Government Agencies (PGAs)** | Federal agencies with authority over specific categories of imported goods. Act through ACE (for integrated PGAs) or separate systems. | Import Clearance, Hold Resolution | Entry data (routed via ACE), PGA-specific filings | Admissibility decisions, may-proceed messages, hold/refuse messages |
| **FDA** | Regulates food, drugs, medical devices, cosmetics, biologics, tobacco, radiation-emitting products. Requires prior notice for food, facility registration, and product-specific data. | Import Clearance | Prior notice data, facility registration, product codes, labeling information | Admissibility decision, import alert flags, detention/refusal |
| **USDA/APHIS** | Regulates plant and animal products. Enforces phytosanitary standards, veterinary requirements, and pest exclusion. | Import Clearance | Phytosanitary certificates, veterinary certificates, permit data | Inspection results, release/hold/refuse decisions |
| **EPA** | Regulates chemicals (TSCA), vehicles/engines (emissions), pesticides (FIFRA). Requires import certifications. | Import Clearance | TSCA certifications, emissions compliance data, vehicle identification | Compliance determination, hold/release |
| **CPSC** | Regulates consumer product safety. Screens for products that may pose unreasonable risks. Operates RAM (Risk Assessment Methodology) program for e-commerce. | Import Clearance | Product identification, testing certifications, Certificate of Compliance (COC/CPC) | Screening results, surveillance exam orders |
| **FCC** | Regulates radio frequency devices. Requires equipment authorization for electronic devices that emit RF energy. | Import Clearance | FCC ID, equipment authorization data | Compliance determination |
| **TTB** | Regulates alcohol, tobacco, firearms, ammunition. Requires permits and certificates of label approval. | Import Clearance | Importer permits, COLA (Certificate of Label Approval), tax classification | Admissibility decision |
| **Fish & Wildlife Service** | Enforces CITES, Lacey Act, Endangered Species Act. Regulates wildlife and wildlife products. | Import Clearance | CITES permits, Lacey Act declarations, species identification | Inspection results, seizure (for protected species) |
| **Surety / Bond Company** | Provides customs bonds (continuous or single-entry) that guarantee the importer's duty and compliance obligations. | Import Clearance, Post-Delivery | Importer financial information, entry volume/value history | Bond issuance, bond sufficiency letters, claims |
| **Insurance Company** | Provides cargo insurance covering loss or damage during transit. Coverage depends on Incoterms (CIF includes insurance; FOB typically does not). | Transit, Delivery, Post-Delivery (claims) | Shipment value, commodity type, routing, carrier | Certificate of insurance, claims settlement |
| **Banking / Financial Institution** | For letter of credit (LC) transactions, the banks of the buyer and seller facilitate payment conditional on presentation of conforming documents. Also provides trade finance (supply chain financing, factoring). | Booking (LC issuance), Delivery (document presentation), Post-Delivery (payment) | LC terms, conforming documents (B/L, commercial invoice, insurance certificate, certificate of origin) | LC issuance, payment release, discrepancy notices |
| **Warehouse Operator (Bonded)** | Operates bonded warehouses where goods may be stored under customs bond before entry, during holds, or in general order. Includes Container Freight Stations (CFS) and Centralized Examination Stations (CES). | Hold Resolution, General Order | Warehouse entry orders, exam staging requests | Storage records, exam presentation, general order transfers |
| **Last-Mile Delivery Partner** | In some markets, the express carrier hands off the final delivery to a local delivery partner. | Last-Mile Delivery | Delivery manifest, consignee address, special instructions | Delivery status, POD data |
| **Government Agencies (Other Countries)** | Each country has its own customs authority (CBSA in Canada, HMRC in UK, GACC in China, etc.) with analogous roles to CBP. | Export Clearance (origin), Import Clearance (destination) | Country-specific entry data | Country-specific release/hold decisions |

### 3.2 Actor Interaction at Each Stage

**Booking**: Shipper <--> Carrier; Freight Forwarder may intermediate.
**Pickup**: Shipper <--> Carrier (courier).
**Origin Processing**: Carrier (internal operations).
**Export Clearance**: Carrier/Broker <--> Origin Customs Authority; Shipper provides data.
**Transit**: Carrier <--> Airlines/Ocean Carriers; Carrier <--> Destination Customs (advance filings).
**Destination Arrival**: Carrier <--> Destination Customs (arrival notification).
**Import Clearance**: Carrier/Broker <--> CBP <--> PGAs; Importer/Broker provide data and responses.
**Hold Resolution**: Carrier/Broker <--> CBP/PGAs <--> Importer; Warehouse operators for caging/exam staging.
**Delivery**: Carrier (courier) <--> Consignee; Last-mile partner if applicable.
**Post-Delivery**: Carrier/Broker <--> Importer <--> CBP (liquidation, audit); Banks (payment); Insurance (claims).

---

## PART 4: CUSTOMS CLEARANCE SPECIFICS

### 4.1 Pre-Clearance vs. Arrival Clearance

**Pre-clearance** means completing all customs processing before goods physically arrive at the destination. When achieved, goods are released immediately upon arrival with no dwell time.

**Requirements for pre-clearance**:
- Complete and accurate entry data filed in advance (minimum 1-4 hours before arrival for express; up to 5 days for formal entries)
- No risk flags triggered by automated targeting
- No PGA holds
- Bond sufficiency confirmed
- Duty payment or deposit secured

**Pre-clearance rates by trade lane and data quality**:
| Scenario | Pre-Clearance Rate |
|---|---|
| Commercial express, complete data, trusted shipper | 70-90% |
| Commercial express, average data quality | 50-70% |
| E-commerce, post de minimis, minimal data | 20-40% |
| Ocean freight, formal entry, commercial importer | 60-80% |

**Arrival clearance** means goods arrive before customs processing is complete. Goods must be held at the carrier's facility or port until cleared. This adds dwell time (hours to days) and facility costs.

### 4.2 Required Documents

The following documents are required or commonly used in international trade. Each serves a specific purpose in the clearance process.

#### 4.2.1 Commercial Invoice

**Purpose**: The primary document establishing the transaction between buyer and seller. Provides the basis for customs valuation.

**Required data elements**:
- Seller (exporter) name and address
- Buyer (importer) name and address
- Invoice date and number
- Detailed description of goods (sufficient for customs classification)
- Quantity and unit of measure
- Unit price and total price
- Currency
- Incoterms and point of delivery
- Country of origin of the goods
- Terms of payment
- Marks and numbers on packages
- Total weight (gross and net)

**Why each field matters**:
- *Seller/Buyer*: Identifies the transaction parties; critical for related-party valuation analysis and denied party screening.
- *Description*: Must be detailed enough for the customs authority to verify the declared HS code. "Widget" is insufficient; "stainless steel hex head bolt, M8 x 25mm, Grade A2-70" is appropriate.
- *Unit price*: Basis for ad valorem duty calculation. Must reflect the actual transaction value per WTO Valuation Agreement Article 1.
- *Incoterms*: Determines which costs (freight, insurance, loading) are included in the invoice price. CBP requires the value to be adjusted to FOB (port of export) for US entries; the EU requires CIF (port of import).
- *Country of origin*: Not the same as country of export. Must reflect where goods were manufactured or underwent substantial transformation.

#### 4.2.2 Packing List

**Purpose**: Details the physical contents of each package in the shipment. Used for cargo verification during physical examinations.

**Required data elements**:
- Number of packages/cartons/pallets
- Contents of each package (by item and quantity)
- Net weight and gross weight of each package
- Dimensions of each package
- Marks and numbers matching the commercial invoice

**Why it matters**: During a tailgate or intensive exam, CBP officers use the packing list to verify that the physical contents match the declared entry data. Discrepancies (undeclared goods, quantity shortages, misdescribed items) trigger further investigation.

#### 4.2.3 Bill of Lading (B/L) / Air Waybill (AWB)

**Purpose**: The transport document serving as:
1. A receipt for goods by the carrier
2. Evidence of the contract of carriage
3. A document of title (for ocean B/L -- the holder can claim the goods)

**Types**:
- **Ocean Bill of Lading**: Issued by the ocean carrier (or NVOCC). May be negotiable (order B/L) or non-negotiable (straight B/L). For LC transactions, a negotiable B/L endorsed in blank is typically required.
- **Air Waybill (AWB)**: Issued by the airline or freight forwarder. Always non-negotiable. Governed by IATA Resolution 600a and the Montreal Convention.
- **House Bill of Lading / House AWB**: Issued by a freight forwarder or NVOCC to the shipper. The forwarder consolidates multiple house B/Ls under a single master B/L with the carrier.
- **Multimodal/Combined Transport Document**: Covers transport by more than one mode (e.g., truck to port, ocean to destination port, truck to consignee).

**Key data elements**:
- Shipper and consignee
- Notify party
- Port of loading and port of discharge
- Description of goods, number of packages, weight
- Container numbers (ocean) or flight details (air)
- Freight charges (prepaid or collect)

**Why it matters for customs**: The B/L or AWB is the basis for the manifest filing. CBP reconciles the manifest (from the carrier) with the entry filing (from the broker). Discrepancies between manifest and entry data trigger risk flags.

#### 4.2.4 Certificate of Origin (CO)

**Purpose**: Certifies the country where the goods were manufactured or underwent substantial transformation. Two distinct types exist:

**Non-preferential CO**: Simply states the origin country. Used for MFN duty rate determination, marking requirements, and trade remedy applicability. Often issued by a chamber of commerce in the origin country.

**Preferential CO (FTA certificate)**: Claims eligibility for reduced or zero duty rates under a Free Trade Agreement. Contains additional certifications about how the goods qualify under the FTA's rules of origin.

Examples:
- **USMCA Certificate of Origin**: Can be on any document (no prescribed form). Must include: blanket period (up to 12 months), certifier identification, HS tariff classification, origin criterion (A through D), and a certification statement. Can be completed by the exporter, producer, or importer.
- **EUR.1**: The EU's preferential origin certificate used under many EU FTAs.
- **Form A (GSP)**: Certificate of origin for GSP (Generalized System of Preferences) claims (now largely suspended for US imports).

#### 4.2.5 Importer Security Filing (ISF 10+2) -- US Ocean Only

**Purpose**: Advance security filing required for all ocean cargo entering the US. Must be filed at least 24 hours before loading at the foreign port of departure.

**Penalty for late/inaccurate filing**: $5,000 per violation (liquidated damages). CBP actively enforces ISF compliance.

**Data elements**: See Section 1.6 above.

**Why it matters**: ISF enables CBP to perform risk assessment before goods are loaded onto vessels destined for the US. It supports the Container Security Initiative (CSI) and the Customs-Trade Partnership Against Terrorism (C-TPAT).

#### 4.2.6 AES / Electronic Export Information (EEI)

**Purpose**: US export declaration filed in the Automated Export System. Required for:
- Exports valued over $2,500 per Schedule B number per destination
- All exports requiring an export license
- Certain exports to specific destinations (Cuba, North Korea, Iran, Syria, Russia)
- Rough diamonds (regardless of value)

**Data elements**:
- USPPI (US Principal Party in Interest -- the exporter)
- Ultimate consignee
- Intermediate consignee (if applicable)
- Forwarding agent
- Schedule B number (6-digit, harmonized with HS)
- Quantity and unit of measure
- Value (selling price or cost if not sold)
- Export Control Classification Number (ECCN) or EAR99
- License number or license exception symbol

**Filing deadline**: By the date of export for most shipments. For vessel shipments, 24 hours before loading.

**Internal Transaction Number (ITN)**: Upon successful filing, AES returns an ITN that must be annotated on the B/L or AWB as proof of filing.

#### 4.2.7 Power of Attorney (POA)

**Purpose**: Legal authorization from the importer of record granting a customs broker the authority to transact customs business on the importer's behalf. Required by 19 CFR 141.46.

**Types**:
- **Limited POA**: For a specific transaction or set of transactions.
- **General (Continuing) POA**: Authorizes the broker for all customs business until revoked.

**Why it matters**: Without a valid POA, a broker cannot file an entry. The POA also establishes the legal relationship between the importer and broker, affecting liability for errors and penalties.

#### 4.2.8 Customs Bond

**Purpose**: A financial guarantee (issued by a surety company licensed by the US Treasury) ensuring that the importer will pay duties, taxes, and fees, and comply with customs laws. Required by 19 USC 1623.

**Types**:
- **Single-entry bond (STB)**: Covers one specific import entry. Bond amount is typically the greater of the entered value plus estimated duties, or $1,000. Used for occasional importers.
- **Continuous bond (annual)**: Covers all entries during a 12-month period. Minimum bond amount is $50,000 or 10% of duties/taxes/fees paid in the prior year, whichever is greater. Used by regular importers.

**Bond sufficiency**: CBP may issue a bond insufficiency notice if the importer's bond amount is inadequate given their import volume and duty obligations. This is increasingly common as tariff rates increase under stacking programs.

### 4.3 HS / HTS Classification Process

**The Harmonized System (HS)** is maintained by the World Customs Organization (WCO) and is used by over 200 countries. It provides a uniform classification system for goods up to the 6-digit level.

**Structure**:
- **Chapters (2 digits)**: 99 chapters organized into 22 sections. Example: Chapter 87 = Vehicles other than railway.
- **Headings (4 digits)**: ~1,200 headings. Example: 8708 = Parts and accessories for motor vehicles.
- **Subheadings (6 digits)**: ~5,000 subheadings. Harmonized internationally. Example: 8708.30 = Brakes and servo-brakes; parts thereof.
- **National tariff lines (8-10 digits)**: Country-specific subdivisions. US uses 10 digits (HTSUS). EU uses 10 digits (CN/TARIC). Example: 8708.30.50.00 (US) = Brakes and parts, other.

**Classification methodology -- General Rules of Interpretation (GRI)**:
1. **GRI 1**: Classification is determined by the terms of the headings and relevant section/chapter notes. (Start with the heading text.)
2. **GRI 2(a)**: Incomplete or unfinished articles are classified as the complete article if they have its essential character.
3. **GRI 2(b)**: Mixtures and combinations of materials are classified per GRI 3.
4. **GRI 3(a)**: When goods are classifiable under two or more headings, the most specific heading prevails.
5. **GRI 3(b)**: Composite goods are classified by the material or component that gives them their essential character.
6. **GRI 3(c)**: If GRI 3(a) and 3(b) fail, the heading last in numerical order is used.
7. **GRI 4**: Goods not classifiable elsewhere are classified under the heading for the most similar goods.
8. **GRI 5**: Cases, containers, and packing materials are classified with the goods they contain (with exceptions).
9. **GRI 6**: Classification at the subheading level uses the same principles applied to headings.

**Classification disputes**:
- CBP may issue a Request for Information (RFI) or a Notice of Action (CF 29) questioning the declared classification.
- The importer/broker responds with technical justification (product specifications, engineering drawings, lab analysis).
- If unresolved, the importer may request a binding ruling from CBP's National Commodity Specialist Division (NCSD) or Headquarters.
- CBP rulings are searchable in the CROSS (Customs Rulings Online Search System) database.
- Rulings take 30-120 days on average; complex cases may take longer.

### 4.4 Duty Calculation

#### 4.4.1 Types of Duty Rates

**Ad valorem**: A percentage of the customs value. Example: 2.5% of CIF value. The most common type. Approximately 80% of US tariff lines use ad valorem rates.

**Specific**: A fixed amount per unit of quantity. Example: 2.8 cents per kilogram. Common for agricultural products, beverages, and some industrial goods.

**Compound**: A combination of ad valorem and specific. Example: 6.5% + 4.4 cents per kilogram. Used for some textile and apparel products.

**Mixed/Alternative**: The higher or lower of two calculations. Example: "20% or $1.50/kg, whichever is higher."

#### 4.4.2 The Duty Calculation Stack (US Imports)

For a US formal entry, duty is calculated by stacking the following layers. The order and applicability depend on the HS code, country of origin, and available program claims.

**Layer 1: MFN (Column 1 General) Duty**
- The baseline duty rate from the HTSUS "General" column.
- Applied to imports from all WTO member countries and countries granted Normal Trade Relations (NTR) status.
- Ranges from 0% (free) to over 30%.
- Determined by the 8-digit HTSUS code.

**Layer 2: Special Tariff Programs (Additive)**

| Program | Legal Authority | Applicability | Rate Range | Applied To |
|---|---|---|---|---|
| Section 301 | Trade Act of 1974, Sec 301-310 | China-origin goods on designated lists | 7.5% - 100% | Declared value |
| Section 232 | Trade Expansion Act of 1962, Sec 232 | Steel (25%), aluminum (10-25%), derivatives | 10% - 25% | Declared value |
| IEEPA Reciprocal | International Emergency Economic Powers Act | Country-specific reciprocal tariffs | 10% - 145% | Declared value |
| AD duties | Tariff Act of 1930, Title VII | Products subject to AD orders, by manufacturer | 0% - 200%+ | Declared value |
| CVD duties | Tariff Act of 1930, Title VII | Products subject to CVD orders, by country/program | 0% - 100%+ | Declared value |

**Layer 3: Preferential Programs (Subtractive)**

| Program | Benefit | Applicability |
|---|---|---|
| USMCA (formerly NAFTA) | Reduced or zero duty | Goods qualifying under USMCA rules of origin; origin in US, Mexico, or Canada |
| US-Korea FTA (KORUS) | Reduced or zero duty | Korean-origin goods meeting KORUS rules of origin |
| US-Australia FTA | Reduced or zero duty | Australian-origin goods meeting FTA rules of origin |
| US-Singapore FTA | Reduced or zero duty | Singaporean-origin goods meeting FTA rules of origin |
| US-Chile FTA | Reduced or zero duty | Chilean-origin goods meeting FTA rules of origin |
| US-Israel FTA | Reduced or zero duty | Israeli-origin goods (no formal certificate required, but origin must be documented) |
| GSP (Generalized System of Preferences) | Reduced or zero duty | Currently suspended for most countries |
| AGOA (African Growth and Opportunity Act) | Reduced or zero duty | Eligible African country goods meeting AGOA rules |
| CBI (Caribbean Basin Initiative) | Reduced or zero duty | Eligible Caribbean/Central American country goods |
| Section 321 (de minimis) | Duty-free (SUSPENDED) | Was available for shipments under $800; suspended Aug 29, 2025 |

**FTA preference claims reduce or eliminate the MFN duty rate but do NOT eliminate Section 301, 232, IEEPA, or AD/CVD duties.** This is a critical distinction: USMCA can zero out the 2.5% MFN rate on auto parts from Mexico, but if the parts contained Chinese-origin steel, the Section 232 tariff on the steel content may still apply (depending on the specific USMCA rule of origin and substantial transformation analysis).

**Layer 4: Fees**

| Fee | Rate | Minimum | Maximum | Applicability |
|---|---|---|---|---|
| Merchandise Processing Fee (MPF) | 0.3464% of value | $31.67 | $614.35 | All formal entries. USMCA-origin goods: flat fee of $2, $6, or $9 depending on value. |
| Harbor Maintenance Fee (HMF) | 0.125% of value | None | None | Ocean freight imports only. Not charged on exports. |
| Cotton Fee | Varies | Varies | Varies | Cotton and cotton waste imports (negligible for most importers) |

**Total calculation**: Landed cost = Declared Value + MFN Duty + Section 301 + Section 232 + IEEPA + AD/CVD - FTA Reduction + MPF + HMF + Other Fees

#### 4.4.3 Valuation for Duty Purposes

CBP determines the customs value using the WTO Valuation Agreement hierarchy:

1. **Transaction value** (Article 1): The price actually paid or payable for the goods when sold for export to the US. This is the primary basis for the vast majority of entries. Adjustments (additions) include:
   - Selling commissions
   - Assists (tools, dies, molds, engineering provided by the buyer to the seller free of charge or at reduced cost)
   - Royalties and license fees paid as a condition of the sale
   - Packing costs
   - Proceeds of subsequent resale that accrue to the seller

   **Deductions** (not included in customs value):
   - Freight, insurance, and other charges incurred AFTER importation (for US; note: EU uses CIF, which includes freight and insurance)
   - US duties and taxes
   - International transport costs (for FOB-based US valuation)

2. **Transaction value of identical goods** (Article 2): If transaction value cannot be determined (no sale, related-party pricing without arm's length demonstration).
3. **Transaction value of similar goods** (Article 3): Similar goods, same country of origin, approximately same time.
4. **Deductive value** (Article 5): Based on the resale price in the US, less deductions for profit, transport, duties.
5. **Computed value** (Article 6): Based on the cost of production plus profit and general expenses.
6. **Fallback method** (Article 7): Reasonable means consistent with the Agreement's principles.

**Currency conversion**: CBP uses quarterly exchange rates published by the Federal Reserve Bank of New York. The rate applicable is the one in effect on the date of exportation. If the invoice currency differs from USD, the carrier/broker must convert using the CBP-certified rate.

### 4.5 Trade Agreements and Preferential Duty Rates

#### 4.5.1 USMCA (United States-Mexico-Canada Agreement)

**Effective**: July 1, 2020 (replaced NAFTA).
**Review**: Subject to a Joint Review every 6 years (first review: 2026).

**Rules of Origin**: Products must meet specific rules to qualify for preferential treatment:

1. **Wholly obtained or produced**: Goods entirely grown, harvested, or extracted in USMCA territory.
2. **Tariff shift**: Non-originating materials must undergo a change in HS classification specified in the product-specific rules (e.g., change to Chapter 62 from any other chapter, except Chapters 51-55 and 60).
3. **Regional Value Content (RVC)**: The product must contain a minimum percentage of North American content:
   - **Transaction value method**: RVC = (TV - VNM) / TV x 100, where TV = transaction value, VNM = value of non-originating materials. Typical threshold: 60%.
   - **Net cost method**: RVC = (NC - VNM) / NC x 100, where NC = net cost (total cost minus sales promotion, royalties, shipping and packing, non-allowable interest). Typical threshold: 50%.
   - **Automotive-specific**: Vehicles require 75% RVC (phased in). Core parts require 70-75%. Steel and aluminum must be "melted and poured" or "smelted and cast" in North America.
4. **De minimis**: Up to 10% of the transaction value of the good may consist of non-originating materials that do not meet a required tariff shift (7% for textiles).

**Automotive rules** (the most complex USMCA rules):
- 75% RVC for vehicles (net cost method)
- 70% for core parts (engines, transmissions, bodies, chassis, axles, suspensions, steering, advanced batteries)
- 65% for principal parts
- Labor Value Content (LVC): 40% of vehicle value from high-wage (>$16/hour) manufacturing; 25% for light trucks
- 70% of steel and aluminum must be melted/poured or smelted/cast in North America

#### 4.5.2 Other US FTAs

| Agreement | Partner(s) | Key Features |
|---|---|---|
| KORUS | South Korea | Duty elimination on most industrial goods; complex rules for textiles |
| US-Australia | Australia | Most goods duty-free; investor-state dispute resolution |
| US-Singapore | Singapore | Comprehensive duty elimination |
| US-Chile | Chile | Phased duty elimination completed |
| US-Colombia | Colombia | TPA (Trade Promotion Agreement) |
| US-Panama | Panama | TPA |
| US-Peru | Peru | TPA |
| US-Israel | Israel | Oldest US FTA (1985); no formal CO required |
| CAFTA-DR | Central America, Dominican Republic | Regional agreement with product-specific rules |
| US-Jordan | Jordan | Industrial goods largely duty-free |
| US-Morocco | Morocco | Phased elimination |
| US-Bahrain | Bahrain | Phased elimination |
| US-Oman | Oman | Phased elimination |

### 4.6 Denied Party Screening (DPS)

Every shipment -- import and export -- must be screened against multiple government-maintained lists of restricted parties. The screening must cover all transaction parties: shipper, consignee, ultimate end-user, intermediate consignee, and any other known parties.

#### 4.6.1 Key Screening Lists

| List | Maintaining Agency | Contents | Legal Basis |
|---|---|---|---|
| **SDN (Specially Designated Nationals)** | OFAC (Treasury) | Individuals and entities owned/controlled by targeted countries or designated under sanctions programs | IEEPA, various sanctions statutes |
| **Entity List** | BIS (Commerce) | Foreign entities deemed contrary to US national security or foreign policy interests. License required for virtually all exports. | EAR Section 744.11 |
| **Denied Persons List** | BIS (Commerce) | Individuals/entities denied export privileges by BIS | EAR Section 764 |
| **Unverified List** | BIS (Commerce) | Parties for whom BIS was unable to complete an end-use check | EAR Section 744.15 |
| **Military End User (MEU) List** | BIS (Commerce) | Chinese, Russian, Venezuelan military end users | EAR Section 744.21 |
| **UFLPA Entity List** | FLETF (DHS) | Entities in Xinjiang or using Xinjiang-sourced inputs | Uyghur Forced Labor Prevention Act |
| **Non-SDN Chinese Military-Industrial Complex Companies List** | OFAC (Treasury) | Chinese military-industrial complex companies | Executive Order 13959 (as amended) |
| **Consolidated Screening List (CSL)** | ITA (Commerce) | Aggregation of multiple lists for convenience | Multiple |
| **EU Consolidated Sanctions List** | EU Council | Sanctioned individuals and entities under EU restrictive measures | EU Common Foreign and Security Policy |
| **UN Security Council Consolidated List** | UN | Individuals and entities subject to UN sanctions | UN Security Council Resolutions |

#### 4.6.2 Screening Process

1. **Extract party names and addresses** from all transaction documents (commercial invoice, B/L, entry data).
2. **Normalize names**: Transliterate non-Latin scripts; standardize abbreviations (Co., Ltd., Corp., Inc.); remove punctuation and special characters.
3. **Exact match**: Search for exact matches against all applicable lists.
4. **Fuzzy match**: Search for near-matches using algorithms:
   - Levenshtein distance (edit distance)
   - Jaro-Winkler similarity
   - Trigram (pg_trgm) similarity
   - Phonetic matching (Soundex, Metaphone)
   - Token-based matching (for name element reordering)
5. **Threshold determination**: Matches above a configurable threshold (typically 85-95% similarity) are flagged for human review.
6. **Human review**: A compliance analyst reviews flagged matches to determine if they are true positives (actual restricted party) or false positives (different entity with a similar name).
7. **Disposition**: True positive matches are blocked and reported. False positives are documented and cleared.

**Consequences of a true positive match**:
- Transaction must be blocked (no shipping, no entry filing, no payment).
- Voluntary self-disclosure to the relevant agency (OFAC, BIS) if the transaction has already proceeded.
- Potential civil and criminal penalties for violations.
- OFAC civil penalties can reach the greater of $356,579 per violation or twice the transaction value (as of 2025 adjustments).

### 4.7 Anti-Dumping / Countervailing Duties (AD/CVD)

**Anti-dumping duties**: Imposed when foreign producers sell goods in the US at less than fair value (below home market price or cost of production), causing material injury to a US industry. Determined by the Department of Commerce (International Trade Administration) and the International Trade Commission (ITC).

**Countervailing duties**: Imposed when foreign governments provide subsidies to their producers, causing material injury to a US industry. Determined by Commerce and ITC.

**Key characteristics**:
- **Product-specific scope**: Each AD/CVD order covers a defined product scope (e.g., "certain steel nails from China" or "crystalline silicon photovoltaic cells from Taiwan").
- **Company-specific rates**: Commerce calculates separate rates for individual foreign producers/exporters. Companies not individually investigated receive an "all others" rate (AD) or a country-wide rate (CVD).
- **Annual administrative reviews**: Rates are recalculated annually based on updated sales and subsidy data. New rates are published in the Federal Register.
- **Cash deposits**: Importers must deposit estimated AD/CVD duties at the time of entry, based on the most recent published rate.
- **Liquidation with actual rate**: After the annual review is completed (often 12-18 months after the review period), entries are liquidated at the final determined rate. Importers pay the difference or receive a refund.

**AD/CVD evasion enforcement**:
- **EAPA (Enforce and Protect Act)**: Allows any party to file an allegation of AD/CVD evasion with CBP. CBP investigates and may impose duties retroactively.
- **Transshipment**: Goods shipped through third countries (e.g., China to Vietnam to US) to avoid AD/CVD duties. CBP actively investigates transshipment schemes using supply chain analysis.
- **Country of origin manipulation**: Minimal processing in a third country to change the country of origin. CBP applies the "substantial transformation" test to determine true origin.

### 4.8 Section 301 and Section 232 Tariffs

**Section 301 (China tariffs)**:
- Four lists (Lists 1-4) covering approximately $370 billion of Chinese-origin goods.
- Rates: 7.5% (List 4a) to 100% (EVs, post-2024 review).
- **Exclusions**: Some products have been granted temporary exclusions from Section 301 tariffs. Exclusions must be applied for and are granted for specific products based on HS code and product description. Exclusions expire and may or may not be renewed.
- **HS code matching**: Section 301 applicability is determined by 8-digit HTSUS code. A product's classification must be checked against the published lists to determine if Section 301 applies and at what rate.
- **Country of origin, not country of export**: Section 301 applies to goods of Chinese origin, regardless of where they were shipped from. A product manufactured in China and transshipped through Vietnam is still subject to Section 301 duties.

**Section 232 (Steel and aluminum)**:
- 25% on steel imports; 10-25% on aluminum imports.
- **Country of melt and pour** (steel) or **country of smelting and casting** (aluminum) determines applicability, not country of manufacture of the finished product.
- **Derivative products**: In 2025, Section 232 was extended to downstream products (articles containing steel or aluminum) from certain countries.
- **Product exclusions**: Available through a petition process; granted for specific products from specific countries when domestic supply is insufficient.
- **Quota arrangements**: Some countries have negotiated quota arrangements in lieu of tariffs (e.g., certain EU steel quotas).

**IEEPA reciprocal tariffs**:
- Country-specific rates announced via executive order.
- Applied to all goods from the designated country (with limited exceptions).
- Rates have changed frequently (announced, paused, modified, reinstated) creating compliance challenges.
- Current rates (as of early 2026): China 145%, EU 20%, most other countries 10% baseline.

### 4.9 Exam Types and Holds

| Exam Type | Description | Typical Duration | Who Conducts |
|---|---|---|---|
| **Document Review (CET)** | Additional documents requested; no physical exam. CBP Compliance Exam Team reviews remotely. | 2-5 days | CBP Import Specialist (remote) |
| **Tailgate Exam** | Visual and physical inspection at the carrier's facility. Officers verify contents, markings, and quantities. | 3-7 days | CBP Officer at carrier facility |
| **Intensive Exam** | Detailed physical examination involving unpacking, measuring, sampling, or testing. May be at a CES. | 5-14 days | CBP Officer at CES or carrier facility |
| **VACIS / NII Exam** | Non-Intrusive Inspection using X-ray or gamma-ray imaging. Container/package is scanned without opening. | 1-3 days | CBP Officer using NII equipment |
| **FDA Exam/Sample** | FDA collects samples for laboratory analysis (food safety, drug composition, device testing). | 7-45 days | FDA field office / contracted lab |
| **USDA Inspection** | USDA/APHIS inspects plant/animal products for pests, diseases, or contamination. | 2-10 days | USDA inspector at port |
| **CPSC Surveillance** | CPSC targets and tests consumer products for safety compliance. | 5-21 days | CPSC field staff / contracted lab |

### 4.10 FDA Prior Notice and PGA Requirements

**FDA Prior Notice (Food)**:
- All food imports must have prior notice filed with FDA before arrival.
- Filing deadlines depend on transport mode:
  - Air: no later than 4 hours before arrival (2 hours for express consignments)
  - Ocean: no later than 8 hours before arrival at the first US port
  - Truck: no later than 2 hours before arrival at the US border
  - Rail: no later than 4 hours before arrival at the US border
- Filed through the FDA Prior Notice System Interface (PNSI) or through ACE.
- Required data: description of food article, FDA product code, manufacturing establishment, anticipated arrival information.

**USDA/APHIS Permits**:
- Plant products may require an import permit (PPQ 587) issued before importation.
- Animal products require a veterinary services (VS) import permit.
- Phytosanitary certificates issued by the origin country's plant protection authority must accompany plant products.
- USDA port inspectors physically inspect regulated products upon arrival.

**EPA Requirements**:
- **TSCA (Toxic Substances Control Act)**: All chemical substances imported into the US must comply with TSCA. The importer must certify that the chemicals are either (a) on the TSCA Inventory and in compliance, or (b) exempt. Filed via EPA's CDX (Central Data Exchange) system.
- **Vehicle emissions**: Imported vehicles and engines must comply with EPA emissions standards. Requires a declaration form (EPA 3520-1) and evidence of compliance (test results, EPA certificate of conformity).

### 4.11 Foreign Trade Zones (FTZ)

**Definition**: FTZs are designated areas within or near US ports of entry that are legally considered outside the customs territory of the US for duty purposes. Goods admitted to an FTZ are not subject to duties until they are formally entered into US commerce.

**Benefits**:
- **Duty deferral**: Duties are not paid until goods leave the FTZ for US consumption.
- **Duty elimination on re-exports**: If goods admitted to an FTZ are subsequently exported, no US duties are paid.
- **Inverted tariff benefit**: If a finished product has a lower duty rate than its components, manufacturing in the FTZ allows the importer to pay the lower finished-good rate (Production Activity authorization required).
- **Weekly entry**: FTZ users may file a single weekly entry covering all goods that left the FTZ during the week, reducing per-entry processing costs and fees.

**Types**:
- **General Purpose Zone**: Operated by a grantee (typically a port authority or economic development agency). Open to multiple users.
- **Subzone**: A special-purpose site, usually an individual company's manufacturing facility, approved for FTZ status.

### 4.12 Temporary Imports

**ATA Carnet**: An international customs document (issued under the ATA Convention and the Istanbul Convention) that allows temporary duty-free import of goods for specific purposes:
- Professional equipment (tools, cameras, musical instruments)
- Commercial samples
- Goods for exhibitions and trade fairs

The Carnet serves as both the customs declaration and the bond. Goods must be re-exported within the Carnet's validity period (typically 12 months). No duties are paid as long as the goods are re-exported.

**Temporary Importation under Bond (TIB)**: US-specific mechanism for temporary duty-free importation. Entry Type 06 or 23. Bond amount is typically double the estimated duties. Goods must be re-exported within 1 year (extendable to 3 years). Common uses: goods for repair, testing, exhibition, or processing that will be re-exported.

### 4.13 Drawback Claims

**Duty drawback** allows importers to recover up to 99% of duties, taxes, and fees paid on imported goods that are subsequently exported.

**Types**:
1. **Manufacturing drawback**: Imported goods are used as inputs in manufacturing, and the manufactured product is exported. Example: imported steel is used to make machinery that is exported.
2. **Unused merchandise drawback**: Imported goods are exported in the same condition as imported (or destroyed under CBP supervision) within 5 years.
3. **Rejected merchandise drawback**: Imported goods are found to be defective or not as ordered and are returned to the foreign seller.
4. **Substitution drawback**: Goods that are commercially interchangeable with the imported goods (same HS code, same quality) are exported, even if the specific imported goods were consumed domestically.

**Filing requirements**:
- Must be filed within 5 years of importation.
- Requires linking specific import entries to specific export transactions.
- Documentation: import entry, commercial invoices, export documentation, proof of exportation.
- Processing time: 6 months to 3 years (CBP has significant backlogs).

---

## PART 5: DATA PRODUCTS / STRUCTURES

### 5.1 Core Data Entities

#### 5.1.1 Shipment

The central entity representing a physical movement of goods from origin to destination.

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| tracking_number | String(30) | Immutable | Primary external identifier (AWB, PRO, B/L). Assigned at booking. Never changes. |
| master_tracking_number | String(30) | Immutable | For consolidated shipments, the master AWB/B/L. Links house-level shipments to the master. |
| service_type | Enum | Immutable | Service level (EXPRESS, ECONOMY, FREIGHT, GROUND). Determines routing and transit time. |
| transport_mode | Enum | Mutable (rare) | AIR, OCEAN, ROAD, RAIL, MULTIMODAL. May change if shipment is re-routed. |
| incoterms | String(3) | Immutable | EXW, FOB, CIF, DDP, DAP, etc. Set by the sales contract. Affects valuation and duty responsibility. |
| status | Enum | Mutable | Current lifecycle status (BOOKED, PICKED_UP, IN_TRANSIT, AT_CUSTOMS, CLEARED, OUT_FOR_DELIVERY, DELIVERED, RETURNED, HELD, SEIZED). |
| shipper_account_id | String(20) | Immutable | Carrier account number of the shipper. Links to the customer master. |
| created_at | Timestamp | Immutable | When the shipment record was created. |
| updated_at | Timestamp | Mutable | Last modification timestamp. |
| estimated_delivery_date | Date | Mutable | Recalculated as shipment progresses and exceptions occur. Time-sensitive. |

#### 5.1.2 Shipment Party

Captures the actors associated with a shipment. Multiple parties per shipment.

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| shipment_id | FK | Immutable | Links to the parent shipment. |
| party_role | Enum | Immutable | SHIPPER, CONSIGNEE, IMPORTER_OF_RECORD, BROKER, MANUFACTURER, NOTIFY_PARTY, SOLD_TO, SHIP_TO |
| party_name | String(500) | Mutable (corrections) | Legal name of the party. May be corrected if initial data was inaccurate. |
| party_address | JSONB | Mutable (corrections) | Structured address: street, city, state/province, postal_code, country_code. |
| party_id_type | String(20) | Immutable | Type of identification: EIN, DUNS, MID, CBP_ASSIGNED, VAT_NUMBER |
| party_id_value | String(50) | Mutable (corrections) | The identifier value. May be corrected post-filing. |
| dps_screened | Boolean | Mutable | Whether this party has been screened against denied party lists. |
| dps_result | JSONB | Mutable | Screening results: list of matches with scores and dispositions. |
| dps_screened_at | Timestamp | Mutable | When screening was last performed. Time-sensitive: must be re-screened if lists are updated. |

#### 5.1.3 Shipment Line Item

Individual commodity lines within a shipment. A shipment may contain multiple line items with different HS codes, origins, and values.

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| shipment_id | FK | Immutable | Links to the parent shipment. |
| line_number | Integer | Immutable | Sequential line number within the shipment. |
| product_description | Text | Mutable (pre-filing) | Free-text description of the commodity. Ideally corrected/enhanced before entry filing. |
| hs_code | String(15) | Mutable (pre-filing, dispute) | Harmonized System code. May be corrected during classification review or dispute. |
| hs_code_confidence | Float | Mutable | Confidence score for AI-assisted classification (0.0 to 1.0). |
| country_of_origin | String(2) | Mutable (rare, dispute) | ISO 3166-1 alpha-2 code. May change if CBP disputes origin. |
| country_of_export | String(2) | Immutable | Where the goods physically shipped from. |
| manufacturer_id | String(30) | Mutable (corrections) | CBP Manufacturer ID (MID). Format: CC + first 3 chars of name + first 3 of city + first 4 of address. |
| declared_value | Decimal(15,2) | Mutable (valuation adjustment) | Transaction value in the declared currency. May be adjusted for assists, royalties, or CBP challenge. |
| currency | String(3) | Immutable | ISO 4217 currency code. |
| quantity | Decimal(15,3) | Mutable (corrections) | Quantity in the declared unit of measure. |
| unit_of_measure | String(20) | Mutable (corrections) | UOM aligned with HTSUS statistical unit (PCS, KG, DZ, M2, etc.). |
| gross_weight_kg | Decimal(10,3) | Mutable (measured) | Actual gross weight. May differ from declared weight. |
| net_weight_kg | Decimal(10,3) | Mutable | Weight of goods excluding packaging. |

#### 5.1.4 Customs Entry

The formal customs declaration filed with the destination customs authority. One entry may cover one or more shipments (consolidated entry) or one shipment may have multiple entries (split shipments).

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| entry_number | String(15) | Immutable | CBP-assigned entry number. Format: 3-digit filer code + 7-digit entry number + 1 check digit. |
| entry_type | String(2) | Immutable | CBP entry type code: 01 (formal), 03 (consumption), 06 (TIB), 11 (informal), 21 (warehouse), 26 (FTZ). |
| entry_date | Date | Immutable | Date the entry was filed. Starts the 314-day liquidation clock. |
| port_of_entry | String(4) | Immutable | CBP 4-digit port code (e.g., 4601 = Memphis, 2704 = JFK, 2809 = Newark). |
| importer_of_record | FK | Immutable | Links to the Shipment Party designated as IOR. |
| bond_type | Enum | Immutable | CONTINUOUS or SINGLE_ENTRY. |
| bond_number | String(20) | Immutable | Customs bond identifier. |
| surety_code | String(3) | Immutable | 3-digit surety company code (Treasury-assigned). |
| status | Enum | Mutable | FILED, ACCEPTED, RELEASED, HELD, EXAM_ORDERED, CONDITIONALLY_RELEASED, LIQUIDATED, PROTESTED. |
| release_date | Date | Mutable | Date CBP issued the release. Null if not yet released. |
| liquidation_date | Date | Mutable | Date entry was liquidated. Null if not yet liquidated. Approximately entry_date + 314 days. |
| liquidation_type | Enum | Mutable | AS_ENTERED (no change), INCREASED (duty increase), DECREASED (duty decrease), VOIDED. |
| total_duty_deposited | Decimal(15,2) | Mutable | Total estimated duties deposited at time of entry. |
| total_duty_liquidated | Decimal(15,2) | Mutable | Final duty amount after liquidation. May differ from deposit. |
| reconciliation_flag | Boolean | Immutable | Whether the entry is flagged for subsequent reconciliation filing. |
| fta_claimed | String(20) | Immutable | FTA program claimed (USMCA, KORUS, etc.) or null if no preference claimed. |

#### 5.1.5 Duty Line Item

Individual duty, tax, and fee calculations for each combination of line item and tariff program.

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| entry_id | FK | Immutable | Links to the parent entry. |
| line_item_id | FK | Immutable | Links to the shipment line item. |
| program | String(30) | Immutable | MFN, SEC301, SEC232, IEEPA, AD, CVD, MPF, HMF, FTA_REDUCTION |
| rate_type | Enum | Immutable | AD_VALOREM, SPECIFIC, COMPOUND |
| rate | Decimal(10,6) | Immutable | The rate applied (percentage for ad valorem, $/unit for specific). |
| dutiable_value | Decimal(15,2) | Mutable | The value against which the rate is applied. |
| calculated_amount | Decimal(15,2) | Mutable | Rate x dutiable_value (or rate x quantity for specific duties). |
| legal_citation | String(200) | Immutable | Statutory or regulatory citation (e.g., "HTSUS 9903.88.03", "19 USC 1673"). |
| effective_date | Date | Immutable | Date the rate became effective. |

#### 5.1.6 Compliance Screening Result

Records the outcome of denied party screening, UFLPA assessment, and other compliance checks.

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| shipment_id | FK | Immutable | Links to the shipment. |
| screening_type | Enum | Immutable | DPS, UFLPA, EXPORT_CONTROL, PGA_REQUIREMENT |
| screened_party_name | String(500) | Immutable | The name that was screened. |
| list_source | String(30) | Immutable | Which list triggered the match (OFAC_SDN, BIS_ENTITY, UFLPA_ENTITY, etc.) |
| match_score | Float | Immutable | Similarity score (0.0 to 1.0) for fuzzy matches. |
| match_type | Enum | Immutable | EXACT, FUZZY, ALIAS |
| disposition | Enum | Mutable | PENDING_REVIEW, TRUE_POSITIVE, FALSE_POSITIVE, BLOCKED. |
| reviewed_by | String(100) | Mutable | ID of the analyst who reviewed the match. |
| reviewed_at | Timestamp | Mutable | When the match was reviewed. |
| notes | Text | Mutable | Analyst notes on the disposition. |

#### 5.1.7 Tracking Event

The event log for a shipment. Captures every milestone and exception in the lifecycle.

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| shipment_id | FK | Immutable | Links to the shipment. |
| event_timestamp | Timestamp | Immutable | When the event occurred (UTC). |
| event_code | String(10) | Immutable | Standardized event code (see Section 7.2). |
| event_description | String(500) | Immutable | Human-readable description. |
| location_code | String(10) | Immutable | Facility/port/station code where the event occurred. |
| location_name | String(200) | Immutable | Facility/port/station name. |
| location_country | String(2) | Immutable | Country where the event occurred. |
| scan_type | Enum | Immutable | PICKUP, DEPARTURE, ARRIVAL, SORT, CUSTOMS, DELIVERY, EXCEPTION |
| additional_data | JSONB | Immutable | Event-specific data (e.g., exam type for customs hold events, recipient name for delivery events). |

#### 5.1.8 Document

Trade documents associated with a shipment.

| Attribute | Type | Mutability | Purpose |
|---|---|---|---|
| shipment_id | FK | Immutable | Links to the shipment. |
| document_type | Enum | Immutable | COMMERCIAL_INVOICE, PACKING_LIST, BILL_OF_LADING, AIR_WAYBILL, CERTIFICATE_OF_ORIGIN, PHYTOSANITARY_CERT, CERTIFICATE_OF_CONFORMITY, POWER_OF_ATTORNEY, CUSTOMS_BOND, FDA_PRIOR_NOTICE, TSCA_CERT, EXPORT_LICENSE, AES_ITN |
| document_number | String(50) | Immutable | Document reference number. |
| document_date | Date | Immutable | Date on the document. |
| file_reference | String(500) | Immutable | Reference to the stored document (file path, S3 key, etc.). |
| uploaded_by | String(100) | Immutable | Who uploaded the document. |
| uploaded_at | Timestamp | Immutable | When the document was uploaded. |
| verified | Boolean | Mutable | Whether the document has been verified by an analyst. |
| expiry_date | Date | Mutable | For documents with expiry (export licenses, Carnets, bonds). Time-sensitive. |

### 5.2 Entity Relationships

```
Shipment (1) ----< (N) ShipmentParty
Shipment (1) ----< (N) ShipmentLineItem
Shipment (1) ----< (N) TrackingEvent
Shipment (1) ----< (N) Document
Shipment (1) ----< (N) ComplianceScreeningResult
Shipment (N) >---- (1) CustomsEntry (many shipments can consolidate into one entry)
CustomsEntry (1) ----< (N) DutyLineItem
ShipmentLineItem (1) ----< (N) DutyLineItem
CustomsEntry (N) >---- (1) ShipmentParty [as IOR]
```

### 5.3 Mutable vs. Immutable Data

**Immutable data** (never changes after creation):
- Tracking numbers, entry numbers, document numbers
- Timestamps (event_timestamp, created_at, entry_date)
- Event codes and event data (tracking events are append-only)
- Service type and Incoterms (set by contract)
- Legal citations, rate types, program codes

**Mutable data** (may change during the lifecycle):
- Shipment status (changes at each stage)
- HS code (may be corrected or disputed)
- Declared value (may be adjusted for assists, royalties, or CBP challenge)
- Party details (corrections to names, addresses, IDs)
- Duty amounts (adjusted at liquidation)
- Estimated delivery date (recalculated on exceptions)
- Compliance screening results (disposition changes from PENDING to TRUE/FALSE_POSITIVE)

**Time-sensitive data** (has temporal constraints):
- FDA prior notice (must be filed 2-24 hours before arrival)
- ISF 10+2 (must be filed 24 hours before vessel loading)
- AES/EEI (must be filed by date of export)
- Entry filing (should be filed before or at arrival for pre-clearance)
- Liquidation (approximately 314 days from entry date)
- Protest (180 days from liquidation)
- Drawback (5 years from importation)
- Record retention (5 years from entry, 10 years for drawback)
- Customs bond expiry (annual renewal for continuous bonds)
- ATA Carnet validity (typically 12 months)
- UFLPA detention response (30 days from detention notice, extendable)

### 5.4 Event Sourcing Patterns

The shipment lifecycle is naturally modeled using event sourcing. Each state change is captured as an immutable event. The current state of a shipment is derived by replaying all events in sequence.

#### 5.4.1 Lifecycle Events

| Event Type | Payload | Triggers |
|---|---|---|
| `ShipmentBooked` | tracking_number, shipper, consignee, service_type, commodity_info, declared_value, incoterms | Booking confirmation |
| `ShipmentPickedUp` | pickup_timestamp, courier_id, actual_weight, piece_count | Pickup scan |
| `ShipmentDeparted` | facility_code, transport_id (flight/vessel/truck), ETD, ATD | Departure scan |
| `ShipmentArrived` | facility_code, transport_id, ETA, ATA | Arrival scan |
| `ShipmentInTransit` | current_facility, next_facility, transport_id | Hub-to-hub movement |
| `CustomsFilingSubmitted` | entry_number, entry_type, port_of_entry, filing_timestamp | Entry filed in ACE |
| `CustomsReleased` | entry_number, release_timestamp, release_type (PRE_CLEARANCE / POST_ARRIVAL) | CBP release message |
| `CustomsHeld` | entry_number, hold_type (DOC_REVIEW / EXAM / PGA / UFLPA), hold_reason, requesting_agency | CBP/PGA hold message |
| `ExamOrdered` | entry_number, exam_type (TAILGATE / INTENSIVE / VACIS), exam_location | CBP exam order |
| `ExamCompleted` | entry_number, exam_result (RELEASED / HELD / SEIZED), findings | CBP exam result |
| `HoldResolved` | entry_number, resolution (RELEASED / SEIZED / EXCLUDED / ABANDONED), resolution_notes | CBP/PGA resolution message |
| `DutyCalculated` | entry_number, line_items: [{program, rate, amount}], total_duty, total_fees | Duty computation completed |
| `DutyPaid` | entry_number, payment_amount, payment_method (ACH / CHECK / BOND), payment_reference | Duty payment confirmed |
| `ShipmentOutForDelivery` | delivery_route_id, estimated_delivery_time | Loaded on delivery vehicle |
| `ShipmentDelivered` | delivery_timestamp, recipient_name, signature_image_ref, gps_coordinates, delivery_location_type | POD captured |
| `DeliveryFailed` | attempt_number, failure_reason (NOT_HOME / WRONG_ADDRESS / REFUSED / ACCESS_DENIED), reattempt_date | Delivery exception |
| `ShipmentReturned` | return_reason (REFUSED / UNDELIVERABLE / CUSTOMER_REQUEST), return_tracking_number | Return initiated |
| `EntryLiquidated` | entry_number, liquidation_date, liquidation_type, liquidated_duty_amount, difference_from_deposit | CBP liquidation posted |
| `ProtestFiled` | entry_number, protest_number, protest_date, protest_reason | Importer protest |
| `PostEntryAmendment` | entry_number, amendment_type (PSC / PEA), fields_changed, original_values, new_values | Correction filed |
| `DrawbackClaimed` | drawback_entry_number, linked_import_entries, linked_export_entries, claimed_amount | Drawback application |
| `ClassificationChanged` | line_item_id, old_hs_code, new_hs_code, change_reason (CORRECTION / CBP_RECLASSIFICATION / RULING) | HS code change |
| `ValueAdjusted` | line_item_id, old_value, new_value, adjustment_reason (ASSIST / ROYALTY / CBP_CHALLENGE) | Value correction |

#### 5.4.2 Event Ordering and Consistency

Events are ordered by event_timestamp. For events occurring simultaneously (e.g., multiple line items released at the same time), a sequence_number provides deterministic ordering.

**Idempotency**: Each event has a unique event_id (UUID). Duplicate events (same event_id) are discarded. This supports at-least-once delivery semantics in distributed systems.

**Causality**: Events carry a `caused_by` field linking to the event_id that triggered them. Example: `CustomsHeld.caused_by = CustomsFilingSubmitted.event_id`. This enables causal chain analysis for exception root cause identification.

---

## PART 6: FINANCIAL / COMMERCIAL DATA

### 6.1 Incoterms and Their Impact on Duty Calculation

Incoterms (International Commercial Terms), published by the International Chamber of Commerce (ICC), define the responsibilities of buyer and seller for delivery, risk transfer, and cost allocation. The 2020 edition (Incoterms 2020) defines 11 terms in two categories.

**Impact on customs valuation**:
- **US (CBP)**: Customs value is based on the **FOB (port of export)** value. If the invoice is on CIF terms, the importer must deduct international freight and insurance to arrive at the FOB value.
- **EU / most other countries**: Customs value is based on **CIF (port of import)** value. If the invoice is on FOB terms, the importer must add international freight and insurance.

| Incoterm | Full Name | Risk Transfer Point | Freight Included? | Insurance Included? | Duty Responsibility (typical) |
|---|---|---|---|---|---|
| EXW | Ex Works | Seller's premises | No | No | Buyer |
| FCA | Free Carrier | Named place of delivery | Origin transport only | No | Buyer |
| FAS | Free Alongside Ship | Alongside vessel at port | To port | No | Buyer |
| FOB | Free On Board | On board vessel at port | To vessel loading | No | Buyer |
| CFR | Cost and Freight | On board vessel at port | To destination port | No | Buyer |
| CIF | Cost, Insurance, Freight | On board vessel at port | To destination port | Yes (minimum) | Buyer |
| CPT | Carriage Paid To | Delivered to first carrier | To named destination | No | Buyer |
| CIP | Carriage and Insurance Paid To | Delivered to first carrier | To named destination | Yes (all risks) | Buyer |
| DAP | Delivered At Place | Named destination | To destination | Seller's discretion | Buyer (duty at destination) |
| DPU | Delivered at Place Unloaded | Named destination, unloaded | To destination + unloading | Seller's discretion | Buyer |
| DDP | Delivered Duty Paid | Named destination | To destination | Seller's discretion | **Seller** (seller pays all duties) |

**DDP is the most consequential Incoterm for clearance operations**: The seller assumes full responsibility for customs clearance and duty payment in the destination country. This means:
- The seller must have an importer of record arrangement (either the seller is the IOR, or a third party acts as IOR on the seller's behalf).
- The seller must accurately estimate duties and include them in the selling price.
- If the actual duty exceeds the estimate, the seller absorbs the difference.
- If the shipment is held or detained, the seller is responsible for resolution.

### 6.2 Insurance and Freight Charges

**Cargo Insurance Types**:
- **All Risks**: Covers all physical loss or damage during transit (with standard exclusions for war, strikes, inherent vice). The broadest coverage.
- **Named Perils**: Covers only specifically listed risks (fire, collision, sinking, etc.).
- **Institute Cargo Clauses (ICC)**: Standardized coverage tiers:
  - ICC(A): All risks (most comprehensive)
  - ICC(B): Named perils, medium coverage
  - ICC(C): Named perils, basic coverage

**Insurance and customs valuation**: Under CIF Incoterms, insurance cost is included in the customs value. Under FOB, insurance is NOT included (for US valuation; but IS included for EU valuation if insurance was paid for the international leg).

**Freight charges and customs valuation**:
- US: International freight is EXCLUDED from customs value (FOB basis). Freight from the foreign port of export to the US port of arrival is not dutiable.
- EU: International freight IS INCLUDED in customs value (CIF basis). The cost of transporting goods to the EU border is part of the dutiable value.
- This means the SAME shipment has a different customs value depending on whether it enters the US or the EU, which directly affects the duty amount.

### 6.3 Currency Conversion

**US Customs**: CBP uses quarterly exchange rates published by the Federal Reserve Bank of New York (Treasury rates). The applicable rate is the rate in effect on the date of exportation. These rates are published in the CBP Bulletin and on the CBP website.

**Why Treasury rates matter**: Using a different exchange rate (e.g., commercial rate on the date of payment, or the rate on the date of entry) can result in a different customs value and therefore a different duty amount. CBP strictly enforces the use of certified rates.

**Conversion process**: If the commercial invoice is denominated in EUR, the broker converts the value to USD using the CBP-certified EUR/USD rate effective on the date of exportation. This USD value becomes the declared value on the entry.

### 6.4 Duty Payment and Bonds

**Payment timing**: Estimated duties must be deposited with CBP before or at the time of entry filing (for formal entries). For pre-clearance, this means duty must be deposited electronically before goods arrive.

**Payment methods**:
- **ACH (Automated Clearing House)**: Electronic funds transfer. Most common for regular importers with continuous bonds.
- **Daily Statement**: Importers on periodic payment file entries during the statement period and pay duties in a single daily or periodic payment.
- **Pay.gov**: Online payment for individual entries.
- **Customs Bond**: The bond guarantees duty payment but is NOT a payment. If the importer fails to pay duties, CBP makes a claim against the bond, and the surety company pays.

**Continuous Bond vs. Single-Entry Bond**:

| Feature | Continuous Bond | Single-Entry Bond |
|---|---|---|
| Coverage | All entries for 12 months | One specific entry |
| Minimum amount | $50,000 or 10% of prior year duties | Greater of value + duties or $1,000 |
| Cost (annual premium) | 0.5-2% of bond amount | 1-5% of bond amount |
| Typical user | Regular importers (>4 entries/year) | Occasional importers |
| Bond sufficiency | Must cover anticipated duties; CBP may demand increase | Per-entry calculation |

**Bond sufficiency crisis**: With stacking tariffs increasing effective duty rates to 50-150%+ on many goods, importers' existing bond amounts are frequently insufficient. CBP issues bond insufficiency notices requiring the importer to increase the bond amount within 30 days or face entry rejection.

### 6.5 Reconciliation

**CBP reconciliation** is a process that allows importers to flag entries where certain data elements are not final at the time of entry. The entry is filed with the best available information, and a reconciliation entry is filed later with the final data.

**Common reconciliation scenarios**:
1. **Value reconciliation**: Transfer pricing adjustments between related parties (e.g., year-end price adjustments). The initial entry uses the provisional transfer price; the reconciliation entry uses the final price.
2. **Classification reconciliation**: Product is under review for correct classification; entry is filed with the current best classification, flagged for reconciliation if the classification changes.
3. **FTA reconciliation**: FTA preference is claimed provisionally; the importer is still gathering origin documentation. Reconciliation entry confirms or withdraws the FTA claim.
4. **9802 (US goods returned) reconciliation**: Value of US-origin components in assembled products is not final at time of entry.

**Filing deadline**: Reconciliation entries must be filed within 21 months of the earliest entry in the reconciliation.

---

## PART 7: TRACKING AND VISIBILITY

### 7.1 Milestone Events

Major logistics companies publish standardized milestone events that are visible to shippers and consignees through tracking interfaces. The following represents the industry-standard milestone framework (with minor variations by carrier).

| Milestone | Description | Typical Carrier Code |
|---|---|---|
| Shipment information sent to carrier | Electronic shipment data received before physical pickup | SI |
| Picked up | Physical package received by carrier | PU |
| Arrived at origin facility | Received at carrier's origin sort facility | AR |
| Departed origin facility | Left origin sort facility | DP |
| Export clearance in progress | Export customs processing initiated | XC |
| Export clearance completed | Export customs release granted | XR |
| In transit | Shipment moving between facilities | IT |
| Arrived at hub / transit facility | Received at intermediate sort hub | AH |
| Departed hub / transit facility | Left intermediate hub | DH |
| Arrived at destination facility | Received at destination gateway/sort facility | AD |
| Import clearance in progress | Import customs processing initiated | CC |
| Import clearance completed / Released | Import customs release granted | CR |
| Customs hold | Shipment held by customs authority | CH |
| Customs exam ordered | Physical examination ordered by customs | CE |
| Customs exception | Issue requiring resolution before release | CX |
| Cleared customs | All customs processing complete | CL |
| In transit to delivery station | Moving from gateway to local delivery station | DT |
| Arrived at delivery station | At local station for delivery | DS |
| Out for delivery | On delivery vehicle en route to consignee | OD |
| Delivery attempted | Delivery attempted but unsuccessful | DA |
| Delivered | Successfully delivered to consignee | DL |
| Delivery exception | Issue preventing delivery (wrong address, refused, etc.) | DE |
| Return initiated | Return shipment created | RI |
| Return in transit | Return shipment moving back to origin | RT |
| Return delivered | Return shipment delivered to original shipper | RD |

### 7.2 Status Codes

Beyond milestone events, carriers use detailed status codes for internal operations and customs processing. The following represents the customs-specific status framework.

| Code | Status | Description |
|---|---|---|
| CUS-PREP | Entry Preparation | Data being compiled for customs filing |
| CUS-FILED | Entry Filed | Entry submitted to customs authority |
| CUS-ACCEPTED | Entry Accepted | Customs authority acknowledged receipt |
| CUS-SCREENING | Under Screening | Entry being processed by targeting/risk systems |
| CUS-PGA-ROUTED | PGA Routed | Entry data sent to Partner Government Agencies |
| CUS-RELEASED | Released | Customs authority issued release |
| CUS-COND-REL | Conditionally Released | Released pending further action |
| CUS-HOLD-DOC | Document Hold | Additional documentation requested |
| CUS-HOLD-EXAM | Exam Hold | Physical examination ordered |
| CUS-HOLD-PGA | PGA Hold | Partner Government Agency hold |
| CUS-HOLD-VAL | Valuation Hold | Value questioned by customs |
| CUS-HOLD-CLASS | Classification Hold | Classification questioned by customs |
| CUS-HOLD-BOND | Bond Insufficiency | Customs bond insufficient |
| CUS-UFLPA | UFLPA Detention | Detained under forced labor presumption |
| CUS-SEIZED | Seized | Goods seized by customs authority |
| CUS-EXCLUDED | Excluded | Entry denied; goods must be exported or destroyed |
| CUS-GEN-ORDER | General Order | Transferred to general order warehouse |
| CUS-ABANDONED | Abandoned | Importer abandoned the goods |

### 7.3 Exception Management

Exceptions are deviations from the expected lifecycle flow. Each exception type has a defined escalation path, resolution workflow, and communication protocol.

**Exception Categories**:

1. **Customs Exceptions** (highest impact on clearance operations):
   - Missing or incorrect HS code
   - Missing or insufficient product description
   - Value discrepancy (under/over declared)
   - Missing country of origin
   - PGA documentation missing (FDA prior notice, USDA certificate, etc.)
   - DPS match (restricted party screening hit)
   - UFLPA flag (forced labor risk)
   - Bond insufficiency
   - Classification dispute
   - Exam ordered

2. **Transport Exceptions**:
   - Misrouted shipment
   - Damaged package
   - Lost/missing package
   - Weather delay
   - Flight/vessel cancellation
   - Security hold (TSA, CBP, or carrier-imposed)
   - Overweight / over-dimension

3. **Delivery Exceptions**:
   - Address not found
   - Consignee not available
   - Refused by consignee (often due to unexpected duty charges)
   - Restricted access (gated community, secure building)
   - Customs charges due on delivery (COD duty collection failure)

**Exception resolution workflow**:
1. Exception event generated (automated detection or manual creation).
2. Exception categorized and severity assessed (Critical, High, Medium, Low).
3. Responsible party identified (carrier, broker, importer, shipper, customs authority).
4. Notification sent to responsible party and stakeholders.
5. Action taken (provide documentation, pay duties, correct data, schedule exam).
6. Resolution confirmed and exception closed.
7. Shipment returns to normal lifecycle flow.

### 7.4 Proof of Delivery (POD)

POD is the definitive record that goods were delivered to the consignee. It serves as:
- Evidence of contract fulfillment (the carrier delivered as agreed)
- Trigger for payment release (in LC transactions, POD may be a required document)
- Starting point for claims (if goods are damaged, the POD captures condition at delivery)
- Compliance record (for controlled substances, age-restricted goods, and high-value items requiring identity verification)

**POD data elements**:
- Date and time of delivery
- Recipient printed name
- Recipient signature (digital or physical)
- Relationship to consignee (self, spouse, colleague, reception, etc.)
- Delivery location type (front door, back door, reception desk, locker, neighbor, FedEx office)
- Photo evidence (photo of package at delivery location)
- GPS coordinates (latitude/longitude of delivery point)
- Driver/courier ID
- Special notes (e.g., "left with reception per instructions")

**Digital POD**: Most major carriers now capture POD digitally through mobile devices carried by delivery drivers. The digital signature, photo, and GPS coordinates are uploaded in real-time and immediately available through tracking interfaces.

---

## PART 8: STANDARDS AND SYSTEMS REFERENCE

### 8.1 Key Standards Bodies and Their Domains

| Organization | Full Name | Domain | Key Standards |
|---|---|---|---|
| **WCO** | World Customs Organization | Customs procedures and data | Harmonized System (HS); WCO Data Model; Revised Kyoto Convention; SAFE Framework |
| **IATA** | International Air Transport Association | Air cargo | Air Waybill (Resolution 600a); Cargo-IMP (IATA Cargo Interchange Message Procedures); e-freight; Dangerous Goods Regulations (DGR) |
| **IMO** | International Maritime Organization | Maritime transport | IMDG Code (dangerous goods); ISPS Code (security); SOLAS (safety); ISM Code (management) |
| **ICC** | International Chamber of Commerce | Trade terms and finance | Incoterms 2020; UCP 600 (Letters of Credit); Uniform Rules for Collections (URC 522) |
| **UN/CEFACT** | UN Centre for Trade Facilitation and Electronic Business | Electronic trade documents | UN/EDIFACT; UNTDID; XML schemas for trade documents |
| **ISO** | International Organization for Standardization | Various | ISO 3166 (country codes); ISO 4217 (currency codes); ISO 6346 (container codes); ISO 8601 (dates/times) |
| **GS1** | Global Standards Organization | Product and shipment identification | GTIN (Global Trade Item Number); SSCC (Serial Shipping Container Code); GS1-128 barcode |

### 8.2 Key Government Systems

| System | Country/Region | Purpose |
|---|---|---|
| **ACE (Automated Commercial Environment)** | US | Primary system for processing all US trade data. Handles entry filing, targeting, PGA messaging, duty collection, and trade data dissemination. |
| **ABI (Automated Broker Interface)** | US | The electronic interface through which customs brokers file entries and receive CBP messages in ACE. Uses ANSI X12 EDI or, increasingly, JSON-based web services. |
| **AES (Automated Export System)** | US | System for filing Electronic Export Information (EEI) for US exports. Accessed via AESDirect (web) or ABI. |
| **ACAS (Air Cargo Advance Screening)** | US | Advance data submission for air cargo security screening. Minimum data set required before aircraft departure to the US. |
| **CROSS (Customs Rulings Online Search System)** | US | Searchable database of CBP classification and other rulings. Legal precedent for classification disputes. |
| **ICS2 (Import Control System 2)** | EU | Pre-arrival security and safety data system. Phased implementation (Release 1-3) covering all transport modes. |
| **CDS (Customs Declaration Service)** | UK | UK's primary customs declaration system (replaced CHIEF in 2023). |
| **TradeNet / NTP** | Singapore | National single window for trade declarations. Considered the global gold standard for trade facilitation. |
| **Single Window** | China | China's single window for customs declarations, inspections, and trade facilitation. |
| **CBSA Portal** | Canada | Canada Border Services Agency online portal for importers and customs brokers. |

### 8.3 Key Identifiers Used in International Trade

| Identifier | Full Name | Format | Assigned By | Purpose |
|---|---|---|---|---|
| **AWB** | Air Waybill Number | 3-digit prefix + 8-digit serial + check digit | Airline (IATA prefix) | Identifies an air cargo shipment. Prefix identifies the issuing airline (e.g., 023 = FedEx). |
| **B/L** | Bill of Lading Number | Carrier-specific format | Ocean carrier / NVOCC | Identifies an ocean cargo shipment. Also a document of title. |
| **SCAC** | Standard Carrier Alpha Code | 2-4 letter alpha code | NMFTA | Identifies US-registered carriers (e.g., FDEG = FedEx Ground, FDCC = FedEx Custom Critical). |
| **MID** | Manufacturer Identification Code | CC + name + city + address fragments | CBP (derived from entry data) | Identifies the foreign manufacturer. Required on US entries since 2009. |
| **EIN** | Employer Identification Number | XX-XXXXXXX | IRS | US tax ID. Used as the importer of record number for US entries. |
| **CBP Assigned Number** | Importer number for non-EIN entities | Format varies | CBP | For importers without an EIN (e.g., foreign entities, individuals). |
| **HTSUS Code** | Harmonized Tariff Schedule Number | 10 digits (XXXX.XX.XXXX) | USITC | Classifies the imported product and determines the duty rate (US-specific at 8-10 digit level). |
| **Schedule B** | Export commodity number | 10 digits | Census Bureau | Classifies exported products. Harmonized with HS at 6-digit level. |
| **ECCN** | Export Control Classification Number | Alphanumeric (e.g., 3A001) | BIS (Commerce) | Classifies items for export control purposes under the EAR. |
| **Entry Number** | Customs entry identifier | 3 + 7 + 1 (11 digits) | CBP (filer code assigned to broker) | Uniquely identifies a customs entry. Filer code identifies the broker. |
| **ISF Transaction Number** | ISF filing identifier | System-generated | CBP/ACE | Identifies an Importer Security Filing for ocean cargo. |
| **ITN** | Internal Transaction Number | AES-generated | Census/AES | Confirms successful AES/EEI export filing. Must appear on the B/L or AWB. |
| **Container Number** | Intermodal container ID | 4 letters + 7 digits | ISO 6346 | Identifies shipping containers globally. Owner code (3 letters) + equipment category (1 letter) + serial (6 digits) + check digit. |
| **IMO Number** | Vessel identification | 7 digits | IMO | Uniquely identifies ocean-going vessels. Permanent, does not change with ownership. |

---

## PART 9: RETURN / REVERSE LOGISTICS LIFECYCLE

Returns represent a complete lifecycle in reverse, with their own customs implications.

### 9.1 Return Scenarios

1. **Consumer return (e-commerce)**: Consignee returns product to the original seller in the origin country.
2. **Warranty return**: Defective product returned to manufacturer for repair or replacement.
3. **Refused delivery**: Consignee refuses the package (typically due to unexpected duty charges). Package is returned to shipper.
4. **Re-export after hold/exam**: Goods that fail to clear customs are exported back to the origin country.

### 9.2 Return Customs Implications

**Exporting from the destination country**:
- A return shipment from the US requires export clearance (AES filing if value exceeds $2,500).
- The goods must clear the destination country's customs authority for export.

**Importing back into the origin country**:
- The goods must clear the origin country's customs authority for import.
- Depending on the origin country's rules, the returning goods may be eligible for duty-free treatment as "returned goods" (e.g., US: HTSUS 9801.00 for US goods returned; EU: Article 203 of UCC for returned goods).

**Duty recovery**:
- Under rejected merchandise drawback (19 USC 1313(c)), an importer may recover duties paid on goods that were imported and subsequently returned because they were defective or not as ordered.
- The goods must be returned within 3 years of importation.
- Documentation linking the original import entry to the return shipment is required.

---

## PART 10: OCEAN FREIGHT SPECIFICS

While the Clearance Intelligence Engine focuses on express clearance, ocean freight represents the majority of international trade by volume and has distinct lifecycle characteristics.

### 10.1 Ocean Freight Lifecycle Differences

| Aspect | Express (Air) | Ocean Freight |
|---|---|---|
| Transit time | 1-5 days | 15-45 days |
| Shipment unit | Individual package/pallet | Container (20ft/40ft/40ftHC) |
| Tracking granularity | Package-level, real-time | Container-level, periodic (AIS) |
| Advance filing | ACAS (before departure) | ISF 10+2 (24 hours before loading) |
| Pre-clearance | Common (data submitted 1-4 hours before arrival) | Less common (entry often filed at or after arrival) |
| Entry filing timing | Before or at arrival | At or after arrival; up to 15 days after arrival |
| Exam location | Carrier facility, CES | Marine terminal, CES, or off-dock facility |
| Container exam cost | N/A | $500-$5,000+ (depending on type: VACIS, intensive, devanning) |
| Demurrage/detention charges | None (goods in carrier custody) | Significant (container at port beyond free time: $100-$300/day; chassis: $50-$150/day) |
| Drayage | N/A | Truck transport from port to warehouse: $200-$1,500+ |

### 10.2 Ocean-Specific Documents

- **Ocean Bill of Lading (OBL)**: Document of title. The original (negotiable) B/L must be surrendered to the carrier to take delivery of the container. For LC transactions, the B/L is presented to the bank as proof of shipment.
- **Sea Waybill**: Non-negotiable. Consignee can claim goods with proof of identity (no original document needed). Faster release but no security for the seller.
- **Mate's Receipt**: Issued by the vessel when goods are loaded. Exchanged for the B/L.
- **Delivery Order**: Issued by the carrier or agent authorizing the terminal to release the container to the consignee or their agent.

### 10.3 Maersk Lifecycle Example

Maersk, as the world's largest container shipping company (operating approximately 700 vessels), provides a representative ocean freight lifecycle:

1. **Booking**: Shipper books container space through Maersk.com or API. Receives a booking number.
2. **Empty container release**: Maersk releases an empty container to the shipper for loading (stuffing).
3. **Container stuffing**: Shipper loads goods into the container. Container is sealed with a high-security seal (ISO 17712). Seal number is recorded.
4. **Gate-in at origin terminal**: Loaded container is delivered to the port terminal. Container details verified (weight, seal number, external condition).
5. **Export clearance**: Customs declaration filed with origin country customs. AES/EEI filed (US exports). Export permit obtained if required.
6. **Vessel loading**: Container loaded onto vessel. Stow plan filed (carrier's 2 elements of ISF).
7. **Vessel departure**: Vessel departs origin port. AIS tracking begins.
8. **In transit**: Vessel transits ocean route. May stop at transshipment ports where the container is offloaded and loaded onto a connecting vessel.
9. **ISF filing**: Importer (or broker) files ISF 10+2 at least 24 hours before loading at the LAST foreign port before the US.
10. **Entry filing**: Entry filed in ACE. May be filed up to 15 days before arrival.
11. **Vessel arrival**: Vessel arrives at US port. Berths at marine terminal.
12. **Discharge**: Container offloaded from vessel to terminal yard.
13. **CBP processing**: CBP processes the entry. Release, hold, or exam decision.
14. **Container release**: If released, the terminal issues a container release. The consignee or their trucker can pick up the container.
15. **Drayage**: Container transported by truck from the marine terminal to the consignee's warehouse.
16. **Container stripping**: Goods unloaded from the container at the warehouse.
17. **Empty container return**: Empty container returned to the terminal or designated depot within the carrier's free time period (typically 4-7 days).

---

## Appendix A: Glossary of Key Terms

| Term | Definition |
|---|---|
| **ACE** | Automated Commercial Environment -- the US government's single window for trade processing |
| **ABI** | Automated Broker Interface -- the electronic connection between customs brokers and ACE |
| **AD/CVD** | Anti-Dumping / Countervailing Duties |
| **AEO** | Authorized Economic Operator -- a trusted trader program (WCO SAFE Framework) |
| **ACAS** | Air Cargo Advance Screening -- advance security data for air cargo |
| **AWB** | Air Waybill -- the contract of carriage for air freight |
| **B/L** | Bill of Lading -- the contract of carriage for ocean freight (also a document of title) |
| **CBP** | US Customs and Border Protection |
| **CEE** | Center of Excellence and Expertise -- CBP's industry-focused trade processing centers |
| **CES** | Centralized Examination Station -- a facility where CBP conducts exams |
| **CIF** | Cost, Insurance, and Freight -- Incoterm where seller pays freight and insurance to destination port |
| **CO** | Certificate of Origin |
| **C-TPAT** | Customs-Trade Partnership Against Terrorism |
| **DDP** | Delivered Duty Paid -- Incoterm where seller bears all costs including duties |
| **DDU** | Delivered Duty Unpaid (superseded by DAP in Incoterms 2020) |
| **DPS** | Denied Party Screening |
| **EAR** | Export Administration Regulations (BIS) |
| **ECCN** | Export Control Classification Number |
| **EEI** | Electronic Export Information (filed in AES) |
| **FOB** | Free On Board -- Incoterm where seller delivers goods onto the vessel |
| **FTZ** | Foreign Trade Zone |
| **GRI** | General Rules of Interpretation (HS classification rules) |
| **HMF** | Harbor Maintenance Fee |
| **HS** | Harmonized System -- the international 6-digit commodity classification |
| **HTSUS** | Harmonized Tariff Schedule of the United States (10-digit) |
| **ICS2** | Import Control System 2 (EU pre-arrival data) |
| **IEEPA** | International Emergency Economic Powers Act |
| **IOR** | Importer of Record |
| **ISF** | Importer Security Filing (10+2) for ocean cargo |
| **ITAR** | International Traffic in Arms Regulations (State Department) |
| **ITN** | Internal Transaction Number (AES confirmation) |
| **MFN** | Most Favored Nation (standard duty rate for WTO members) |
| **MID** | Manufacturer Identification Code |
| **MPF** | Merchandise Processing Fee |
| **NII** | Non-Intrusive Inspection (X-ray/gamma-ray scanning) |
| **NVOCC** | Non-Vessel Operating Common Carrier |
| **OFAC** | Office of Foreign Assets Control (Treasury) |
| **PGA** | Partner Government Agency |
| **POD** | Proof of Delivery |
| **POA** | Power of Attorney |
| **SDN** | Specially Designated Nationals (OFAC sanctions list) |
| **STB** | Single Transaction Bond |
| **TIB** | Temporary Importation under Bond |
| **TSCA** | Toxic Substances Control Act (EPA) |
| **UCC** | Union Customs Code (EU) |
| **UFLPA** | Uyghur Forced Labor Prevention Act |
| **ULD** | Unit Load Device (air cargo container/pallet) |
| **USMCA** | United States-Mexico-Canada Agreement |
| **VACIS** | Vehicle and Cargo Inspection System (large-scale X-ray) |
| **WCO** | World Customs Organization |

---

## Appendix B: Data Entity Relationship Diagram (Textual)

```
+-------------------+       +--------------------+       +-------------------+
|    SHIPMENT       |1    N |  SHIPMENT_PARTY    |       |  CUSTOMS_ENTRY    |
|-------------------|-------|--------------------| N   1 |-------------------|
| tracking_number   |       | party_role         |-------|  entry_number     |
| service_type      |       | party_name         |       |  entry_type       |
| transport_mode    |       | party_address      |       |  entry_date       |
| incoterms         |       | party_id           |       |  port_of_entry    |
| status            |       | dps_screened       |       |  status           |
| est_delivery_date |       | dps_result         |       |  release_date     |
+--------+----------+       +--------------------+       |  liquidation_date |
         |                                                |  total_duty       |
         | 1                                              |  fta_claimed      |
         |                                                +--------+----------+
         | N                                                       |
+--------+----------+                                              | 1
| SHIPMENT_LINE_ITEM|                                              |
|-------------------|                                              | N
| line_number       |                                    +--------+----------+
| product_desc      |                                    | DUTY_LINE_ITEM   |
| hs_code           |                                    |-------------------|
| country_of_origin |                                    | program           |
| declared_value    |                                    | rate_type         |
| quantity          |                                    | rate              |
| unit_of_measure   |                                    | dutiable_value    |
| gross_weight_kg   |                                    | calculated_amount |
+--------+----------+                                    | legal_citation    |
         |                                                +-------------------+
         | 1
         |
         | N
+--------+----------+       +--------------------+       +-------------------+
| TRACKING_EVENT    |       |    DOCUMENT        |       | COMPLIANCE_SCREEN |
|-------------------|       |--------------------|       |-------------------|
| event_timestamp   |       | document_type      |       | screening_type    |
| event_code        |       | document_number    |       | screened_party    |
| event_description |       | document_date      |       | list_source       |
| location_code     |       | file_reference     |       | match_score       |
| scan_type         |       | verified           |       | disposition       |
| additional_data   |       | expiry_date        |       | reviewed_by       |
+-------------------+       +--------------------+       +-------------------+
```

---

## Appendix C: Mapping to Current Codebase

The following maps this lifecycle reference to existing entities in the Clearance Intelligence Engine.

| Lifecycle Concept | Current Codebase Entity | File Path |
|---|---|---|
| Shipment (basic) | `demo_shipments` table | `/clearance-engine/backend/alembic/versions/001_initial_schema.py` |
| HS Classification | `htsus_chapters`, `htsus_headings` | `/clearance-engine/backend/app/knowledge/models/tariff.py` |
| Section 301 | `section_301_lists` | `/clearance-engine/backend/app/knowledge/models/tariff.py` |
| Section 232 | `section_232_scope` | `/clearance-engine/backend/app/knowledge/models/tariff.py` |
| IEEPA | `ieepa_rates` | `/clearance-engine/backend/app/knowledge/models/tariff.py` |
| AD/CVD | `adcvd_orders` | `/clearance-engine/backend/app/knowledge/models/tariff.py` |
| Denied Party Screening | `restricted_parties` | `/clearance-engine/backend/app/knowledge/models/compliance.py` |
| PGA Requirements | `pga_mappings` | `/clearance-engine/backend/app/knowledge/models/compliance.py` |
| FTA Rules of Origin | `fta_rules` | `/clearance-engine/backend/app/knowledge/models/compliance.py` |
| CBP Rulings | `cross_rulings` | `/clearance-engine/backend/app/knowledge/models/reference.py` |
| Tax Rates | `tax_rates` | `/clearance-engine/backend/app/knowledge/models/reference.py` |
| Fee Schedule (MPF/HMF) | `fee_schedule` | `/clearance-engine/backend/app/knowledge/models/reference.py` |
| Exchange Rates | `exchange_rates` | `/clearance-engine/backend/app/knowledge/models/reference.py` |
| Tax Regime Templates | `tax_regime_templates` | `/clearance-engine/backend/app/knowledge/models/reference.py` |
| Regulatory Signals | `regulatory_signals` | `/clearance-engine/backend/alembic/versions/001_initial_schema.py` |
| Tariff Engine | `TariffEngine` (E2) | `/clearance-engine/backend/app/engines/e2_tariff/engine.py` |
| Classification Engine | Engine E1 | `/clearance-engine/backend/app/engines/e1_classification/engine.py` |
| Compliance Engine | Engine E3 | `/clearance-engine/backend/app/engines/e3_compliance/engine.py` |
| FTA Engine | Engine E4 | `/clearance-engine/backend/app/engines/e4_fta/engine.py` |
| Exception Engine | Engine E5 | `/clearance-engine/backend/app/engines/e5_exception/engine.py` |
| Regulatory Engine | Engine E6 | `/clearance-engine/backend/app/engines/e6_regulatory/engine.py` |
| Landed Cost Formatting | `format_platform_detail`, `format_shipper_summary`, `format_buyer_price` | `/clearance-engine/backend/app/engines/e2_tariff/landed_cost.py` |

**Gaps between lifecycle reference and current codebase** (entities defined in this document but not yet in the schema):
- Full `Shipment` entity with lifecycle states (currently only `demo_shipments` with basic fields)
- `ShipmentParty` entity with multi-role support and DPS tracking
- `ShipmentLineItem` entity (currently line items are flat within demo_shipments)
- `CustomsEntry` entity with full entry lifecycle (filing, release, liquidation, protest)
- `DutyLineItem` entity with per-program duty breakdown
- `TrackingEvent` entity (event sourcing for shipment lifecycle)
- `Document` entity with document management
- `ComplianceScreeningResult` entity (partially implemented in demo_shipments.compliance_flags)

---

*This document is part of the FedEx Global Clearance Knowledge Base. For related context, see:*
- *[01: Regulatory Landscape](01_regulatory_landscape.md)*
- *[02: Pain Points Exhaustive](02_pain_points_exhaustive.md)*
- *[04: Clearance Process Today](04_clearance_process_today.md)*
- *[06: Supply Chain Traceability from Clearance Data](06_supply_chain_traceability_from_clearance_data.md)*
