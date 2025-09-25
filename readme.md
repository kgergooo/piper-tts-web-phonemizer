# Piper Web Phonemizer with Per-Language Datasets

[PiperTTS voices on Hugging Face](https://huggingface.co/rhasspy/piper-voices)

This project optimizes the phonemizer assets that ship with [Mintplex-Labs/piper-tts-web](https://github.com/Mintplex-Labs/piper-tts-web), a web API that runs PiperTTS models in the browser with WebAssembly. The original build exposes a single `piper_phonemizer.data` (≈18 MB) that bundles every language. Here we split that dataset per language so browsers only download and mount the files they actually need.

## Shout-out

Huge thanks to the team behind [Mintplex-Labs/piper-tts-web](https://github.com/Mintplex-Labs/piper-tts-web) for building the browser-ready PiperTTS experience this project builds upon.

## Why split the dataset?

- Each voice only requires the espeak-ng resources for its language, but the original package included all of them, increasing download time and memory usage.
- Serving language-specific `.data` files keeps the payload small and dramatically speeds up phonemizer initialization in the browser.

## How it works

- Every language has its own `piper_phonemize-<lang>.data` file plus a JSON manifest generated from the original Emscripten bundle (Max 1MB).
- The manifest lists each virtual file (`filename`, `start`, `end`) so the loader can mount that language inside the in-memory filesystem when the phonemizer starts.
- At runtime you fetch the manifest, pass it to `createPiperPhonemize`, and supply a `locateFile` handler that resolves the per-language `.data` file.

## Usage (TypeScript/Angular)

```typescript
import { BehaviorSubject, firstValueFrom, from, Observable, of } from "rxjs";
import { finalize, switchMap } from "rxjs/operators";
import { createPiperPhonemize } from "./piper_phonemizer.js";

let phonemizer: any;
const phonemizerMessage$ = new BehaviorSubject<number[] | null>(null);

async function initPhonemizer(language: string) {
  const lang = language ?? "en";
  const dataFileUrl = `/assets/language-data/piper_phonemize-${lang}.data`;
  const manifestUrl = `/assets/language-data/piper_phonemize-${lang}.json`;

  const manifestResponse = await fetch(manifestUrl);
  const manifest = await manifestResponse.json();

  phonemizer ??= await createPiperPhonemize({
    language: lang,
    phonemizerManifest: manifest,
    locateFile: (url: string) => {
      if (url.endsWith(".wasm")) return "/assets/piper_phonemize.wasm";
      if (url.endsWith(".data")) return dataFileUrl;
      return url;
    },
    print: (msg: string) => {
      const phonemes = JSON.parse(msg)?.phoneme_ids ?? null;
      phonemizerMessage$.next(phonemes);
    },
    printErr: () => phonemizerMessage$.next(null),
  });

  await phonemizer.ready;
  return phonemizer;
}

function phonemize(text: string, voiceId: string): Observable<number[] | null> {
  const eSpeakVoiceId = voiceId.split("_")[0];

  return from(initPhonemizer(eSpeakVoiceId)).pipe(
    switchMap((module: any) => {
      if (!module) return of(null);

      const input = JSON.stringify([{ text: text.trim() }]);
      module.callMain([
        "-l",
        eSpeakVoiceId,
        "--input",
        input,
        "--espeak_data",
        "/espeak-ng-data",
      ]);
      return phonemizerMessage$.asObservable();
    }),
    finalize(() => phonemizerMessage$.next(null))
  );
}

const phonemes = await firstValueFrom(phonemize("Text", "hu_HU-anna-medium"));
```

Hope that is useful. Enjoy it.
