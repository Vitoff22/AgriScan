# AgriScan.ai — Cinematic Trailer

A 42-second cinematic trailer for the AgriScan.ai mobile app.

## Setup

```bash
npm ci
npm run start
```

## Render

```bash
npm run render
```

Output: `out/trailer.mp4`

## Dependency maintenance

All `remotion` and `@remotion/*` packages use the same exact version. Keep them in sync when upgrading, commit `package-lock.json`, then run:

```bash
npm audit
npx tsc --noEmit
npx remotion bundle src/index.ts --out-dir=out/check
```

The trailer is a local creation tool, not the FlutterFlow application or its backend. Keep the development studio private. The audited lockfile was updated on 2026-10-13 to address the reported dependency advisories; repeat the audit when dependencies or advisory data change. Backend security requirements and validation limits are documented in [`../../security/README.md`](../../security/README.md).
