# Secret Santa

A small static Vue 3 + TypeScript app for generating one-time Secret Santa assignments and printable individual reveal links.

## Run locally

```bash
npm install
npm run dev
```

## Build

```bash
npm run build
npm run preview
```

## How it works

- The organizer enters participant names and generates a randomized no-self-assignment drawing.
- The drawing is saved in this browser's `localStorage`.
- Each participant's recipient name is UTF-8 encoded as Base64URL in a `?gift=` query parameter.
- The distribution sheet prints each participant's name and individual reveal URL.
- Opening a reveal URL decodes and displays only the recipient name encoded in that URL.

## Important

This is intentionally obfuscation, not security. Anyone who sees a link can decode its recipient. The organizer's browser also retains the full drawing in localStorage. Keep the printed distribution sheet private.

The app has no backend and sends no data anywhere. Deploy the contents of `dist/` to any static host. The site must be hosted at a stable URL before generating links that will be distributed.
