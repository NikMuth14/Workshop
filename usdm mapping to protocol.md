USDM Node → Protocol Section Mapping
Template: Pfizer CT02‑GSOP Oncology Clinical Protocol Template (01 April 2022).
Scope: the 23 USDM nodes listed in your skills table. All are children of study.versions[0] unless otherwise noted.
How to read each entry

Source (primary): the section/table/named block in the protocol that is the authoritative source for this node. Always check this first.
Source (fallback / supplementary): other places where the same data is restated and can be used to verify or fill gaps.
What to extract: the literal pieces of content the skill should pull.
Target USDM fields: the JSON shape produced.
Anchor / regex hints: the textual landmarks the skill can use to find the section reliably even when page numbers shift.
Edge cases / gotchas: known pitfalls.

The synopsis (§1.1) is an aggregator that restates many fields. Always treat §1.1 as a fallback, never as primary — fields in the body of the document are authoritative.

1. usdm-versionIdentifier

Source (primary): Title page (page 1) — the line directly under the protocol title, formatted "Final Protocol Amendment <N>, <DD Month YYYY>" or "Original Protocol, <DD Month YYYY>".
Source (fallback): running header on every page (Final Protocol Amendment 3, 17 November 2025), and §1.1 Synopsis → Rationale for Protocol Amendment <N>.
What to extract: the amendment label string only (e.g., "Amendment 3" or "Original Protocol").
Target USDM fields: study.versions[0].versionIdentifier (string).
Anchor / regex hints: ^(Final Protocol Amendment \d+|Original Protocol),\s+\d+\s+\w+\s+\d{4}$.
Edge cases: title page sometimes shows "Final" vs "Draft"; only Final should be ingested. Do not include the date here — the date belongs in dateValues.


2. usdm-rationale

Source (primary): §4.2 Scientific Rationale for Study Design (full prose).
Source (fallback): §1.1 Synopsis → "Rationale:" sub‑block (1‑paragraph version of the same content). Also §1.1 Synopsis → "Rationale for Protocol Amendment <N>:" for the amendment rationale (this is distinct content — see Note below).
What to extract: the full prose rationale for the study design.
Target USDM fields: study.versions[0].rationale (string, plain text or limited HTML).
Anchor / regex hints: heading 4.2. Scientific Rationale for Study Design; in synopsis use the bold label Rationale: followed by paragraph(s) up to the next bold label.
Edge cases / gotchas: Do NOT confuse with the amendment rationale. The amendment rationale ("Rationale for Protocol Amendment N:" in §1.1) belongs on amendments[i].summary, not on versions[0].rationale. Two different fields, two different sources.


3. usdm-documentVersionIds

Source (primary): Document History table (immediately before the TOC, untitled or titled "Document History" / "Amendment History"). This table lists every prior version: Original Protocol, Amendment 1, Amendment 2, Amendment 3, etc., each with its date.
Source (fallback): §10.14 Appendix 14: Protocol Amendment History lists every amendment with its full change log.
What to extract: one identifier per prior document version. The values are references to StudyDefinitionDocumentVersion objects that the skill must also build.
Target USDM fields: study.versions[0].documentVersionIds (string array). Should reference the id of each StudyDefinitionDocumentVersion under study.documentedBy.versions.
Anchor / regex hints: the table directly above TABLE OF CONTENTS. Rows formatted as Original Protocol, Amendment 1, Amendment 2, … each with <DD Month YYYY>.
Edge cases: the values must be IDs (e.g., StudyDefinitionDocumentVersion_1), not free‑text labels. The current generator is incorrectly putting strings here ("Amendment 3", "Amendment 2", etc.) — that must be fixed in the skill.


4. usdm-dateValues

Source (primary, multi‑location):

Title page → effective date of current amendment (next to the version identifier).
Footer of every page → "Approved On: <DD-Mon-YYYY HH:MM (GMT)>" → sponsor approval timestamp.
Document History table (before TOC) → effective date of every prior amendment + original protocol date.
§4.4 End of Study Definition → planned study end date / definition (for Primary Completion Date and Study Completion Date).
§1.1 Synopsis → "Estimated Study Duration" if present, gives planned start / FPI / LPLV.


What to extract: (name, type, dateValue) triples for each date.
Target USDM fields: study.versions[0].dateValues[]. Each GovernanceDate carries name, type (CDISC code e.g. Protocol Effective Date, Approval Date, Primary Completion Date), and dateValue (ISO‑8601).
Anchor / regex hints:

