# DocuBot — AI-Powered PDF Reader for Educators

## 1. Project Background

A teacher came to me with a problem I hear constantly in education: too many PDFs, not enough time.

Between lesson plans, research papers, curriculum guides, district policy documents, and student resources, educators spend hours manually hunting through documents for specific information. A teacher preparing a unit on a new topic might have 10–15 PDFs open at once, skimming pages looking for a single paragraph that answers a specific question.

I built DocuBot to solve that problem. Upload any PDF — a textbook chapter, a research study, an IEP document, a school policy guide — and ask questions about it in plain English. DocuBot reads the entire document and answers instantly, grounded in exactly what that document says. No more scrolling. No more Ctrl+F. No more reading pages you do not need.

The goal was to keep it dead simple for a non-technical user: one file upload, one text box, one button. The AI handles everything else.

---

## 2. What DocuBot Does

### The Teacher's Workflow — Before and After

**Before DocuBot:**
A teacher receives a 47-page district curriculum guide. She needs to know the specific learning objectives for Grade 5 Science Unit 3. She opens the PDF, searches for keywords, scrolls through three sections, finds a table, reads surrounding context to confirm it is the right one. Five minutes for one answer.

**With DocuBot:**
She uploads the PDF, types *"What are the learning objectives for Grade 5 Science Unit 3?"* and reads the answer in seconds — with the exact page number it came from.

That is the entire pitch. The technology underneath is sophisticated. The experience should feel effortless.

---

## 3. How It Works

DocuBot is built on three components: **PyMuPDF** extracts the text from the uploaded PDF, **Claude AI** reads and understands it, and **Gradio** provides the simple two-panel interface teachers interact with.

```
Teacher uploads PDF
        │
        ▼
┌──────────────────────────────┐
│   PyMuPDF reads the document │  — Every page extracted with [Page N] label
│   page by page               │  — Preserves reading order and structure
└──────────────────────────────┘
        │
        ▼
┌──────────────────────────────┐
│   Teacher types a question   │  — Plain English, any question
└──────────────────────────────┘
        │
        ▼
┌──────────────────────────────┐
│   Claude AI receives:        │  — Full document text
│   • The entire document      │  — Full conversation history
│   • The question asked       │  — Clear instruction to stay on-document
│   • Prior conversation       │
└──────────────────────────────┘
        │
        ▼
┌──────────────────────────────┐
│   Answer returned to teacher │  — Grounded in the document
│   in the chat panel          │  — Cites page where relevant
│                              │  — Honest when answer isn't there
└──────────────────────────────┘
```

The key design decision: the **entire document** is passed to Claude on every question, not just a retrieved excerpt. This means Claude can answer questions that span multiple sections, compare information from different parts of the document, or summarize across the whole thing — not just match keywords to a nearby paragraph.

---

## 4. Built For Teachers

Every design choice in DocuBot was made with a non-technical user in mind.

### The Interface Is Two Things: Upload and Ask

![DocuBot Interface](https://github.com/YSayaovong/PDF-AI-Chatbot-/blob/main/assets/chatbot.png?raw=true)

The left panel has one job: accept a PDF. The right panel has one job: let you chat. There are no settings to configure, no model parameters to tune, no prompt templates to fill out. Upload the file, ask the question, get the answer.

### It Remembers the Conversation

A teacher does not ask one question and leave. She asks a follow-up. Then another. DocuBot maintains the full conversation history across every turn — so she can ask *"What does the document say about differentiated instruction?"* and then follow up with *"Can you give me an example from the text?"* and Claude understands the context without being reminded.

### It Does Not Make Things Up

Claude is explicitly instructed: *"Base your answers on the document contents provided. If the answer is not in the document, say so clearly and helpfully."* If a teacher asks about something the document does not cover, DocuBot says so — it does not invent a plausible-sounding answer. For educators making curriculum and policy decisions, that honesty is not optional.

### It Tells You Where to Look

The PDF extraction labels every page — `[Page 1]`, `[Page 2]`, and so on — and Claude receives that structure. When it references information, it can tell the teacher which page it came from, so she can go verify the original source herself if needed.

![Why Full-Document Context Matters](https://github.com/YSayaovong/PDF-AI-Chatbot-/blob/main/assets/difference.png?raw=true)

---

## 5. Classroom Use Cases

DocuBot was built general enough to handle any PDF a teacher might work with:

| Document Type | Example Question |
|---|---|
| Textbook chapters | "Summarize the main causes of the Civil War from this chapter." |
| Curriculum guides | "What are the Grade 4 Math standards covered in Unit 2?" |
| Research papers | "What does this study conclude about reading intervention strategies?" |
| IEP documents | "What accommodations are listed for this student?" |
| School policy guides | "What is the attendance policy for excused absences?" |
| Lesson plan templates | "What materials are needed for the Week 3 science activity?" |
| Professional development resources | "What does this PD guide recommend for formative assessment?" |

Any document a teacher already has — no reformatting, no preparation, just upload and ask.

---

## 6. What I Would Build Next

DocuBot solves the core problem well. Here is what I would add to make it more powerful for educators specifically:

- **Multi-document comparison.** Teachers often have the old curriculum guide and the new one side by side. Uploading both and asking *"What changed between the 2022 and 2024 versions of this standard?"* would be immediately useful. This requires extending the file upload to accept multiple PDFs and labeling each document's content separately so Claude can distinguish sources.

- **Automatic document summary on upload.** The first thing a teacher wants when she uploads a new document is orientation — what is this, what does it cover, how long is it? Triggering an automatic summary the moment a PDF is uploaded would replace the mental overhead of skimming the table of contents.

- **Saved sessions by document.** Right now, closing the tab ends the conversation. Saving chat history tied to each document — so a teacher can pick up where she left off on the same curriculum guide tomorrow — would make DocuBot a long-term research assistant rather than a one-session tool.

- **Export answers to a note document.** A teacher finds three important passages across a 60-page research paper. She wants to paste them into her lesson plan. Adding an export button that compiles the conversation Q&A pairs into a clean Word or PDF document would eliminate the manual copy-paste step.

- **Classroom-safe deployment.** The current setup runs locally with a personal API key. For school-wide use, wrapping this in a simple authenticated web app — one login per teacher, no API key management required — would make it deployable across an entire department or district without requiring any technical setup from the end users.

---

## Tools Used

- Python
- Gradio — two-panel web interface, instant public sharing
- Anthropic SDK — Claude (`claude-sonnet-4-20250514`) for document Q&A
- PyMuPDF (`fitz`) — fast, reliable PDF text extraction with page-level structure
- Git / GitHub — version control

---

## Disclaimer
This project was developed as an independent use case study based on direct experience. No proprietary data, customer specifications, or internal company documents were used. The workflow analysis, KPIs, and ROI model are based on observed process patterns and industry-representative estimates.
