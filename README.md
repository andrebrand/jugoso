# jugoso

jugoso is a small Spanish pronunciation studio built with Vue 3, TypeScript, and Vite. Choose syllables from the word library to preview their videos, arrange them into a sequence, and share the mix with a link.

## Run locally

```sh
pnpm install
pnpm dev
```

Create a production build with `pnpm build`, or preview it locally with `pnpm preview`.

## Add words

Edit [`src/data/words.json`](./src/data/words.json). Each word has a `syllable` array; every syllable needs an English-style phonetic `label` and a video `link`:

```json
{
  "jugoso": {
    "syllable": [
      {
        "label": "hoo",
        "link": "https://example.com/jugoso-syllable-1.mp4"
      },
      {
        "label": "goh",
        "link": "https://example.com/jugoso-syllable-2.mp4"
      }
    ]
  },
  "hola": {
    "syllable": [
      {
        "label": "oh",
        "link": "https://example.com/hola-syllable-1.mp4"
      }
    ]
  }
}
```

The starter catalog also includes `jugo`, reusing the matching `ju` and `go` syllable clips from `jugoso`. Use publicly accessible MP4 links that allow playback on your site. The app uses the word key and syllable position to save selections, so keep word keys and syllable order stable if you want existing shared links to keep working.

## Share links

A selection is stored in the URL hash as compact base64url-encoded data, so the chosen words are not plainly shown in the address. This is encoding, not encryption. When a shared link opens, jugoso attempts to play the sequence and reveals the syllables after playback; browsers that block autoplay show a play prompt instead.

## GitHub Pages

The included GitHub Actions workflow builds and deploys the `jugoso` repository to GitHub Pages on pushes to `main`. In the repository settings, set **Pages → Build and deployment → Source** to **GitHub Actions**.
