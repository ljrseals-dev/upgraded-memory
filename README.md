# ElevenX Provisional Patent Package

**Repository:** https://github.com/ljrseals-dev/ElevenX  
**Inventor:** Lewis James Seals Jr.  
**Package prepared:** 2026-10-29  
**Purpose:** Supporting materials for a United States Provisional Patent Application (PPA)

---

## Contents

```
ElevenX-Patent-Package/
├── README.md                          (this file)
├── specification/
│   └── PROVISIONAL_PATENT_SPECIFICATION.md
├── source/                            (working tree of ElevenX at time of packaging)
│   ├── src/elevenx/                   (engine, api, emotions, speakers, studio, …)
│   ├── config.yaml
│   ├── requirements.txt
│   ├── pyproject.toml
│   ├── README.md
│   └── LICENSE
├── docs/
│   ├── FLORILEGIUM.md                 (curated list of potential inventive elements)
│   ├── FILING_CHECKLIST.md
│   └── repo-docs/                     (docs copied from the public repository)
└── forms/
    └── COVER_SHEET_NOTES.md           (how to complete USPTO Form PTO/SB/16)
```

---

## Important Notices

1. **Not legal advice.** This package is an educational template assembled from publicly available repository materials and the accompanying florilegium analysis. It does not constitute legal advice. Filing a provisional application has legal consequences; consult a registered patent practitioner if you intend to commercialize or enforce rights.

2. **First-inventor-to-file.** Under the America Invents Act, priority is determined by filing date, not conception date. A sealed envelope or self-addressed mailing provides no patent rights.

3. **Micro-entity fee.** As of the 2026 USPTO fee schedule the micro-entity provisional filing fee is $65, subject to eligibility certification (income and prior-filing limits). Confirm current fees at uspto.gov.

4. **Disclosure quality.** A provisional application can only claim priority to matter that is adequately described in the specification filed on the priority date. New matter cannot be added later. Review the detailed description against the actual source code before filing.

5. **Open-source status.** The ElevenX repository is published under an MIT-style license with copyright notice. Public disclosure can affect patentability timelines in some jurisdictions. Seek counsel regarding the interaction of open-source release and patent filing strategy.

---

## Recommended Next Steps

1. Review and edit `specification/PROVISIONAL_PATENT_SPECIFICATION.md` for technical accuracy and completeness.
2. Complete personal information (inventor residence, correspondence address).
3. Optionally prepare simple block diagrams of the request flow and module relationships (drawings strengthen the record even though they are not mandatory for a provisional).
4. Create a USPTO.gov account and file via Patent Center using Form PTO/SB/16.
5. Pay the applicable micro-entity (or small-entity / large-entity) fee.
6. Calendar the 12-month deadline for a non-provisional utility application that claims the benefit of this provisional.

---

## Source Snapshot

The `source/` directory contains the ElevenX source tree as of the packaging date, including:

- Lazy-loaded XTTS-v2 orchestration (`engine.py`)
- Optional ECAPA speaker encoder (`speakers.py`)
- Emotion/style presets (`emotions.py`)
- Serialized synthesis with backpressure and threaded inference (`api.py`)
- Bundled Voice Studio static UI (`static/index.html` + `studio.py`)
- Configuration defaults (`config.yaml`)

This snapshot serves as supporting technical disclosure for the provisional specification.

---

*Prepared for educational and documentation purposes in connection with the ElevenX project.*
