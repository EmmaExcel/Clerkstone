Live demo: https://clerk.excelemma.site/

All patient data here is synthetic. This is a personal concept project. It has no NHS approval, no clinical validation, and it must not be used for patient care.

The problem

NHS trusts move from one patient record system to another fairly often. Updates, changes, and sometimes two trusts merge. When that happens they inherit data they can't fully trust. NHS numbers go missing. Discharge dates land before admission dates. IDs get duplicated. Local codes map to nothing. Those errors then follow the data into the live system.

I found this while researching healthcare data for ML projects. NHS workers talk about it a lot. I wanted to see how I would catch those errors before they get anywhere near a live system, so I built this.

What it does

	•	Takes in FHIR R4 bundles, skips duplicates by logical id, and stores every resource
	•	Checks structure with fhir.resources, then checks UK Core profiles with the HL7 FHIR Validator running in its own container
	•	Runs 65 data quality rules across 9 categories: required fields, identifiers, references, dates, physiological ranges, terminology, duplicates, cohort and structure
	•	Reports every defect with a rule ID, a severity and the FHIRPath location
	•	Serves the data through a REST API with FHIR search, $validate and $export, and converts HL7 v2 ADT messages
	•	Shows a patient timeline in a React UI built with the NHS.UK design system, using the NHS frontend MCP server I built earlier
	•	Logs every read and write in an append-only audit log, with each entry hash-chained using SHA-256
	•	Locks things down with JWT (RS256) auth, roles and field level access

Stack

	•	FastAPI and Pydantic v2 for the API
	•	PostgreSQL 16 for storage. Raw FHIR sits in a JSONB table, and indexed tables hold patients, encounters, observations, conditions and medication requests
	•	fhir.resources for the R4 models
	•	HL7 FHIR Validator as a sidecar container
	•	Next.js 15 and TypeScript for the UI

Numbers

	•	20 synthetic patients in the committed fixture, up to 500 with make synth
	•	13 FHIR resource types
	•	65 rules, each with a passing, a failing and a boundary test
	•	48 defect fixtures with exact expected reports
	•	91 backend tests at 85% line coverage
	•	0 serious axe-core violations

The data comes from Synthea. A script (scripts/corrupt.py) then breaks it on purpose using seeds saved in data/fixtures/manifest.json, so every run gives the same defects.

Run it locally

brew install openjdk@17                   # Synthea and the validator need Java 17
make install                              # uv sync
make up                                   # start Postgres with docker compose
make migrate                              # apply Alembic migrations
make synth && make corrupt && make load   # generate, corrupt and load the data
make dev                                  # API on :8000

Tests run with make test. To check the audit log chain, run make verify-audit.

Limits

	•	UK Core is a work in progress from NHS England and is marked not for implementation. My profile checks are a best effort. The validator version is pinned and shown in every report.
	•	The SNOMED CT and dm+d mapping table is small and hand checked. Don't treat it as an official NHS mapping.
	•	The rule engine only catches the defects I thought of. Every report shows which ruleset version ran.
	•	There is no ML in this project. Every check is a rule, and a model guessing what a rule already knows would add nothing. The reasoning is in docs/DECISIONS.md (ADR-002).
	•	Terminology lookups use a local cached subset. The live NHS England Terminology Server needs an approved account. The adapter is ready to swap when I get one.
	•	I'm not a registered Clinical Safety Officer, so there is no signed safety case. The safety docs follow DCB0129 as far as I could take them alone.

More docs

	•	docs/architecture.md
	•	docs/DEFECT_TAXONOMY.md
	•	docs/API_GUIDE.md
	•	docs/Hazard_Log.md
	•	docs/THREAT_MODEL.md
	•	docs/accessibility.md
	•	docs/RUNBOOK.md

Data and licences

Patient data comes from Synthea (Apache-2.0). No real patient data is stored, processed or sent anywhere. More on sources in docs/DATA_LICENCES.md.