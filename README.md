# DocuBot — PDF Q&A Chatbot Powered by Claude AI

## 1. Project Background

As a developer applying AI to a practical document problem, I built DocuBot to eliminate the most common friction point in working with PDFs: *you have a document, you have a question, and reading the whole file to find the answer is not a reasonable use of time.*

Standard LLMs can answer general knowledge questions, but they cannot answer questions about a specific document you uploaded today. DocuBot bridges that gap — it reads the PDF you provide, passes the full extracted text alongside your question to Claude, and returns answers that are grounded in that specific document's content. If the answer is not in the document, it says so clearly rather than hallucinating a response.

The system is built on three components working together: **PyMuPDF** for reliable page-by-page text extraction, **Claude (claude-sonnet-4-20250514)** as the reasoning engine, and **Gradio** as the shareable web interface. The full pipeline runs in a single Python file with no vector database, no embedding model, and no retrieval infrastructure — the entire extracted document is passed to Claude on every turn, letting the model reason over the full context directly.

The project is designed around two core capabilities:

- **Full-document comprehension:** Every page of the PDF is extracted with page number labels and passed as structured context on every API call — Claude always has the complete picture, not a retrieved fragment
- **Multi-turn conversation:** The full conversation history is maintained across turns, enabling follow-up questions, clarifications, and references to prior answers without the user re-stating context

---

## 2. System Architecture & Data Flow

DocuBot uses a direct context injection approach rather than RAG (Retrieval-Augmented Generation). Instead of embedding and retrieving chunks, the entire extracted document is included in every API call. This is the correct design choice for typical PDF lengths where full context fits within Claude's context window.

```
User Action
──────────────────────────────────────────────────────────────
    │
    ▼
┌─────────────────────────────────────┐
│         Gradio Web Interface         │
│  - PDF file upload (left panel)      │
│  - Chat input + history (right panel)│
│  - Clear chat button                 │
└─────────────────────────────────────┘
    │
    ▼  PDF filepath passed to extract_pdf_text()
┌─────────────────────────────────────┐
│         PyMuPDF Extraction           │  fitz.open() → page.get_text()
│  - Opens PDF via fitz                │
│  - Iterates all pages                │
│  - Labels each page: [Page N]        │
│  - Joins into single text string     │
└─────────────────────────────────────┘
    │
    ▼  doc_text injected into message
┌─────────────────────────────────────┐
│         Message Construction         │
│  - If PDF present:                   │
│    "Here are its contents: {doc_text}│
│     User question: {user_message}"   │
│  - If no PDF: prompt to upload one   │
└─────────────────────────────────────┘
    │
    ▼  Full conversation history + current message
┌─────────────────────────────────────┐
│         Anthropic Claude API         │
│  Model: claude-sonnet-4-20250514     │
│  Max tokens: 1,024                   │
│  System prompt: DocuBot persona +    │
│    document-grounding instruction    │
│  Messages: history + current turn    │
└─────────────────────────────────────┘
    │
    ▼  response.content[0].text
┌─────────────────────────────────────┐
│         Gradio Chat Display          │
│  - Bot reply appended to history     │
│  - Input box cleared automatically   │
│  - State synced for next turn        │
└─────────────────────────────────────┘
```

| Component | Role | Implementation |
|---|---|---|
| Gradio Blocks | Web UI — upload panel + chat panel | `gr.Blocks`, `gr.File`, `gr.Chatbot`, `gr.Textbox` |
| PyMuPDF (`fitz`) | PDF text extraction with page labels | `fitz.open()` → `page.get_text()` |
| Message builder | Injects doc context into every user turn | String formatting with doc_text + user_message |
| Anthropic client | LLM API call with conversation history | `client.messages.create()` |
| Gradio State | Persists chat history across turns | `gr.State([])` synced on every `.then()` chain |

---

## 3. Code Walkthrough

### PDF Extraction — `extract_pdf_text()`

```python
def extract_pdf_text(pdf_path):
    doc = fitz.open(pdf_path)
    pages = []
    for i, page in enumerate(doc):
        pages.append(f"[Page {i+1}]\n{page.get_text()}")
    doc.close()
    return "\n\n".join(pages).strip()
```

Each page is labeled `[Page N]` before extraction. This means Claude receives structured, page-aware context — when it references where information came from, it can cite a specific page number rather than returning an unattributed answer. The full text is returned as a single string, joining all pages with double newlines for readability.

### Message Construction and Context Injection

```python
if doc_text:
    full_message = (
        f"The user has uploaded a PDF document. Here are its contents:\n\n"
        f"{doc_text}\n\n"
        f"---\nUser question: {user_message}"
    )
else:
    full_message = user_message + "\n\n(No PDF uploaded yet — please upload one using the panel on the left.)"
```

The document text is injected into every user turn, not just the first one. This ensures Claude always has full document context regardless of where in the conversation the question falls — the model cannot "forget" the document between turns. When no PDF is uploaded, the system prompts the user to upload one rather than attempting to answer without context.

### Conversation History — Anthropic Message Format

