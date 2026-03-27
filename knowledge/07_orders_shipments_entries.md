# Orders, Shipments, and Customs Entries: Entity Relationships and Lifecycles

> Clearance Intelligence Engine -- Operational Knowledge Base
> Last Updated: February 2026

## 1. Purchase Order (PO) to Shipment Relationship

### The Many-to-Many Reality

The relationship between purchase orders and shipments is fundamentally **many-to-many**. This is one of the most critical data modeling facts for any clearance software platform.

**One PO can span multiple shipments (partial shipments):**
- A partial shipment occurs when part of a customer's order is shipped first, and the rest follows later. The order is divided into multiple deliveries instead of being sent all at once. This happens when some products are out of stock, stored in different warehouses, or take longer to manufacture.
- Each partial shipment may have its own tracking number, its own bill of lading, and its own delivery schedule.
- Partial shipments are allowed under most shipping and trade terms, unless specifically restricted by the purchase contract or letter of credit. A letter of credit may explicitly prohibit partial shipments with a clause like "partial shipments not allowed."
- Software implication: A single PO line item can be split across multiple shipments. You need a junction entity (e.g., `shipment_line` or `shipment_item`) that references both the PO line and the shipment, tracking quantities fulfilled per shipment.

**Multiple POs can consolidate into a single shipment:**
- **LCL (Less-than-Container Load) consolidation**: Multiple shipments from different shippers to different consignees are combined into one container based on shared destination. A freight forwarder or NVOCC collects goods from multiple customers and loads them into one container. Each shipper gets a House Bill of Lading (HBL); the container moves under a single Master Bill of Lading (MBL).
- **Buyer's consolidation**: When a single consignee purchases from multiple suppliers in the same origin country, a freight forwarder consolidates those separate POs into a single FCL container. Each PO retains its own HBL, but the entire container ships under one MBL. This is distinctly different from LCL -- all goods belong to the same buyer.
- Software implication: A shipment (identified by a bill of lading) can contain goods from multiple POs. The system must track which PO lines map to which shipment lines, with quantities.

### POs and Commercial Invoices

The document chain follows a strict sequence:

1. **Proforma Invoice** -- Issued by the seller before a sale is confirmed. It is a preliminary bill or quote containing an estimate of charges, product descriptions, quantities, and terms. It is NOT legally binding. It serves as the basis for the buyer to arrange financing, obtain import licenses, or open a letter of credit. Financial institutions will not issue an LC based on a simple email -- they need the proforma.

2. **Purchase Order (PO)** -- Issued by the buyer after accepting the proforma invoice terms. The PO is the buyer's formal commitment to buy. It mirrors the proforma's details but is the buyer's document.

3. **Commercial Invoice** -- Issued by the seller when goods are ready to ship or have shipped. This IS a legally binding document. It is the primary document used by customs authorities worldwide for commodity control and valuation. It contains goods info, value, Incoterms, payment terms, and HS codes. Discrepancies between the commercial invoice and the PO or LC will cause customs delays and bank payment refusals.

Key fact: The final quantity on the commercial invoice can differ from the proforma/PO (e.g., the factory produced 980 units instead of 1,000). Consistency across all trade documents is paramount -- any mismatch triggers customs inspection and LC payment refusal.

One PO may result in multiple commercial invoices (partial shipments). Multiple POs to the same seller may be covered by a single commercial invoice. The commercial invoice is what customs actually uses; the PO is a supporting document.

### Letters of Credit (LC)

An LC is a payment mechanism where a bank guarantees payment to the seller, provided the seller presents compliant documents. The LC lifecycle interleaves with the order lifecycle:

