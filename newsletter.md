# Building a Production-Ready RAG Knowledge Base for Carbon Registries

For a production RAG system, you should **never think of the knowledge base as something you manually update**. Instead, think of it as a **continuously synchronized mirror of authoritative sources**.

For carbon registries (Verra, Gold Standard, Climate Action Reserve, ACR, Puro, Plan Vivo, GCC, etc.), it's best to build an **Update Pipeline** that is completely separate from the RAG pipeline.

---

# 1. Subscribe to Official Update Channels

Most major carbon registries already notify users whenever they publish:

- New standards
- New methodology versions
- Corrections
- Public consultations
- New templates
- Registry changes

## Examples

### Verra

- Newsletter
- Program updates
- Methodology consultations
- VCS version releases

For example, **VCS Version 5** was announced through its newsletter.

### Gold Standard

- Technical newsletter
- Quarterly technical updates
- Methodology updates
- Registry announcements

### Climate Action Reserve (CAR)

- Monthly newsletter
- Protocol updates
- Public comment periods

### Puro.earth

- Monthly newsletter
- Methodology updates
- Registry updates

Most of these newsletters announce new versions **before or immediately when they're released**.

---

# 2. Don't Rely Only on Newsletters

Newsletters are helpful, but they are **not sufficient**.

Some organizations silently upload:

- VCS Standard v4.8
- Methodology VM0047 v2.0
- New guidance PDFs

without sending a major newsletter announcement.

Instead, monitor multiple sources:

```text
Official Website
      ↓
RSS (if available)
      ↓
Newsletter
      ↓
Document pages
      ↓
Public consultation pages
      ↓
Registry announcements
```

This ensures you catch both announced and silent document updates.

---

# 3. Build an Update Agent

Run an automated update agent on a schedule (for example, every night).

```text
Cron Job
    ↓
Visit Verra document library
    ↓
Download page
    ↓
Compare with yesterday
    ↓
New PDF?
    ↓
YES
    ↓
Download
    ↓
Extract text
    ↓
Chunk
    ↓
Embed
    ↓
Update Vector DB
    ↓
Notify Slack
```

This approach is much more reliable than waiting for emails.

---

# 4. Build a Version-Aware Knowledge Base

**Never overwrite old documents.**

Instead, keep every released version.

Example:

```text
VCS Standard

v4.0
v4.1
v4.2
v4.3
v5.0
```

Store metadata such as:

```json
{
  "registry": "Verra",
  "document": "VCS Standard",
  "version": "5.0",
  "published": "2025-12-18",
  "status": "Current"
}
```

Older versions remain stored:

```json
{
  "version": "4.4",
  "status": "Superseded"
}
```

This allows your RAG system to answer questions like:

- "According to VCS v4.4..."
- "According to the latest VCS..."

without losing historical context.

---

# 5. Automatic Ingestion Pipeline

Whenever a new PDF appears:

```text
Download
    ↓
OCR (if needed)
    ↓
Extract tables
    ↓
Chunk
    ↓
Generate embeddings
    ↓
Insert into Pinecone / Weaviate / Qdrant
    ↓
Mark previous version as superseded
```

No human intervention should be required.

---

# 6. Maintain a Document Registry

Store metadata for every ingested document.

| Registry | Document | Version | Release Date | Hash |
|----------|----------|----------|--------------|------|
| Verra | VCS Standard | 5.0 | 2025 | SHA256 |
| Gold Standard | Requirements | 2.1 | 2026 | SHA256 |

The **SHA256 hash** helps detect whether a PDF changed even if the filename remained the same.

---

# 7. Detect What Actually Changed

Instead of embedding the entire document again:

```text
Old PDF
    ↓
New PDF
    ↓
Compare sections
    ↓
Only re-embed changed chunks
```

Example:

```text
Section 2 → Unchanged

Section 3 → Changed

Appendix B → Changed
```

Only the modified sections are re-indexed, which significantly reduces embedding costs.

---

# 8. Maintain a Change Log

Keep a structured log of all detected updates.

| Date | Registry | Change |
|------|----------|--------|
| Jan 2026 | Verra | VCS v5 released |
| Feb 2026 | Gold Standard | New methodology |
| Mar 2026 | CAR | Soil protocol updated |

This enables the RAG system to answer questions like:

> "What's changed since VCS v4?"

without comparing PDFs during query time.

---

# 9. Recommended Architecture

```text
                  Official Sources

             Verra
             Gold Standard
             CAR
             ACR
             Puro
             Plan Vivo
             GCC
                    │
                    ▼
         Update Monitoring Agent
      (daily / every 6 hours)
                    │
      ┌─────────────┴─────────────┐
      ▼                           ▼
 New Document               Changed Document
      ▼                           ▼
 Extract Text             PDF Diff Engine
      ▼                           ▼
 Chunk Documents      Re-embed Changed Chunks
      └─────────────┬─────────────┘
                    ▼
             Vector Database
          (Qdrant / Pinecone / Weaviate)
                    ▼
                RAG Agent
```

---

# 10. If I Were Building This Today

I would **not rely on newsletters alone**.

Instead, I would combine:

- Newsletters for early awareness of major releases.
- Scheduled crawlers that check official document libraries and methodology pages daily.
- File hashing to detect silent updates.
- Version-aware storage so no document is ever overwritten.
- Incremental embedding so only changed content is re-indexed.
- Human notifications (Slack/email) whenever a new standard, methodology, or guidance document is detected.

This approach is robust enough for **enterprise-grade RAG systems in regulated domains** because it ensures your knowledge base remains synchronized with authoritative sources while preserving historical versions for traceability.
