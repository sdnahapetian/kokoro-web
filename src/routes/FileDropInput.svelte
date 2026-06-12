<script lang="ts">
  import { toaster } from "$lib/client/toaster";
  import { Upload, Loader } from "lucide-svelte";

  interface Props {
    ontext: (text: string) => void;
  }

  let { ontext }: Props = $props();

  let dragging = $state(false);
  let processing = $state(false);

  const ACCEPTED = [".txt", ".md", ".csv", ".srt", ".pdf", ".docx"];
  const MAX_BYTES = 50_000_000; // 50 MB

  async function extractPdfText(file: File): Promise<string> {
    const pdfjsLib = await import("pdfjs-dist");
    pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
      "pdfjs-dist/build/pdf.worker.min.mjs",
      import.meta.url
    ).toString();

    const arrayBuffer = await file.arrayBuffer();
    const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;

    const pages: string[] = [];
    for (let i = 1; i <= pdf.numPages; i++) {
      const page = await pdf.getPage(i);
      const content = await page.getTextContent();
      const pageText = content.items
        .map((item: any) => ("str" in item ? item.str : ""))
        .join(" ")
        .replace(/\s+/g, " ")
        .trim();
      if (pageText) pages.push(pageText);
    }

    return pages.join("\n\n");
  }

  async function extractDocxText(file: File): Promise<string> {
    const mammoth = await import("mammoth");
    const arrayBuffer = await file.arrayBuffer();
    const result = await mammoth.extractRawText({ arrayBuffer });
    return result.value;
  }

  async function readFile(file: File) {
    if (file.size > MAX_BYTES) {
      toaster.error("File too large (max 50 MB)");
      return;
    }

    const ext = ("." + file.name.split(".").pop()?.toLowerCase()) as string;
    if (!ACCEPTED.includes(ext)) {
      toaster.error(`Unsupported file type. Accepted: ${ACCEPTED.join(", ")}`);
      return;
    }

    processing = true;
    try {
      let text: string;

      if (ext === ".pdf") {
        text = await extractPdfText(file);
      } else if (ext === ".docx") {
        text = await extractDocxText(file);
      } else {
        text = await file.text();
      }

      if (!text.trim()) {
        toaster.error("File appears to be empty or has no extractable text");
        return;
      }

      ontext(text);
      toaster.success(`Loaded "${file.name}"`);
    } catch (err) {
      console.error(err);
      toaster.error("Failed to read file — see console for details");
    } finally {
      processing = false;
    }
  }

  function onDrop(e: DragEvent) {
    e.preventDefault();
    dragging = false;
    const file = e.dataTransfer?.files?.[0];
    if (file) readFile(file);
  }

  function onDragOver(e: DragEvent) {
    e.preventDefault();
    dragging = true;
  }

  function onDragLeave() {
    dragging = false;
  }

  function onFileInput(e: Event) {
    const file = (e.target as HTMLInputElement).files?.[0];
    if (file) readFile(file);
    (e.target as HTMLInputElement).value = "";
  }

  function triggerPicker() {
    if (!processing) document.getElementById("file-drop-input")?.click();
  }
</script>

<div
  role="button"
  tabindex="0"
  class={{
    "border-base-content/20 hover:border-base-content/40 flex cursor-pointer items-center justify-center gap-3 rounded-lg border-2 border-dashed px-4 py-3 transition-colors": true,
    "border-primary bg-primary/10": dragging,
    "cursor-wait opacity-60": processing,
  }}
  ondrop={onDrop}
  ondragover={onDragOver}
  ondragleave={onDragLeave}
  onclick={triggerPicker}
  onkeydown={(e) => e.key === "Enter" && triggerPicker()}
>
  {#if processing}
    <Loader class="text-base-content/50 size-5 shrink-0 animate-spin" />
    <span class="text-base-content/60 text-sm">Extracting text…</span>
  {:else}
    <Upload class="text-base-content/50 size-5 shrink-0" />
    <span class="text-base-content/60 text-sm">
      {dragging ? "Drop to load" : "Drop a file or click to browse"}
      <span class="text-base-content/40 ml-1">({ACCEPTED.join(", ")})</span>
    </span>
  {/if}
  <input
    id="file-drop-input"
    type="file"
    accept={ACCEPTED.join(",")}
    class="hidden"
    onchange={onFileInput}
  />
</div>