1. Buyer and seller agree on terms (including LC as payment method)
2. Buyer applies to issuing bank for LC
3. Issuing bank sends LC to advising bank (seller's bank)
4. Seller ships goods and gathers required documents (commercial invoice, bill of lading, packing list, certificate of origin, insurance certificate)
5. Seller presents documents to negotiating bank
6. Bank examines documents for strict compliance with LC terms
7. If compliant, payment is released to seller

Critical software implication: Banks deal in documents only, not goods. The LC is a separate contract from the sales contract. The system should track LC numbers, LC amounts, expiry dates, and document presentation status.

---

## 2. Shipment to Customs Entry Relationship

### The General Rule

Under U.S. customs regulations (19 CFR Part 141), all merchandise arriving on one conveyance and consigned to one consignee must be included on one entry. The default is: **one shipment to one consignee = one customs entry**.

### When One Shipment Has Multiple Entries

The CBP Center Director may permit separate entries for different portions of a shipment arriving on one vessel when:
- A consolidated shipment addressed to one nominal consignee (e.g., a freight forwarder) contains goods for multiple ultimate consignees -- each ultimate consignee's portion may be entered separately.
- Different portions require different entry types (e.g., some goods are for consumption, others are for a Foreign Trade Zone, others are in-bond for re-export).
- Different importers of record exist for different portions of the cargo.
- In LCL shipments: Each shipper's portion under separate HBLs may be entered individually, even though they share the same MBL and container. The freight forwarder (as nominal consignee on the MBL) does not have the right to make entry as owner/purchaser but can designate a broker.

### When One Entry Covers Multiple Shipments

This is rarer but explicitly provided for:
- **Split shipments** (19 CFR 141.57): A single shipment split by the carrier across multiple conveyances arriving at the same port may be processed under a single entry. The importer submits a copy of CF 3461 adjusted for each arriving portion.
- **Consolidated entry summaries**: The CBP Center Director may permit one entry summary for merchandise from separate entries if: same country of exportation, same country of origin, same vessel or carrier, same consignee, entries span no more than 1 week, and entry summary filed within 10 working days of the first entry.

### Container-Level vs. Entry-Level Tracking

This is a critical architectural decision for software:

| Approach | Pros | Cons |
|----------|------|------|
| One entry per multiple containers (same BOL) | Lower MPF fees (capped at $651.50 per entry), lower broker fees | One customs hold delays ALL containers; less granular tracking |
| Separate entries per container (separate BOLs) | Only affected container is delayed on hold/exam; granular tracking | Higher MPF fees, higher broker fees, more paperwork |

CBP operates holds and exams at both entry level and bill-of-lading level. A manifest hold on a bill associated with a specific entry suspends release for that entry. Experienced importers often recommend separate BOLs per container when inspection risk is high.

Software implication: The data model needs a `customs_entry` entity that has a many-to-many relationship with both `shipment` and `container`. An entry can reference multiple containers/BOLs, and (rarely) a single BOL's cargo can generate multiple entries.

### ISF (Importer Security Filing) -- The "10+2"

For ocean freight into the U.S., the ISF must be filed at least 24 hours before cargo is loaded onto the vessel at the foreign port. It contains 10 data elements from the importer and 2 from the carrier. The ISF is linked to the customs entry through the bill of lading number -- that is the only common identifier. ISF is NOT the entry itself, but CBP compares ISF data to entry data for risk analysis.

ISF does not apply to air freight. Bulk and certain breakbulk cargoes are exempt.

---

## 3. Order Lifecycle in International Trade

### The Full Lifecycle

```
DRAFT --> CONFIRMED --> PRODUCTION --> BOOKED --> SHIPPED --> IN TRANSIT --> AT PORT --> CUSTOMS --> DELIVERED
```

Detailed stages:

1. **Draft / Negotiation**: Buyer and seller negotiate terms. Buyer may request a proforma invoice. Incoterms are agreed upon (FOB, CIF, DDP, etc.), defining who pays for freight, insurance, customs, and at what point risk transfers.

2. **Confirmed / PO Issued**: Buyer issues a formal purchase order. If payment is by LC, the buyer applies to their bank. Seller may wait for LC confirmation before starting production.

3. **Production / Manufacturing**: Seller manufactures or sources goods. Pre-shipment inspections may occur. This stage may take weeks to months.

4. **Booked**: The responsible party (per Incoterms) contacts a freight forwarder, who provides a quote. Accepting the quote creates a booking -- a contract for freight services. The forwarder books space with the carrier (ocean line or airline). A booking confirmation is issued.

5. **Shipped / Export Clearance**: Goods are loaded onto the vessel or aircraft. Export customs clearance occurs at origin. Electronic Export Information (EEI) is filed through AES (for U.S. exports). The carrier issues the bill of lading or air waybill. For U.S. ocean imports, ISF must have been filed at least 24 hours before loading.

6. **In Transit**: Cargo is on the water or in the air. Ocean transit typically takes 2-6 weeks depending on route. Tracking is available via AIS (Automatic Identification System) for vessels, or carrier portals.

7. **At Port / Arrived**: Vessel arrives at destination port. Containers are discharged. This does NOT mean goods are available -- several processes follow.

8. **Customs Clearance**: Entry documents (CF 3461) are filed. CBP determines admissibility. Status progresses through: On File -> Admissible -> Released (or Hold/Document Review/Intensive Exam). Entry summary (CF 7501) must be filed within 10 working days of release, with estimated duties paid.

9. **Delivered / Final Mile**: After customs release, goods move via drayage/trucking to the warehouse or final destination. LCL cargo goes to a CFS (Container Freight Station) for deconsolidation first.

### "Clearance Order" vs. "Purchase Order"

These are distinct concepts:
- A **purchase order** is a commercial document from buyer to seller, governing the sale of goods.
- A **clearance order** (or customs release order) is a government-issued authorization from customs authorities permitting goods to cross the border. It is the output of the customs clearance process, not a commercial transaction.
- In customs brokerage software, a "clearance order" or "clearance job" often refers to the broker's internal work order -- the set of tasks and documents needed to clear a specific shipment through customs. It is created when a broker receives instructions from an importer to clear incoming goods.

### How Freight Forwarders Create Bookings from Orders

The forwarder receives shipping instructions (often derived from the PO and proforma invoice). They determine the best routing, mode (ocean/air/multimodal), and carrier. They submit a booking request to the carrier. The carrier confirms space. The forwarder issues a booking confirmation to the shipper. A single quote may include multiple bookings for separate freight services (main carriage, trucking, insurance, customs brokerage) -- all part of the same shipment.

---

## 4. Key Parties and Their Relationships

### Party Definitions

**Shipper/Exporter**: The party that sells and ships the goods. Listed as "shipper" on the bill of lading. Often the manufacturer or trading company in the origin country.

**Consignee**: The party to whom the goods are being delivered, as listed on the bill of lading or air waybill. They are the recipient. They are NOT always legally responsible for customs compliance -- that falls to the IOR.

**Importer of Record (IOR)**: The legally liable party responsible for ensuring imported goods comply with all local laws and regulations. They must classify and value goods, file import declarations, pay customs duties/taxes, and acquire necessary permits. Identified to CBP via EIN, SSN, or CBP-assigned importer number. Must maintain records for 5 years.

**Customs Broker**: A licensed intermediary who files entry documents with CBP on behalf of the importer. They act as an agent, NOT as the IOR (except in rare cases). They handle classification, valuation, entry filing, and compliance.

**Freight Forwarder**: Arranges transportation of goods from origin to destination. Acts as intermediary between shippers and carriers. May operate as an NVOCC (Non-Vessel Operating Common Carrier) and issue their own bills of lading.

**Notify Party**: The individual or entity that receives arrival notification from the carrier. Their role is purely informational -- they have no legal claim to the cargo. Often the customs broker or freight forwarder, to facilitate prompt clearance. In LC transactions where a bank is listed as consignee, the notify party is typically the actual buyer.

### When IOR Differs from Consignee

Common scenarios:
- A U.S. retailer (IOR) directs delivery to a third-party warehouse (consignee).
- A foreign seller ships DDP (Delivered Duty Paid) and acts as IOR while the buyer is the consignee.
- A company imports goods through a third-party IOR service provider when they lack a U.S. entity.
- In LC transactions, the bank may be the consignee while the buyer is the IOR.

### Ultimate Consignee vs. Intermediate Consignee

**Ultimate Consignee**: The person or entity located abroad (for exports) who actually receives the export shipment as the final recipient -- for use, resale, or distribution. Per FTR Section 30.1, the ultimate consignee is NOT a foreign forwarding agent or intermediate consignee. They can be classified as: Direct Consumer, Government Entity, Reseller, or Other/Unknown.

**Intermediate Consignee**: An entity located abroad that temporarily takes physical possession of goods to facilitate delivery to the ultimate consignee. They act as an agent -- commonly a freight forwarder, customs broker in a transit country, or a distributor. If a distributor adds significant value or alters the goods, they may become the ultimate consignee instead. Reported in AES filing as a conditional field.

For U.S. customs entry, the ultimate consignee number must be reported on the entry summary (CF 7501).

---

## 5. Real-World Shipment Structures

### Ocean Freight: FCL vs. LCL

**FCL (Full Container Load)**:
- One shipper uses an entire container.
- Goods are loaded at origin and remain sealed until destination.
- One MBL typically issued by the carrier to the forwarder.
- The forwarder may issue one or more HBLs (e.g., if buyer consolidation).
- Standard container sizes: 20ft (TEU), 40ft, 40ft HC, 45ft.

**LCL (Less-than-Container Load)**:
- Multiple shippers share a container.
- Goods from different shippers are consolidated at a CFS (Container Freight Station) at origin.
- At destination, the container is deconsolidated at a CFS.
- One MBL covers the full container (consigned to the NVOCC/forwarder).
- Multiple HBLs issued -- one per shipper/consignment.
- Cost calculated primarily by volume (CBM); transit times are slightly longer than FCL due to consolidation/deconsolidation.
- Higher per-unit cost than FCL; more handling = higher damage risk.

**Buyer's Consolidation** (a hybrid):
- Multiple POs from the same buyer but different suppliers are consolidated into one FCL.
- Each supplier gets its own HBL; one MBL covers the container.
- Only needs deconsolidation at the final destination (not at an intermediate CFS).
- Customs entry may be filed under the single MBL.

### Ocean Freight Documentation: MBL / HBL

```
Shipping Line (Carrier)
    |
    +-- Issues MBL (Master Bill of Lading)
            |
            +-- To: Freight Forwarder / NVOCC
                    |
                    +-- Issues HBL(s) (House Bill of Lading)
                            |
                            +-- To: Individual Shipper(s)
```

- The MBL is a contract of carriage between the carrier and the forwarder. It contains voyage number, vessel name, ports of loading/discharge.
- The HBL is a contract between the forwarder and the shipper. It serves as a receipt and contract for carriage but is issued by the forwarder, not the carrier.
- MBL is negotiable (serves as document of title). HBL is generally non-negotiable.
- Multiple HBLs can exist under a single MBL.
- At destination, the HBL holder must exchange it through the forwarder's agent to obtain release against the MBL.

### Air Freight Documentation: MAWB / HAWB

```
Airline (Carrier)
    |
    +-- Issues MAWB (Master Air Waybill)
            |
            +-- To: Freight Forwarder
                    |
                    +-- Issues HAWB(s) (House Air Waybill)
                            |
                            +-- To: Individual Shipper(s)
```

- The MAWB is issued by the airline or its agent. First three digits of the MAWB number identify the airline.
- The MAWB consignee is typically the forwarder's destination office, NOT the final importer.
- The HAWB is issued by the freight forwarder to the individual shipper. The HAWB consignee IS the real importer/buyer.
- Multiple HAWBs exist under a single MAWB in consolidated shipments.
- Unlike ocean bills, air waybills are ALWAYS non-negotiable and do NOT serve as documents of title.

### Break-Bulk Cargo

- Goods that cannot fit inside a standard container due to size, shape, or weight.
- Includes heavy machinery, vehicles, infrastructure materials, odd-shaped cargo.
- Recorded on distinct bills of lading listing different products.
- Volume has declined since the 1960s with mass container adoption, but remains essential for project cargo and oversized items.

### Multimodal Transport

- A multimodal transport document covers goods moving across multiple transportation modes (sea, rail, road, air) under a single contract.
- Issued by a Multimodal Transport Operator (MTO) -- typically a shipping line, freight forwarder, or NVOCC.
- The MTO takes responsibility for the goods during the entire transport period, across all modes.
- Distinct from intermodal: multimodal = one contract with one carrier for the entire journey; intermodal = multiple contracts with different carriers for different legs.
- Liability follows the "network liability system" -- the rules governing the specific mode where loss/damage occurred apply.

---

## 6. Status Transitions That Matter for Software

### Order Statuses (Purchase/Commercial Order)

| Status | Description |
|--------|-------------|
| `DRAFT` | Order being prepared; terms under negotiation |
| `PROFORMA_SENT` | Proforma invoice sent to buyer |
| `CONFIRMED` | PO issued and accepted by seller |
| `LC_OPENED` | Letter of credit opened (if applicable) |
| `IN_PRODUCTION` | Goods being manufactured |
| `READY_TO_SHIP` | Production complete; awaiting booking |
| `BOOKED` | Freight booked with carrier |
| `PARTIALLY_SHIPPED` | Some PO lines shipped; others pending |
| `FULLY_SHIPPED` | All PO lines have shipped |
| `DELIVERED` | All goods received at destination |
| `CLOSED` | Order complete; all financial obligations settled |
| `CANCELLED` | Order cancelled |

### Shipment Statuses

| Status | Description |
|--------|-------------|
| `BOOKING_REQUESTED` | Booking submitted to carrier |
| `BOOKING_CONFIRMED` | Carrier confirmed space allocation |
| `ISF_FILED` | Importer Security Filing submitted (ocean only) |
| `CARGO_RECEIVED` | Goods received at origin warehouse/CFS |
| `EXPORT_CLEARED` | Export customs clearance completed |
| `LOADED` | Cargo loaded onto vessel/aircraft |
| `DEPARTED` | Vessel/aircraft departed origin |
| `IN_TRANSIT` | En route to destination; may include transshipments |
| `TRANSSHIPMENT` | Cargo being transferred between vessels at an intermediate port |
| `ARRIVED_AT_PORT` | Vessel/aircraft arrived at destination port |
| `DISCHARGED` | Container/cargo unloaded from vessel |
| `AT_CFS` | At Container Freight Station for deconsolidation (LCL) |
| `CUSTOMS_HOLD` | Held pending customs clearance |
| `CUSTOMS_RELEASED` | Cleared by customs |
| `OUT_FOR_DELIVERY` | En route to final destination via drayage/trucking |
| `DELIVERED` | Goods received at final destination |
| `DEMURRAGE` | Container at port beyond free time (fees accruing) |
| `DETENTION` | Container at consignee beyond free time (fees accruing) |

### Customs Entry Statuses (per CBP ACE)

| Status | Description |
|--------|-------------|
| `NOT_FILED` | Entry not yet submitted to CBP |
| `ON_FILE` | Cargo release entry on file; pending carrier bill match |
| `ADMISSIBLE` | Bill match occurred; no holds; awaiting release window |
| `DOCUMENT_REVIEW` | CBP requests document review before release |
| `INTENSIVE_EXAM` | Physical examination required |
| `MANIFEST_HOLD` | Manifest-level hold on associated bill |
| `RELEASED` | Entry released; all holds removed |
| `CANCELLED` | Entry cancelled; entry number cannot be reused |
| `ENTRY_SUMMARY_FILED` | CBP Form 7501 filed; estimated duties deposited |
| `PSC_FILED` | Post-Summary Correction submitted (before liquidation) |
| `LIQUIDATED` | CBP finalized duty assessment (~314 days from entry) |
| `LIQUIDATED_NO_CHANGE` | Final duty = estimated duty |
| `LIQUIDATED_INCREASE` | Actual duty > estimated; bill issued |
| `LIQUIDATED_DECREASE` | Actual duty < estimated; refund issued |
| `PROTESTED` | Importer filed 19 USC 1514 protest (within 180 days of liquidation) |
| `RELIQUIDATED` | Entry reopened and duties reassessed after protest granted |

### How the Three Lifecycles Interconnect

```
ORDER LIFECYCLE                  SHIPMENT LIFECYCLE               ENTRY LIFECYCLE
==============                   ==================               ===============

DRAFT
PROFORMA_SENT
CONFIRMED
LC_OPENED
IN_PRODUCTION
READY_TO_SHIP --------+
BOOKED               |
                      +-------> BOOKING_CONFIRMED
                                ISF_FILED
                                CARGO_RECEIVED
PARTIALLY_SHIPPED <------------ EXPORT_CLEARED
                                LOADED
                                DEPARTED
                                IN_TRANSIT
                                ARRIVED_AT_PORT
                                DISCHARGED --------+
                                                   +-----------> ON_FILE
                                                                 ADMISSIBLE
                                CUSTOMS_HOLD <------------------- INTENSIVE_EXAM / HOLD
                                CUSTOMS_RELEASED <--------------- RELEASED
                                                                 ENTRY_SUMMARY_FILED
                                OUT_FOR_DELIVERY
FULLY_SHIPPED                   DELIVERED
DELIVERED
                                                                 (months later...)
                                                                 LIQUIDATED
CLOSED                                                           PROTESTED (if needed)
                                                                 RELIQUIDATED (if needed)
```

Key interconnection rules:
- An **order** transitions to `PARTIALLY_SHIPPED` when its first shipment departs, and `FULLY_SHIPPED` when all shipments covering all PO lines have departed.
- A **shipment** triggers creation of an entry when it arrives at the destination port (or earlier for pre-clearance).
- A **customs entry** status of `RELEASED` triggers the shipment to move to `CUSTOMS_RELEASED`.
- An **order** is not `CLOSED` until all entries are liquidated and all financial obligations (duties, fees) are settled -- which can be up to 314+ days after arrival.
- The order lifecycle is the longest (months to over a year from draft to close). The shipment lifecycle is weeks to months. The entry lifecycle extends from arrival through liquidation (up to 314 days + protest period).

### The Many-to-Many Summary

```
PURCHASE_ORDER ---< PO_LINE >--- (many-to-many) ---< SHIPMENT_LINE >--- SHIPMENT
                                                                            |
                                                                    (many-to-many)
                                                                            |
                                                                     CUSTOMS_ENTRY
                                                                            |
                                                                    (one-to-many)
                                                                            |
                                                                     ENTRY_LINE
```

- One PO can produce multiple shipments (partial shipments).
- One shipment can contain goods from multiple POs (consolidation).
- One shipment typically produces one customs entry, but can produce multiple (different IORs, different entry types).
- One customs entry typically covers one shipment, but can cover multiple (split shipments, consolidated entry summaries).
- Each customs entry has multiple entry lines, each with its own HTS classification, value, duty rate, and country of origin.

---

## Sources

- [DHL - Less than Container Load (LCL)](https://www.dhl.com/us-en/home/global-forwarding/products-and-solutions/ocean-freight/less-than-container-load.html)
- [InterlogUSA - Buyer Consolidation](https://www.interlogusa.com/answers/blog/domestic-reduce-shipping-spend-buyer-consolidation/)
- [SHIPIT Logistics - Buyer's Consolidation](https://www.shipit.com/buyers-consolidation)
- [Freightos - LCL Shipping Guide](https://www.freightos.com/freight-resources/what-is-lcl-shipping-the-complete-guide/)
- [USA Customs Clearance - IOR vs Consignee](https://usacustomsclearance.com/process/importer-of-record-vs-consignee/)
- [eCFR 19 CFR Part 141 - Entry of Merchandise](https://www.ecfr.gov/current/title-19/chapter-I/part-141)
- [eCFR 19 CFR Part 142 - Entry Process](https://www.ecfr.gov/current/title-19/chapter-I/part-142)
- [CBP Customs Directive 3530-002A](https://www.cbp.gov/sites/default/files/documents/3530-002a_3.pdf)
- [Ship4wd - MBL vs HBL](https://ship4wd.com/logistics-shipping/mbl-vs-hbl)
- [Ship4wd - MAWB vs HAWB](https://ship4wd.com/import-guides/mawb-vs-hawb)
- [Shipping Solutions - Proforma vs Commercial Invoice](https://shippingsolutionssoftware.com/blog/proforma-vs-commercial-invoice)
- [Trade.gov - Letter of Credit](https://www.trade.gov/letter-credit)
- [Drip Capital - Letter of Credit](https://www.dripcapital.com/resources/blog/letter-of-credit-lc)
- [CBP - Importer Security Filing 10+2](https://www.cbp.gov/border-security/ports-entry/cargo-security/importer-security-filing-102)
- [CBP - Liquidation in ACE](https://www.cbp.gov/trade/automated/news/liquidation)
- [CBP - Entry Summary and Post Release](https://www.cbp.gov/trade/programs-administration/entry-summary)
- [CBP - ACE Transaction Details](https://www.cbp.gov/trade/automated/ace-transaction-details)
- [GHY International - US Customs Entry Process & Lifecycle](https://www.ghy.com/trade-talk/us-customs-entry-process-lifecycle/)
- [CBP - Form 7501 Entry Summary](https://www.cbp.gov/trade/programs-administration/entry-summary/cbp-form-7501)
- [CBP - ACE Protest FAQ](https://www.cbp.gov/trade/programs-administration/entry-summary/protests/ace-protest-frequently-asked-questions)
- [CustomsCity - Entry Type Differences](https://customscity.com/understanding-the-differences-between-type-01-type-11-and-type-86-customs-entries-when-to-use-each/)
- [Approved Forwarders - Freight Shipping Guide](https://www.approvedforwarders.com/your-international-freight-shipping-guide-step-by-step/)
- [Freightos - Key Freight Documents](https://www.freightos.com/freight-resources/key-freight-documents/)
- [Census.gov - Ultimate Consignee Part 1](https://www.census.gov/newsroom/blogs/global-reach/2023/09/ultimate-consignee-part-1.html)
- [Census.gov - Ultimate Consignee Part 2](https://www.census.gov/newsroom/blogs/global-reach/2023/10/ultimate-consignee-part-2.html)
- [CrimsonLogic - Intermediate vs Ultimate Consignee](https://crimsonlogic-northamerica.com/docs/aes-003-what-is-the-difference-between-an-intermediate-consignee-and-an-ultimate-consignee/)
- [Shipsgo - Container Milestones](https://blog.shipsgo.com/container-milestones/)
- [19 CFR 141.57 - Single Entry for Split Shipments](https://www.law.cornell.edu/cfr/text/19/141.57)
- [ICE Transport - Customs Clearance Costs](https://www.icetransport.com/blog/how-much-does-customs-clearance-cost)
- [Shapiro - ISF 10+2](https://www.shapiro.com/resources/what-you-need-to-know-about-importer-security-filing-isf-102/)
- [Maersk - Break Bulk Cargo](https://www.maersk.com/logistics-explained/transportation-and-freight/2024/09/18/break-bulk-cargo)
- [PackageX - Partial Shipments](https://packagex.io/blog/what-is-partial-shipment)