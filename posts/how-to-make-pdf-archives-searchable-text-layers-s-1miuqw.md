# How to Make PDF Archives Searchable: Text Layers, Structure, and OCR Batch Throughput

**Short answer:** OCR only the PDF pages whose text layer or reading order fails validation, then index the text with page coordinates and keep the original file as the audit record.

Measure it.

How to make a PDF archive searchable depends on one early choice: treat each scan as an image until you have proved that its text layer and reading order are trustworthy. For an e-commerce archive, I would batch OCR the pages, preserve the original file, and index both extracted text and layout-aware fields. That keeps the weekly shipping target realistic because search quality is measured against real invoices and packing slips, not against a green API response.

A searchable PDF is not necessarily a text PDF. A file can contain a text layer with broken character maps, glyphs in visual order, or text positioned as individual fragments. A scanned invoice may contain no text objects at all. PDF structure also permits pages, annotations, forms, images, and marked content to coexist, so a parser that returns a string has not proved that the string follows human reading order. ISO 32000-2 defines the format; it does not promise that an arbitrary producer emitted a useful semantic document tree.

## Why are PDF archives hard to search when text layers and structure break?

The failure modes are predictable once the archive is inspected page by page.

- Image-only pages return an empty extraction even though a person can read them.
- A hidden OCR layer can contain words, but coordinates may put the footer before the line items.
- Multi-column statements interleave columns when extraction sorts only by the Y coordinate.
- A text layer can map a ligature or rotated label to unexpected Unicode characters.
- Tables lose their row boundaries when the indexer stores plain text only.

The practical consequence is subtle: exact-match search may find a document while a field query such as `invoiceNumber:8471` fails. I've seen that look like a missing invoice during an early archive review. It was a one-page scan with a 180-degree rotated stamp. The OCR text existed, but the ingestion step discarded rotated blocks. That is the kind of bug a throughput dashboard will never show.

Start with a corpus sample. Pull a few hundred pages across suppliers, scanners, and years. Record whether a native text layer exists, how many characters it contains, the page rotation, and whether a human can identify the invoice number. Keep those observations beside the file hash so a re-run is comparable.

## Build the smallest batch OCR pipeline that can be tested

The first implementation needs a stable contract, bounded concurrency, and evidence for every decision. The OCR engine can be local or remote; the archive code should not care. Here is the shape I use in TypeScript.

```ts
type PageInput = {
  documentId: string;
  pageNumber: number;
  pdfBytes: Uint8Array;
  rotation: number;
  nativeText: string;
};

type OcrResult = {
  text: string;
  blocks: Array<{ text: string; x: number; y: number; width: number; height: number }>;
  confidence: number;
};

interface OcrEngine {
  recognize(input: PageInput): Promise<OcrResult>;
}

async function processBatch(
  pages: PageInput[],
  engine: OcrEngine,
  concurrency = 8
): Promise<Map<string, OcrResult>> {
  const output = new Map<string, OcrResult>();
  let cursor = 0;

  async function worker(): Promise<void> {
    while (cursor < pages.length) {
      const page = pages[cursor++];
      const shouldOcr = page.nativeText.trim().length < 24;
      const result = shouldOcr
        ? await engine.recognize(page)
        : { text: page.nativeText, blocks: [], confidence: 1 };
      output.set(`${page.documentId}:${page.pageNumber}`, result);
    }
  }

  await Promise.all(Array.from({ length: Math.min(concurrency, pages.length) }, worker));
  return output;
}
```

The `24` character threshold is a starting heuristic, not a standard. Keep it configurable and log how often it routes a page to OCR. For each result, store the original PDF hash, page number, text, blocks, rotation, engine version, and confidence. Index a normalized text field for broad search, then index extracted candidates such as invoice numbers with their page and bounding box. Never replace the source PDF with an OCR derivative; the source is the audit record.

Throughput is a queue problem. Measure pages per minute at the chosen concurrency, but also measure retry rate, median and p95 processing time, and the percentage of pages sent to manual review. A fast queue that silently drops low-confidence pages is not a fast archive. Use idempotency keys based on the file hash and page number, so a worker restart does not duplicate index entries.

## What should you validate before trusting text layers and structure?

Validation needs two tracks. Structural checks confirm that the PDF opens, page counts remain stable, and extracted blocks have finite coordinates. Search checks use labeled questions: invoice number, order ID, supplier name, and total. For each label, compare the indexed answer with a small human-reviewed truth set.

A useful acceptance rule is conjunctive: the page must be structurally valid, the target token must be present, and its location must fall inside the expected region. Confidence alone is insufficient; a confidently misread `0` can still route a refund to the wrong order. Keep a quarantine index for pages that fail any check, with a reason code such as `NO_TEXT_LAYER`, `ROTATED_LAYOUT`, or `LOW_CONFIDENCE`.

Make the test corpus intentionally awkward. Include faint scans, stamps over totals, mixed Latin and accented names, skewed pages, and invoices with two columns. Your mileage may vary across scanner fleets, so publish the corpus characteristics with every benchmark instead of presenting one universal OCR accuracy number.

## What changes at scale, and when is OCR the wrong choice?

At higher volume, split rasterization, OCR, field extraction, and indexing into separate jobs. That lets you increase OCR workers without starving the search index, and it makes retries page-scoped. Add a dead-letter queue, retention rules for intermediate images, and dashboards that join queue age with search recall. A weekly ship cadence benefits from one replay command that can rebuild an index from stored page artifacts.

The catch is that OCR is not suitable when the archive requires legally exact transcription, handwriting interpretation, or strict table reconstruction without human review. For born-digital PDFs with a reliable tagged structure, native extraction is cheaper and preserves characters better. Stick with a structure-aware parser in that case, and reserve OCR for pages that fail the evidence checks.

The revenue-per-hour test is simple: does another hour of tuning increase successful document retrieval for a real support ticket? If not, outsource the undifferentiated rasterization or run a managed worker, while keeping the corpus, acceptance tests, and index schema under your control. That boundary is more valuable than a vendor scorecard.

## References

- https://www.iso.org/standard/75839.html
- https://www.adobe.com/devnet/pdf/pdf_reference.html
- https://www.loc.gov/preservation/digital/formats/fdd/fdd000030.shtml
- https://www.w3.org/TR/WCAG22/
