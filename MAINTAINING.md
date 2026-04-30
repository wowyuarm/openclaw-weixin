# OpenClaw Weixin local maintenance

This repo keeps a small self-use patch layer on top of the official `@tencent-weixin/openclaw-weixin` npm package.

## Branches

- `vendor/npm-2.3.1`: exact official npm package import.
- `yu/base/npm-2.3.1-dev`: vendor branch plus `tsconfig.json` so local checks can run.
- `yu/patch/*`: one topic branch per patch.
- `yu/current`: merge branch used locally. Patch branches are merged with `--no-ff` so the included patch set is easy to see.

## Current patch branches

- `yu/patch/nonblocking-monitor`: keep polling while inbound message handling is still running; lets OpenClaw `messages.queue.mode = "steer"` see follow-up messages in time.
- `yu/patch/configurable-block-streaming`: make Weixin block replies configurable via `channels.openclaw-weixin.blockStreaming` and account override.
- `yu/patch/sender-id`: populate `SenderId` for multi-user routing.
- `yu/patch/media-dedup`: dedupe repeated media sends within a short window.
- `yu/patch/gateway-methods`: declare Weixin web login gateway methods.
- `yu/patch/voice-ref-msg`: preserve quoted context for voice messages with `ref_msg`.
- `yu/patch/log-dedupe`: reduce repeated compat/runtime initialization logs.

## Check

```bash
cd /home/yu/projects/openclaw-weixin-maintained
npm install --ignore-scripts
npm run typecheck
```

## Update when official npm changes

1. Import new npm package to a new vendor branch.

```bash
VERSION=2.3.2
npm pack @tencent-weixin/openclaw-weixin@$VERSION --pack-destination /tmp
rm -rf /tmp/openclaw-weixin-npm-$VERSION
mkdir -p /tmp/openclaw-weixin-npm-$VERSION
tar -xzf /tmp/tencent-weixin-openclaw-weixin-$VERSION.tgz -C /tmp/openclaw-weixin-npm-$VERSION --strip-components=1

git switch --orphan vendor/npm-$VERSION
git rm -rf .
cp -a /tmp/openclaw-weixin-npm-$VERSION/. .
git add .
git commit -m "vendor: import npm $VERSION package"
```

2. Create a dev base.

```bash
git switch -c yu/base/npm-$VERSION-dev
cp /home/yu/projects/openclaw-weixin/tsconfig.json ./tsconfig.json
git add tsconfig.json
git commit -m "chore: add TypeScript project config for maintenance"
```

3. Recreate or rebase patch branches on the new base. Drop any patch already fixed upstream.

4. Rebuild `yu/current` from the new base and merge kept patches with `--no-ff`.

```bash
git switch -C yu/current yu/base/npm-$VERSION-dev
for b in \
  yu/patch/nonblocking-monitor \
  yu/patch/configurable-block-streaming \
  yu/patch/sender-id \
  yu/patch/media-dedup \
  yu/patch/gateway-methods \
  yu/patch/voice-ref-msg \
  yu/patch/log-dedupe; do
  git merge --no-ff --no-edit "$b"
done
```

5. Run checks, then copy/install into local OpenClaw extension and restart gateway.

## Local install idea

Keep current live plugin untouched until `yu/current` passes checks. Then copy the branch contents into `~/.openclaw/extensions/openclaw-weixin`, preserving account state under `~/.openclaw`.