```python
messages = []
for human, bot in history:
    messages.append({"role": "user",      "content": human})
    messages.append({"role": "assistant", "content": bot})
messages.append({"role": "user", "content": full_message})
```

Gradio stores history as `[[human, bot], [human, bot], ...]` pairs. This loop converts that format into the alternating `user`/`assistant` message list that the Anthropic API expects, then appends the current turn. The result is a complete, ordered conversation that Claude can reason over — prior questions and answers are fully visible when generating the next response.

### System Prompt — DocuBot Identity and Grounding Instruction

```python
system=(
    "You are DocuBot, a helpful assistant that answers questions about uploaded PDF documents. "
    "Base your answers on the document contents provided. "
    "If the answer is not in the document, say so clearly and helpfully."
)
```

Three directives in one system prompt: a defined persona (DocuBot), a grounding instruction (base answers on the document), and a hallucination prevention rule (say so if the answer is not there). The grounding instruction is what separates this from a general-purpose Claude conversation — the model is explicitly instructed to treat the injected document as the authoritative source.

---

## 4. Interface Design

The UI is built with `gr.Blocks` and a custom CSS layer, organized into a two-panel layout:

- **Left panel (scale=1):** PDF upload component accepting `.pdf` files, hint text, and the Clear Chat button
- **Right panel (scale=2):** Full-height chatbot display, text input box, and Send button

Both the Send button click and the Enter key (`user_input.submit`) are wired to the same `chat()` function. Each event chain uses `.then()` to synchronize the Gradio State with the chatbot output and clear the input box after each submission — standard Gradio multi-step event handling.

![DocuBot Interface](https://github.com/YSayaovong/PDF-AI-Chatbot-/blob/main/assets/chatbot.png?raw=true)

![Direct Context Injection vs RAG](https://github.com/YSayaovong/PDF-AI-Chatbot-/blob/main/assets/difference.png?raw=true)

---

## 5. Design Decisions

### Direct Context Injection vs. RAG

The `difference.png` asset illustrates this choice. A RAG pipeline — embedding chunks, storing vectors, retrieving by similarity — is the right architecture when a document is too large to fit in the model's context window, or when querying across many documents simultaneously. For typical PDFs (reports, papers, contracts), full-document context fits comfortably within Claude's context window.

Direct injection is simpler, more reliable, and produces better answers for this use case: Claude reasons over the complete document rather than a retrieved subset, which means questions that draw on multiple sections — comparisons, summaries, conditional logic across paragraphs — are answered correctly rather than being limited by retrieval precision.

### Why Claude (Anthropic) Over OpenAI

Claude's large context window and instruction-following reliability make it well-suited for document Q&A. The model consistently follows the grounding instruction ("base your answers on the document") and correctly identifies when a question falls outside the document scope — reducing hallucination without additional prompt engineering.

### Why PyMuPDF (`fitz`) Over Other PDF Libraries

PyMuPDF is faster and more reliable than `PyPDF2` for extracting text from complex PDFs — it handles multi-column layouts, embedded fonts, and scanned-document OCR modes more gracefully. The page-by-page iteration with explicit `[Page N]` labeling is a deliberate design choice: it gives Claude spatial context within the document that page-agnostic extraction strips away.

---

## 6. Recommendations & Next Steps

Based on the current architecture, the following enhancements would meaningfully extend DocuBot's capability and production-readiness:

- **Add source page citation to responses.** The extracted text already contains `[Page N]` labels that Claude can reference. Updating the system prompt to explicitly instruct Claude to cite page numbers when answering would give users a direct path to verify answers in the original document — critical for legal, financial, and research use cases.

- **Implement document-length handling for very large PDFs.** Direct context injection works well for typical documents, but PDFs exceeding Claude's context window (very long reports, book-length documents) will truncate. Adding a character count check and a chunked summarization fallback for oversized documents would make the system robust to edge cases without requiring a full RAG rewrite.

- **Persist conversation history to a session file.** Currently all chat history is lost when the browser tab is closed. Writing history to a local JSON file keyed by document hash would allow users to resume previous sessions — particularly useful for long research or review workflows where the same document is revisited multiple times.

- **Support multi-PDF upload.** The current `gr.File` component accepts one file. Extending this to `gr.File(file_count="multiple")` and concatenating extracted text with document-level labels (`[Document: filename.pdf]`) would let users query across multiple related documents simultaneously — comparing two contracts, cross-referencing multiple reports, or querying a set of research papers together.

- **Add a document summary on upload.** When a PDF is first uploaded, triggering an automatic one-paragraph summary ("Here is what this document covers: ...") would orient the user before they start asking questions, reduce exploratory queries, and confirm that the extraction succeeded and the content is readable.

- **Stream responses for long answers.** The current implementation waits for the full API response before displaying it. Using `client.messages.stream()` instead of `client.messages.create()` would let tokens appear in the chat interface progressively — improving perceived responsiveness for long, detailed answers.

---

## Tools Used

- Python
- Gradio (two-panel web UI, shareable public URL)
- Anthropic Python SDK (Claude API — `claude-sonnet-4-20250514`)
- PyMuPDF / `fitz` (page-by-page PDF text extraction with page labels)
- Git / GitHub (version control)
