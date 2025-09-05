Here is the fixed README. Paste it directly into `README.md`.

# JSON Knowledge Base Chatbot (Colab)

A Colab notebook that answers questions against a JSON or JSON-LD knowledge base. It finds the most relevant entries using embeddings and simple keywords, fetches the linked PDFs on demand, extracts text, and builds a reply. You can edit the system prompt and add a bias theme.

---

## Contents

* [Why it is built this way](#why-it-is-built-this-way)
* [Quick start](#quick-start)
* [What the JSON looks like](#what-the-json-looks-like)
* [How relevance is chosen](#how-relevance-is-chosen)
* [What happens when you ask a question](#what-happens-when-you-ask-a-question)
* [Scaling considerations](#scaling-considerations)
* [Notes on PDF handling](#notes-on-pdf-handling)
* [Data handling](#data-handling)
* [Troubleshooting](#troubleshooting)
* [Extending](#extending)

---

## Why it is built this way

Large archives change over time. Files appear and move. Pre-building a vector database for every page can become a maintenance task that never ends. This notebook aims for a lighter path.

* **Fetch at query time.** The JSON holds links to source PDFs. The notebook scores entries for a given question, then fetches only the few PDFs that look relevant. This avoids a heavy ingestion pipeline and removes the need to keep a separate vector index in sync with the archive.
* **Summaries rather than full text for indexing.** Each entry has a short summary. Embeddings are created for these summaries once per session. Summaries are cheap to embed and quick to compare. The heavy lifting happens only for the top few documents that the query selects.
* **Whole-document context rather than pre-cut chunks.** Many retrieval systems split every PDF into fragments and embed each fragment. That approach can confuse meaning when the original order matters. Here, the notebook pulls the text of the selected PDFs and sends contiguous spans up to a character limit per document. Order is preserved inside each span.
* **Simple and portable.** Everything runs in one notebook. There is no separate service, database, or queue. The JSON can live in a repo or a drive folder. If a URL points at a GitHub blob, the notebook converts it to the raw file automatically.

---

## Quick start

1. Open the notebook in Google Colab.
2. Run the first cell to install dependencies.
3. Enter your OpenAI API key in the password field and click **Save API Key**.
4. Click **Proceed**.
5. Upload your JSON or JSON-LD knowledge base file when prompted.
6. Wait while the notebook builds embeddings for entry summaries.
7. Use the chat box to ask a question.

   * The notebook shows cards for the entries it selects.
   * It fetches those PDFs and extracts text.
   * The assistant replies using the text and your system prompt.
8. Adjust sliders and checkboxes to control:

   * **Top K** entries to pull
   * **Keyword weight**
   * **Chars/doc** text cap
   * **OCR fallback** for scanned PDFs
   * **Memory stats** for debugging

---

## What the JSON looks like

The key field is the **PDF URL**. Each entry should include a `summary` and a link to a PDF. Other metadata fields are optional but help with scoring and display.

Example:

```json
[
  {
    "filename": "page-01.pdf",
    "summary": "Bulletin of the Computer Arts Society, April 1963. Events, meetings, exhibitions, and contacts.",
    "url": "https://github.com/seanzshow1/CAAData/blob/main/page-01.pdf"
  },
  {
    "filename": "page-02.pdf",
    "summary": "Public meetings, weekend course, contest deadlines, exhibitions, and society aims.",
    "url": "https://github.com/seanzshow1/CAAData/blob/main/page-02.pdf"
  }
]
```

**Fields the notebook understands:**

* `summary` → used for embeddings
* `url` → should point to a PDF (GitHub blob links are supported)
* `filename`, `title`, `name`, `date`, `datePublished`, `author`, `keywords` → optional, improve display and scoring

The loader also supports JSON-LD containers such as `@graph`, `graph`, `items`, `entries`, `documents`, and `records`.

---

## How relevance is chosen

1. The query is embedded with `text-embedding-3-small`.
2. Cosine similarity is computed against the pre-built summary embeddings.
3. A keyword score is added by counting query tokens across entry fields.
4. Scores are combined and the top **K** entries are selected.

---

## What happens when you ask a question

* Cards are shown for the selected entries.
* PDFs are fetched and parsed with **PyMuPDF**.
* If text is missing and OCR is enabled, **Tesseract** runs.
* Each document is truncated to your character cap.
* The text is sent to `gpt-4o-mini` with your system prompt and query.
* The assistant’s reply appears in the chat.

---

## Scaling considerations

* Thousands of entries can be loaded in one JSON. Only summaries are embedded up front.
* Costs scale with number of entries (embeddings) and with size and number of PDFs fetched.
* Repeated queries reuse cached PDF bytes.

---

## Notes on PDF handling

* PyMuPDF is used first for extraction.
* OCR fallback uses `pdf2image` and `tesseract`. It is slower and heavier.
* GitHub `blob` links are rewritten to `raw` links.

---

## Data handling

* API key lives only in the Colab session.
* Model input is system prompt, query, and extracted snippets.
* PDFs are fetched on demand via HTTPS.
* No external storage unless you download manually.

---

## Troubleshooting

* **Key error** → Enter and save API key.
* **401** → Check key, billing, and model access.
* **Invalid JSON** → Ensure valid UTF-8 and correct containers.
* **Empty PDF text** → Enable OCR fallback.
* **Slow replies** → Reduce Top K or character cap.

---

## Extending

* Change embedding or chat model.
* Swap keyword count for a better extractor.
* Add citations with page indices.
* Persist embeddings in a store if cross-session memory is needed.