Approval date: Approved On:\s+(\d{2}-\w{3}-\d{4}).
Amendment effective dates: rows of the Document History table.
Original protocol date: row labeled Original Protocol in Document History.


Edge cases: the footer Approved On appears on every page — pull only once. Some protocols give date as <DD Month YYYY>, others as <DD-Mon-YYYY> — normalize to ISO.


5. usdm-amendments

Source (primary): §10.14 Appendix 14: Protocol Amendment History. This appendix contains a separate sub‑section for each amendment (Amendment 1, Amendment 2, Amendment 3, …), each with its own structured change table containing columns like Section / Section Title / Description of Change / Brief Rationale.
Source (fallback for the current amendment only):

Document History table (before TOC) — gives the change‑summary table for the latest amendment with columns Document Section(s) Affected, Description of Change, Brief Rationale.
§1.1 Synopsis → "Rationale for Protocol Amendment <N>:" — short prose summary of the current amendment.


What to extract for each amendment:

number, summary (prose rationale), effectiveDate
studyChanges[] — one row per change‑table row (sectionNumber, sectionTitle, description, rationale → maps to afterText)
studyAmendmentImpacts[] — derived from rationale text (manual mapping or LLM extraction); affected section + impact level
studyAmendmentReasons[] — categorize each change rationale into Operational / Safety / Scientific / Regulatory / Other
enrollmentStatus — usually inferable from amendment text ("enrollment was completed", "Closed", etc.)
primaryReason — the dominant reason category from the rationale paragraph


Target USDM fields: study.versions[0].amendments[] (one per amendment).
Anchor / regex hints: sub‑headings in §10.14 like Document History / Amendment <N> / Document Section(s) Affected.
Edge cases / gotchas:

The Document History table preceding the TOC is the summary of only the latest amendment. For amendments 1, 2 (and earlier), you must go to §10.14.
primaryReason.code must be a real CDISC code, not the placeholder C99999 the current generator emits.
Amendment numbers are strings in USDM (e.g., "3"), not integers.




6. usdm-businessTherapeuticAreas

Source (primary): Title page — typically the running title contains the disease (e.g., "ER‑Positive, HER2‑Negative Breast Cancer").
Source (fallback): §1.1 Synopsis → "Indication" sub‑block, and §2.2.1 Breast Cancer (or equivalent disease‑background subsection of §2.2).
What to extract: therapeutic area at the business level (Oncology, Cardiology, Neurology, etc.) — broader than the indication.
Target USDM fields: study.versions[0].businessTherapeuticAreas[] of Code objects.
Anchor / regex hints: the protocol footer brand line CT02-GSOP **Oncology** Clinical Protocol Template directly tells you the therapeutic area for any protocol on this template.
Edge cases: keep the indication itself out of this field — that goes on studyDesign.indications[].


7. usdm-studyIdentifiers

Source (primary): §1.1 Synopsis → "Regulatory Agency Identification Number(s):" table. Standard rows:

