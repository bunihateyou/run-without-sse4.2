# run-without-sse4.2

Instructions for running programs on x86_64 Linux CPUs **without SSE4.2 / SSE4.1 / SSSE3 / AVX** — e.g. AMD Athlon II X4, Phenom, Opteron K10, and similar pre-2008 microarchitectures.

> **Prerequisite:** You need a working SSE4.2-free Bun binary first. Build one at [`bunihateyou/bun-no-sse4.2`](https://github.com/bunihateyou/bun-no-sse4.2) — a custom Bun v1.4.0 build with WebKit/JavaScriptCore compiled at `-march=barcelona`.
>
> The Bun binary alone isn't always enough — programs built *on top of* Bun can ship their own prebuilt SSE4.2 binaries that need separate attention. Below are instructions for two such programs.

---

## <sub>`omp`</sub> (Oh-My-Pi)

Repo: [`can1357/oh-my-pi`](https://github.com/can1357/oh-my-pi)

`omp` depends on `@oh-my-pi/pi-natives`, a Rust/N-API native addon shipped as a prebuilt `.node` binary. Even the `"baseline"` variant is compiled at `target-cpu=x86-64-v2` (SSE4.1/SSSE3) and SIGILLs on K10 when loaded. You must rebuild it from source at `target-cpu=barcelona`:

```sh
git clone --depth 1 https://github.com/can1357/oh-my-pi.git /tmp/oh-my-pi
cd /tmp/oh-my-pi

# RUSTFLAGS overrides omp's build script (which defaults to x86-64-v2/v3).
# The script only sets RUSTFLAGS when unset, so pre-setting wins.
RUSTFLAGS="-C target-cpu=barcelona" cargo build --release -p pi-natives

# Replace every prebuilt baseline .node in the global install:
find ~/.bun/install/global -name 'pi_natives.linux-x64-baseline.node' \
  -exec cp target/release/libpi_natives.so {} \;
```

> **After any `omp update`:** the SSE4.1 `.node` comes back from npm. Re-run the `cp` line above. The `modern` variant can be ignored — it's not loaded on non-AVX2 CPUs.

---

## <sub>`claude-code`</sub> (Claude Code)

Repo: [`anthropics/claude-code`](https://github.com/anthropics/claude-code)

Claude Code ships as a `bun build --compile` standalone binary — a 238 MB ELF with an embedded Bun runtime + JavaScriptCore baked in at modern ISA. It SIGILLs on K10 and can't be patched in-place. However, the bundled JS source can be **extracted** from the binary and run with our barcelona Bun instead:

```sh
# 1. Download the native binary (from npm — it's the same binary the curl installer gives)
mkdir -p ~/claude-code && cd ~/claude-code
curl -sL -o cc.tgz "https://registry.npmjs.org/@anthropic-ai/claude-code-linux-x64/-/claude-code-linux-x64-$(curl -s https://registry.npmjs.org/@anthropic-ai/claude-code/latest | grep -o '"version":"[^"]*"' | cut -d'"' -f4).tgz"
mkdir -p cc-extract && tar xzf cc.tgz -C cc-extract
# the native binary is at cc-extract/package/claude

# 2. Extract the bundled JS source using bun-extractor
curl -sL -o extract.py https://raw.githubusercontent.com/lauralex/bun-extractor/main/bun_extractor.py
python3 extract.py cc-extract/package/claude
# → outputs /tmp/claude_extracted/root/src/entrypoints/cli.js (the full Claude Code source, ~18 MB)

# 3. Install the extracted source + a wrapper that runs it with our bun
cp /tmp/claude_extracted/root/src/entrypoints/cli.js ~/claude-code/cli.js

cat > ~/claude-code/claude <<'EOF'
#!/bin/sh
export DISABLE_INSTALLATION_CHECKS=1
export DISABLE_UPDATES=1
exec "$HOME/bun-barcelona/bun" "$HOME/claude-code/cli.js" "$@"
EOF
chmod +x ~/claude-code/claude
ln -sf ~/claude-code/claude ~/.local/bin/claude

# 4. Two patches needed in cli.js (the endpoint/model compatibility fixes):
#    a) Skip model validation (endpoint 404s on model IDs with slashes)
perl -i -pe 's/retrieve\(e,t=\{\},n\)\{let\{betas:r\}=t\?\?\{\};return this\._client\.get\(Ea`\/v1\/models\/\$\{encodeURIComponent\(e\)\}\?beta=true`,\{\.\.\.n,headers:hs\(\[\{\.\.\.r\?\.toString\(\)!=null\?\{"anthropic-beta":r\?\.toString\(\)\}:void 0\},n\?\.headers\]\)\}\)\}/retrieve(e,t={},n){return Promise.resolve({id:e,object:"model",display_name:e,created:0,owned_by:"gateway"})}/g' ~/claude-code/cli.js
perl -i -pe 's/retrieve\(e,t=\{\},n\)\{let\{betas:r\}=t\?\?\{\};return this\._client\.get\(Ea`\/v1\/models\/\$\{encodeURIComponent\(e\)\}`,\{\.\.\.n,headers:hs\(\[\{\.\.\.r\?\.toString\(\)!=null\?\{"anthropic-beta":r\?\.toString\(\)\}:void 0\},n\?\.headers\]\)\}\)\}/retrieve(e,t={},n){return Promise.resolve({id:e,object:"model",display_name:e,created:0,owned_by:"gateway"})}/g' ~/claude-code/cli.js
#    b) Convert mid-conversation system messages to user-role (some endpoints reject role:"system" in messages)
perl -i -pe 's/if\(f\.type==="api_system"\)return\{role:"system",content:f\.message\.content\}/if(f.type==="api_system")return{role:"user",content:[{type:"text",text:"<system-reminder>"+(typeof f.message.content==="string"?f.message.content:JSON.stringify(f.message.content))+"<\/system-reminder>"}]}/g' ~/claude-code/cli.js

# 5. Verify
claude --version   # → 2.1.x (Claude Code)
```

> **Note:** Claude Code's bundled source uses a `bun build --compile` format with a `\n---- Bun! ----\n` trailer. The extractor reads this format directly. The two `perl` patches fix: (a) model-ID-with-slashes causing a 404 on `/v1/models/{id}` validation, and (b) `role:"system"` in the messages array being rejected by some Anthropic-compatible endpoints. Neither patch affects functionality — they only bypass client-side validation and rewrap system messages.
>
> **After updating Claude Code:** re-run steps 1-4 to extract + patch the new version. The `DISABLE_UPDATES=1` env var in the wrapper prevents the auto-updater from replacing your setup silently.