US IND Number:
EU CT Number: (also EudraCT Number in older protocols)
ClinicalTrials.gov ID:
Pediatric Investigational Plan Number:
Protocol Number: (sponsor's internal)
Phase:


Source (fallback): title page (Protocol Number appears in the running header on every page).
What to extract: for each identifier, (text, scope, type). The scope is the regulatory agency or sponsor; the type is the code (IND / NCT / EU CT / Sponsor Internal).
Target USDM fields: study.versions[0].studyIdentifiers[] (one StudyIdentifier per row).
Anchor / regex hints: bold labels followed by colon and value on the same line; layout‑mode pdftotext often misaligns the labels and values (see §4.2 of my earlier report — the JSON has a parsing artifact where 140264 floats above US IND Number:). The skill should pair labels with values robustly, e.g., by reading right‑hand column of a 2‑column synopsis table.
Edge cases: "Not Applicable" should be skipped, not stored. Phase: is not a study identifier — it goes on studyDesign.studyPhase.


8. usdm-referenceIdentifiers

Source (primary): §11 References (the bibliography) — every cited document.
Source (fallback): in‑text citations referencing standards (RECIST v1.1, CTCAE v5.0, CKD‑EPI 2021, AJCC 8th Ed., ASCO/CAP HER2 guideline, ICH GCP, etc.) plus the CT02‑GSOP template ID itself in the page footer.
What to extract: for each cited standard or guideline that the protocol applies as a normative reference (not every paper in the bibliography), produce (text, scope, version) and link to an organization where applicable (NCI for CTCAE, RECIST WG for RECIST, AJCC for staging, etc.).
Target USDM fields: study.versions[0].referenceIdentifiers[] of ReferenceIdentifier. Each should set scopeId to an Organization id (currently the generator leaves scopeId empty — fix in skill).
Anchor / regex hints: bold heading 11. REFERENCES. For normative standards, scan §8 (Assessments) and the appendices for phrases like per RECIST v1.1, NCI CTCAE v5.0, 2021 CKD-EPI Equations, AJCC 8th edition, etc.
Edge cases: distinguish scientific references (which can stay in references if your model has it) from regulatory/methodology references (which belong here). Only the latter goes in referenceIdentifiers.


9. usdm-titles

Source (primary): §1.1 Synopsis — bold labels Protocol Title: and Brief Title: directly above their respective values.
Source (fallback): title page (Official title appears verbatim).
What to extract:

Official Study Title (long version) — value next to Protocol Title:
Brief Study Title (short version) — value next to Brief Title:
Study Acronym — usually parenthesized at the end of the official title (e.g., (VERITAC‑3))
Public Study Title — same as Brief unless protocol distinguishes them
Scientific Study Title — same as Official unless protocol distinguishes


Target USDM fields: study.versions[0].titles[] of StudyTitle. Each has text and type (CDISC code: Official Study Title, Brief Study Title, Study Acronym, Public Study Title, Scientific Study Title).
Anchor / regex hints: capture ^Protocol Title:\s*(.+?)\s*$ then continue lines until next bold label. Acronym extracted from the parenthetical at the title's end (e.g., \(([A-Z][A-Z0-9-]+)\)$).
Edge cases: the brief title sometimes ends with the acronym in parens too; deduplicate.


10. usdm-studyDesigns


| StudyDesign node | Primary source | Supplementary / cross-check |
|---|---|---|
| id | generated | — |
| extensionAttributes | sponsor extensions, if any | — |
| name | generated convention, e.g., `MAIN_DESIGN` | — |
| label | §4.1 Overall Design, heading or first-line description | §1.1 Synopsis “Study Design” sub-block |
| description | §4.1 Overall Design, full prose | §1.1 Synopsis |
| studyType | §4.1 Overall Design | §1.1 Synopsis “Study Design” — typically Interventional Study from CT02-GSOP template |
| studyPhase | §1.1 Synopsis “Phase” row | Title page; running title sometimes states phase |
| therapeuticAreas | §1.1 Synopsis “Indication” + §2.2.1 disease background | Title page; protocol footer template name, e.g., “Oncology Clinical Protocol Template” |
| characteristics | §4.1 Overall Design + §4.2 Scientific Rationale | §1.1 Synopsis, study design features like adaptive, multicenter, international |
| encounters | §1.3 SoA column headers, Table 1 | §10.14 Appendix 14 historical SoA tables, Tables 16, 17, 18 |
| activities | §1.3 SoA row labels, Table 1 | §8 Study Assessments and Procedures; §10.14 Appendix 14 |
| arms | §6.1 Tables 4, SLI, + 6, Phase 3 | §1.2 Schema; §1.1 Synopsis “Treatments” |
| studyCells | §1.2 Schema + §4.1 Overall Design, arm × epoch flow | computed from arms × epochs, phase-restricted |
| rationale | §1.1 Synopsis “Rationale” sub-block | §4.2 Scientific Rationale for Study Design |
| epochs | §1.2 Schema + §4.1 Overall Design + §7.1.1 Post-Treatment Follow-up | §1.3 SoA column groupings |
| elements | §4.1 Overall Design + §6 Study Intervention(s), one element per epoch defining the activities/intervention regime | §1.2 Schema |
| estimands | §3 Objectives, Endpoints, and Estimands table; refer if needed to §9.1 Statistical Hypotheses | §3 Objectives, Endpoints, and Estimands table |
| indications | §1.1 Synopsis “Indication” | Title page; §2.2.1 disease background; §5 Study Population |
| studyInterventionIds | §6.1 Tables 3, SLI, + 5, Phase 3 | links to `versions[0].studyInterventions[]` |
| objectives | §3 Objectives, Endpoints, and Estimands, the §3 table | §1.1 Synopsis “Objectives and Endpoints” sub-block |
| population | §5 Study Population, intro + §5.1 + §5.2, + §1.1 Synopsis enrollment numbers | §1.2 Schema, N per cohort; §9.5 Sample Size Determination |
| scheduleTimelines | §1.3 Schedule of Activities | §10.14 Appendix 14 Tables 16, 17, 18, historical timelines |
| biospecimenRetentions | §8.6.2 Retained Research Samples for Genetics + §8.7 Biomarkers, tumor biopsies, blood samples | §10.5 Appendix 5 Genetics |
| documentVersionIds | Document History table before TOC + §10.14 Appendix 14 | references `study.documentedBy.versions[]` |
| eligibilityCriteria | §5.1 Inclusion Criteria + §5.2 Exclusion Criteria | §1.1 Synopsis “Key Eligibility Criteria”; links to `versions[0].eligibilityCriterionItems[]` via `criterionItemId` |
| analysisPopulations | §9.2 Analysis Sets | §9.3 Statistical Analyses, when populations are referenced in analysis methods |
| notes | sponsor-defined annotations on the design | typically empty unless the design needs annotation |
| instanceType | constant `InterventionalStudyDesign` | — |
| subTypes | §4.1 Overall Design, e.g., dose-escalation + parallel-group | §1.1 Synopsis |
| model | §4.1 Overall Design, e.g., parallel-group, crossover, single-arm | §1.1 Synopsis “Study Design” |
| intentTypes | §3 Objectives, e.g., treatment / prevention / diagnostic / supportive care | §1.1 Synopsis, inferred from primary objective |
| blindingSchema | §4.1 Overall Design + §6.4 Blinding | §1.1 Synopsis “Study Design”, e.g., open-label, double-blind |


FOR SOA EXTRACTION


Activity and Encounter are reusable definitions — they live on the StudyDesign and don't know anything about the schedule. You could lift them out and reuse them in a completely different study.

ScheduledActivityInstance and Timing are schedule-specific — they live inside a ScheduleTimeline and represent how those reusable building blocks are arranged in time for this protocol.


Practical consequence for your skill
When parsing multiple SoA tables, the algorithm should:
    1. First pass — walk all SoA tables and accumulate the unique set of column headers and row labels into the shared pools (encounters[], activities[], bcCategories[], biomedicalConcepts[]).
    2. Second pass — for each table, build its ScheduleTimeline by referencing back to the shared pool ids; only the instances[], timings[], entryId, and the timeline-level metadata are unique to that table.
    3. Validate — every Encounter and Activity that appears in any timeline's instances should be defined once and only once at the studyDesign level. No duplicates with the same name.
That two-pass approach is what gives you the right shape — shared vocabulary, distinct schedules.


11. usdm-organizations

    • Source (primary): Title page (page 1) — sponsor address blocks. 
    • Source (fallback): 
        ○ §1.1 Synopsis — sponsor names mentioned briefly
        ○ §10.1.6



What to extract: for each org: name, identifier (DUNS or sponsor‑internal), identifierScheme (DUNS, RingGold, Sponsor), type (Sponsor, CRO, Vendor, Independent Committee), and legalAddress (lines, city, district/state, country, postalCode).
Target USDM fields: study.versions[0].organizations[].
Anchor / regex hints: title page contains literal labels Sponsor Name and Legal Registered Address (US) / (non-US) with the address block immediately below.
Edge cases: the current generator extracts org names but leaves legalAddress.lines = [] and city = "" — the skill must parse the address blocks on the title page (4–5 lines each).


12. usdm-studyInterventions

Source (primary):

§6.1 Study Intervention(s) Administered — overview of all interventions
Table 3 SLI Study Interventions, Table 5 Phase 3 Study Intervention — structured tables giving name, role, dose levels, schedule
§6.1.1 Administration → 6.1.1.1 / 6.1.1.2 / 6.1.1.3 (per‑drug subsections) — give route, frequency, fasted/fed, etc.
Tables 7, 8, 9 — dose levels per drug


Source (fallback):

§1.1 Synopsis → "Study Intervention" / "Treatments" sub‑block (concise table)
§4.3 Justification for Dose — rationale, supports description
§6.9 Prior and Concomitant Therapy subsections (6.9.1 LHRH, 6.9.2 prohibited, 6.9.6 growth factors, 6.9.7 antiemetics/antidiarrheals, 6.9.8 corticosteroids) — each becomes a StudyIntervention with role Concomitant Medication or Prohibited Medication


What to extract: for each intervention: name, label, description, type (Drug/Biological/Device), role (Investigational/Comparator/Concomitant Medication/Prohibited), codes (preferably WHODrug or RxNorm), pharmacologicalClass, productDesignation, administrableProductIds[] (link to administrableProducts), and administrations[].
Target USDM fields: study.versions[0].studyInterventions[].
Anchor / regex hints: Tables 3, 4, 5, 6 are the most reliable structured source. The §6.9.X sub‑sections are the source for concomitant‑medication interventions.
Edge cases / gotchas:

The generator is currently leaving administrations skeletal (empty route, dose, frequency). Pull these from §6.1.1 prose like "vepdegestrant 200 mg orally once daily continuously in 28-day cycles" and structure them.
Modeling concomitant‑medication categories (§6.9.6 etc.) as StudyIntervention is unconventional — consider whether they belong elsewhere in your skill's output.




13. usdm-administrableProducts

Source (primary):

Table 7 vepdegestrant Available Dose Levels, Table 8 Palbociclib Available Dose Levels, Table 9 Palbociclib Dose Levels for Phase 3 Control Arm — these tables enumerate every dose strength as a separable product.
§6.1.1 Administration subsections — give dose form (tablet/capsule), route, fasted/fed in. I structions.
§6.2 Preparation, Handling, Storage and Accountability — manufacturer, packaging, supply chain.


Source (fallback): Investigator Brochure (referenced in §11) — but typically out of scope for the skill.
What to extract: for each (drug × strength) combination: name (e.g., VEPDEGESTRANT_200MG), label, description, doseForm (Code: tablet, capsule), strength, ingredient[], productDesignation (IMP/NIMP), administrations[] with route and frequency.
Target USDM fields: study.versions[0].administrableProducts[].
Anchor / regex hints: Table 7/8/9 captions; rows like Dose Level 1: Palbociclib 100 mg PO QD give name + route + frequency directly.
Edge cases: one drug at multiple strengths = multiple AdministrableProducts but a single StudyIntervention (linked via administrableProductIds). This generator collapsed them correctly but left fields empty.


14. usdm-medicalDevices

Source (primary): §6 — if any device is part of the intervention (e.g., a companion diagnostic, an electronic dose dispenser, an injection device). Look for sub‑sections explicitly naming a device or using the word "Device".
Source (fallback):

§8.4.9 Medical Device Deficiencies — confirms whether the protocol uses a device at all (if this section is "Not applicable" / boilerplate, there are no devices).
§8.7.1.1 Tumor Biopsies — mentions companion diagnostics or biopsy devices in some protocols.


What to extract: for each device: name, description, type, regulatoryStatus, manufacturer (link to org).
Target USDM fields: study.versions[0].medicalDevices[].
Anchor / regex hints: explicit mentions of device, companion diagnostic, IVD, wearable, eDiary, injector. For VERITAC‑3 specifically: no devices (§8.4.9 is boilerplate only) → empty array is correct.
Edge cases: an eCOA / eDiary system used for PRO collection (§10.13.1) is debatably a device; current convention is to treat eCOA as a system, not a MedicalDevice, unless the protocol specifically classifies it as one.


15. usdm-productOrganizationRoles

Source (primary):

Title page — sponsor responsibility (manufacturer/MA holder) for IMP.
§6.2 Preparation, Handling, Storage and Accountability — names the supplier/manufacturer for each IMP.
§10.1.13 Transfer of Obligation Statement — sponsor ↔ CRO role transfers.


Source (fallback): Investigator Brochure references; §10.1 generally for sponsor obligations.
What to extract: triples of (role, organizationId, appliesToIds) where role is Manufacturer / Marketing Authorization Holder / Supplier / Sponsor, organizationId references organizations[], and appliesToIds references studyInterventions[] or administrableProducts[].
Target USDM fields: study.versions[0].productOrganizationRoles[].
Anchor / regex hints: §6.2 typically contains phrases like IMP will be supplied by <sponsor> or manufactured by <manufacturer>.
Edge cases: the current generator emits 3 placeholder entries with all fields empty — replace with real extractions or remove.


16. usdm-biomedicalConcepts

Source (primary): §8 Study Assessments and Procedures — every numbered subsection from §8.1 through §8.9 maps to one or more biomedical concepts:

§8.1 Administrative & Baseline → Demographics, Informed Consent
§8.2 Efficacy → Tumor Response (RECIST v1.1), Disease Assessments
§8.3 Safety → Physical Examination, Vital Signs, ECG, Clinical Safety Labs, Pregnancy Testing
§8.4 AEs/SAEs → Adverse Event (concept)
§8.5 Pharmacokinetics → PK Sample Collection (per drug)
§8.6 Genetics → Genetic Sample
§8.7 Biomarkers and PD → Tumor Biopsies, Blood Samples for biomarkers, ctDNA, TK1, Ki67
§8.8 Immunogenicity → Immunogenicity Sample
§8.9 Health Economics → PRO/HRQoL instruments


Source (supplementary):

Table 1 Schedule of Activities — every row name in the SoA is a BC (or maps to one).
§10.2 Appendix 2: Clinical Laboratory Assessments — gives the per‑analyte breakdown for the "Clinical Safety Labs" BC (Hematology, Chemistry, Coagulation, Urinalysis panels with each individual test as a property).
§10.13 Appendix 13: Additional Study Assessments, Inventories, and Questionnaires — PRO instruments & ECOG.


What to extract: for each BC: name, label, code (CDISC SDTM concept code), reference (CDISC URL), and properties[] where each property is a measurable variable (e.g., for "Vital Signs": systolic BP, diastolic BP, pulse, temperature, each with units and data type).
Target USDM fields: study.versions[0].biomedicalConcepts[].
Anchor / regex hints: SoA row labels are the cleanest enumeration; §8 sub‑headings are the next cleanest.
Edge cases / gotchas: the generator currently emits 21 BCs as label‑only placeholders (code: "", properties: []). The skill must enrich them — ideally by mapping each row name to a CDISC SDTM Biomedical Concept library entry.


17. usdm-bcCategories

Source (primary): §8 sub‑headings as natural categories:

Efficacy ← §8.2
Safety ← §8.3
Pharmacokinetics ← §8.5
Genetics ← §8.6
Biomarkers / Pharmacodynamics ← §8.7
Immunogenicity ← §8.8
Health Economics / PROs ← §8.9
Reproductive Safety ← §8.3.5 + Appendix 4 (split out for typical convention)


Source (supplementary): SoA section dividers (the row groups in Table 1 with shaded category headers).
What to extract: category name, label, and memberIds[] listing the BiomedicalConcept ids that belong to this category.
Target USDM fields: study.versions[0].bcCategories[].
Anchor / regex hints: §8 sub‑section numbers map 1‑to‑1 with categories.
Edge cases: keep the category set small (5–8); avoid one‑off categories.


18. usdm-bcSurrogates

Source (primary):

§8.7.1 Biomarkers subsections (8.7.1.1 Tumor Biopsies, 8.7.1.2 Blood Samples) — names individual biomarkers (e.g., ctDNA, TK1, Ki67, ESR1).
§8.7.2 Pharmacodynamics (SLI only) — PD markers.


Source (fallback): Section 1.1 Synopsis "Biomarker / PD" sub‑block when present.
What to extract: name, label, and reference (typically a UMLS / GenBank / HGNC reference for the biomarker entity).
Target USDM fields: study.versions[0].bcSurrogates[]. Each is a BiomedicalConceptSurrogate and is referenced by a BiomedicalConcept via its surrogateId field.
Anchor / regex hints: in §8.7, sentences like "ctDNA will be assessed for ESR1 mutations" produce surrogates ctDNA, ESR1 (the latter as a target gene).
Edge cases: distinguish a surrogate (a biomarker that stands in for an outcome) from a property of a BC (a measured variable on a sample). The convention isn't fully crisp; align with how downstream consumers expect these.


19. usdm-eligibilityCriterionItems

Source (primary):

§5.1 Inclusion Criteria — usually grouped under sub‑headings such as Age and Sex, Type of Participant and Disease Characteristics, Medical Conditions, Prior Therapy, Diagnostic Assessments, Other.
§5.2 Exclusion Criteria — same sub‑heading structure.


Source (fallback): §1.1 Synopsis → "Key Eligibility Criteria" — abbreviated list for the synopsis only.
What to extract: for each numbered criterion: identifier (e.g., IN01, EX05), text (the verbatim criterion), category (Inclusion/Exclusion), and the sub‑heading it lives under (e.g., Medical Conditions).
Target USDM fields:

study.versions[0].eligibilityCriterionItems[] — the reusable, text‑bearing items.
study.versions[0].studyDesigns[0].eligibilityCriteria[] — the applied criteria, each with a criterionItemId link to the item above.


Anchor / regex hints: numbered list under headings 5.1.\s and 5.2.\s. Each item starts with a digit and a period.
Edge cases / gotchas:

The two arrays must be linked: every EligibilityCriterion.criterionItemId should reference an EligibilityCriterionItem.id. The current generator leaves criterionItemId as "" — the skill must fix this.
The sub‑heading group should be preserved (currently flattened to a single Inclusion/Exclusion category).




20. usdm-narrativeContentItems

Source (primary): the entire body of the protocol, section by section. The mapping is essentially 1‑to‑1 with the major TOC sections of the CT02‑GSOP template:

Item nameProtocol sectionNotesNARR_TOC(TOC pages)optionalNARR_SYNOPSIS§1.1full synopsis proseNARR_SCHEMA§1.2schema text + figure captionNARR_SOA§1.3 (and §1.3.1)SoA narrative; the SoA table is structural dataNARR_INTRO§2.1 + §2.2study rationale + backgroundNARR_BACKGROUND_DISEASE§2.2.1breast cancer / disease backgroundNARR_BACKGROUND_TREATMENT§2.2.2current standard of careNARR_BACKGROUND_PRODUCT§2.2.3investigational product backgroundNARR_BENEFIT_RISK§2.3risk + benefit + overallNARR_OBJECTIVES§3objectives & endpoints prose around the tableNARR_OVERALL_DESIGN§4.1NARR_DESIGN_RATIONALE§4.2overlaps with versions[0].rationale — keep bothNARR_DOSE_JUSTIFICATION§4.3NARR_END_OF_STUDY§4.4NARR_POPULATION§5 (intro)NARR_LIFESTYLE§5.3NARR_INTERVENTION§6.1, §6.1.1NARR_DOSE_MOD§6.6NARR_PRIOR_CONCOMITANT§6.9NARR_DISCONTINUATION§7NARR_ASSESSMENTS_<X>§8.1–§8.9one per sub‑sectionNARR_STATISTICAL§9one per §9.1–§9.5 ideallyNARR_APPENDIX_01 … NARR_APPENDIX_15§10.1 … §10.15one per appendixNARR_REFERENCES§11

What to extract: for each NCI: name, text (HTML or markdown of the section body), and a NarrativeContent wrapper that gives sectionNumber, sectionTitle, and parent/child relationships.
Target USDM fields:

study.versions[0].narrativeContentItems[] — the text bodies.
study.documentedBy.versions[0].contents[] — the NarrativeContent hierarchy that anchors each item to a section number and parent. Currently empty in the generator output — this is the major fix.


Anchor / regex hints: TOC entries give exact section number → title pairs. Each section heading in the body matches the regex ^(\d+(?:\.\d+)*)\.\s+([A-Z][^\n]+)$.
Edge cases / gotchas:

All 15 appendices must be modeled. The current generator covers only the main body and skips appendices entirely.
Tables within sections (Table 1 SoA, Tables 7‑11 dose tables, etc.) should ideally remain as structural data (not narrative text), but if your downstream consumer expects rendered protocol text, the table text should be embedded in the relevant NCI.




21. usdm-abbreviations

Source (primary): §10.15 Appendix 15: Abbreviations — a 2‑column table of Abbreviation | Definition for the entire protocol.
Source (fallback): in‑text first‑use definitions like progression‑free survival (PFS) — but Appendix 15 should be authoritative.
What to extract: for each row: abbreviatedText (left column) and expandedText (right column).
Target USDM fields: study.versions[0].abbreviations[] of Abbreviation.
Anchor / regex hints: appendix heading 10.15. Appendix 15: Abbreviations. Two‑column table with single‑word/acronym left, multi‑word definition right.
Edge cases: some abbreviations carry punctuation (e.g., ER(+)) — preserve as‑is. Some entries are multi‑line definitions.


22. usdm-roles

Source (primary):

Title page — sponsor entities (US, non‑US) with their declared role.
§10.1.5 Committees Structure — DMC, Steering Committee, Independent Reviewer (RECIST BICR) roles.
§10.1.12 Sponsor's Medically Qualified Individual — sponsor MD role.
§10.1.13 Transfer of Obligation Statement — CRO/Vendor role assignments.


Source (fallback):

§9.4 Interim Analyses — names DMC/eDMC role.
§8.2.3 Independent Review of Disease Assessments — BICR role (Phase 3).


What to extract: for each role: name, label, code (CDISC NCI Thesaurus code: Sponsor, Investigator, Independent Data Monitoring Committee, Steering Committee, BICR Reviewer, Medical Monitor, CRO, etc.), and the appliesToIds[] linking organizations or persons.
Target USDM fields: study.versions[0].roles[].
Anchor / regex hints: §10.1.5 is the most reliable structured source for committee roles.
Edge cases: Sponsor (US) is not a standard CDISC code — use Sponsor and qualify with the organization's address country, rather than inventing a sub‑variant.


23. usdm-dictionaries

Source (primary): explicit version mentions in the protocol body:

MedDRA — referenced in §8.4 (AE coding) and Appendix 3; protocol typically says MedDRA version <X.Y> somewhere in §8.4 or §10.3.
NCI CTCAE v5.0 — explicit mentions in §8.3.4 / §8.4 / Appendix 7.3 (Adverse Event Grading for Kidney Safety).
RECIST v1.1 — Appendix 11 in full.
WHODrug — for prior/concomitant medication coding (§6.9 or sometimes only mentioned in CRF documentation, not the protocol).
ECOG Performance Status — Appendix 13.13.2.


Source (fallback): §11 References for the canonical citation of each standard.
What to extract: for each dictionary: name, label, version, url / reference.
Target USDM fields: study.versions[0].dictionaries[].
Anchor / regex hints:

MedDRA: regex MedDRA\s+(?:version\s+)?(\d+\.\d+).
CTCAE: regex CTCAE\s+v?(\d+\.\d+) (typically v5.0).
RECIST: regex RECIST\s+v?(\d\.\d) (typically v1.1).


Edge cases: if no MedDRA version is stated in the protocol, leave version empty rather than guessing — but flag for review (the current generator already does this).


Cross‑node integrity rules (skill should enforce)
These are not nodes themselves but cross‑cutting integrity rules the skill must enforce after extraction:

Every EligibilityCriterion.criterionItemId ⇒ exists in eligibilityCriterionItems[].id.
Every Arm.populationIds[] ⇒ exists in studyDesign.population.id (or cohorts[].id).
Every StudyCell ⇒ armId and epochId belong to the same phase (no SLI × Phase 3 cross‑pairings).
Every BiomedicalConcept ⇒ has at least one bcCategory referencing it via memberIds.
Every StudyDefinitionDocumentVersion.contents[] NarrativeContent ⇒ references a narrativeContentItems[].id that exists.
Every documentVersionIds[] ⇒ resolves to a StudyDefinitionDocumentVersion.id under study.documentedBy.
No CDISC code = C99999. Every Code.code must be a real CDISC NCI Thesaurus concept.
Every referenceIdentifier.scopeId ⇒ resolves to an Organization.id.


Quick‑lookup index (alphabetical by node)
NodePrimary sourceabbreviations§10.15 Appendix 15administrableProductsTables 7, 8, 9 + §6.1.1amendments§10.14 Appendix 14 (full) + Document History (latest only)bcCategories§8 sub‑headingsbcSurrogates§8.7.1, §8.7.2biomedicalConcepts§8 + Appendix 2 + Appendix 13 + SoA rowsbusinessTherapeuticAreasTitle page + footer brand linedateValuesTitle page + footer "Approved On" + Document History + §4.4dictionaries§8.4, §10.3 (MedDRA), §8.3 (CTCAE), §10.11 (RECIST), §10.13.2 (ECOG)documentVersionIdsDocument History (before TOC) + §10.14eligibilityCriterionItems§5.1, §5.2medicalDevices§6 (if present); §8.4.9 confirms presencenarrativeContentItemsAll sections §1–§11 + appendices §10.1–§10.15organizationsTitle page (sponsor blocks) + §10.1.5, §10.1.12, §10.1.13productOrganizationRolesTitle page + §6.2 + §10.1.13rationale§4.2 (primary), §1.1 (fallback)referenceIdentifiers§11 References + in‑text standard citationsroles§10.1.5, §10.1.12, §10.1.13 + Title pagestudyDesigns§1.2, §1.3, §3, §4, §5, §6, §9.1studyIdentifiers§1.1 Synopsis "Regulatory Agency Identification Number(s)"studyInterventionsTables 3, 5 + §6.1, §6.1.1, §6.9.Xtitles§1.1 Synopsis "Protocol Title" / "Brief Title" + Title pageversionIdentifierTitle page (line under title)