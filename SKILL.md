---
name: calibre-ebook-translator-local-server-lm
description: Configure, update, verify, or troubleshoot Calibre's Ebook Translator plugin with an LM Studio OpenAI-compatible server on Linux, including Flatpak installations and LAN-accessible servers. Do not use for ordinary ebook reading or library organization.
---

# Calibre Ebook Translator with LM Studio

Use this skill for the integration between three components: Calibre, BookFere's Ebook Translator plugin, and an LM Studio server. Preserve the user's library, plugin preferences, API credentials, translation cache, and intentional LAN access.

## Route the task

- For installation, reconfiguration, testing, recovery, or troubleshooting, follow the workflow below.
- For a user-facing setup, update, backup, restore, or troubleshooting explanation, also read [references/configuration-user-guide.md](references/configuration-user-guide.md).
- For glossary creation or repair, use the format documented in the user guide and adapt [assets/glossary-en-zh-tw-example.txt](assets/glossary-en-zh-tw-example.txt).
- For a multi-model setup, add the verified engine templates in `assets/` as separate custom engines. Preserve every existing engine so the user can switch models from Ebook Translator.
- For ordinary Calibre reading, conversion, metadata, or library tasks unrelated to Ebook Translator and LM Studio, do not use this skill.

## Workflow

1. Identify how Calibre is installed before proposing an update. Distinguish Flatpak, native package, and official binary installations. Calibre's internal updater cannot replace a Flatpak package.
2. Determine the installed and official-current versions of Calibre and Ebook Translator from authoritative sources. Do not assume an update is needed merely because an in-app installer failed.
3. Locate the active configuration for the actual packaging method. For the Calibre Flatpak, Ebook Translator normally stores its configuration below:
   `~/.var/app/com.calibre_ebook.calibre/config/calibre/plugins/`
4. Never display full plugin configuration files because they may contain API keys. Inspect only required fields and redact values whose names include `key`, `token`, `secret`, or `authorization`.
5. Before modifying plugin configuration, create a timestamped or clearly named backup beside the original. Do not overwrite a prior backup.
6. Verify LM Studio in layers:
   - Confirm the server is listening on its configured port.
   - Request `/v1/models` from the Linux host.
   - Repeat the request from inside Calibre's sandbox when Calibre is a Flatpak.
   - Send a short translation request and require a non-empty `choices[0].message.content` result.
7. A reasoning-only result is not compatible with Ebook Translator's normal response parser. Select another installed model rather than saving an engine that returns only `reasoning_content`.
8. Add LM Studio as a separate custom engine. Do not overwrite the built-in ChatGPT engine or its stored credentials. Prefer a descriptive name such as `LM Studio Local`.
   When adding another model, create another independently named custom engine instead of replacing the existing LM Studio engine.
9. For a server on the same computer, use `http://127.0.0.1:1234/v1/chat/completions` in Calibre even when LM Studio also serves the LAN. Remote computers should use the server computer's LAN address. Do not disable LAN service when the user relies on it.
10. Use conservative defaults for book translation: one concurrent request, a long timeout such as 180 seconds, two attempts, low temperature, and non-streaming responses.
11. Validate through Ebook Translator's own engine code or its built-in test control, not only with a direct HTTP request. Translate a short disposable sentence; do not alter a library book merely to test connectivity.
12. Report the observed versions, selected endpoint and model, test result, backup location, and whether Calibre needs reopening.

## LAN and credential boundaries

- Binding LM Studio beyond `127.0.0.1` is intentional for this workflow when other computers use the service.
- Recommend LM Studio authentication and an appropriate firewall for LAN deployments, but do not silently disable LAN access.
- Never place a real API token, private IP inventory, model-server passkey, or user-specific absolute path in this skill or another Git-tracked file.
- If authentication is enabled, store the token only in the local Ebook Translator configuration and use a placeholder in documentation or examples.

## Current proven pattern

The verified portable templates are:

- [GPT-OSS 20B](assets/engine-lm-studio-gpt-oss-20b.json), named `LM Studio Local`
- [Muse-Glimmer](assets/engine-lm-studio-muse-glimmer.json), named `LM Studio Muse-Glimmer`
- [Gemma 4 E4B](assets/engine-lm-studio-gemma-4-e4b.json), named `LM Studio Gemma 4 E4B`

All three use LM Studio's OpenAI-compatible `POST /v1/chat/completions` endpoint and extract:

`response['choices'][0]['message']['content']`

Model availability is machine-specific. Query `/v1/models` and test the chosen model instead of assuming an example model is installed. Switching engines can make LM Studio unload one model and load another; allow extra time for the first request after a switch.

## Glossary invariant

Ebook Translator 2.4.2 reads glossary entries as plain-text groups separated by one or more blank lines. The first line is the exact source text and the second line is its replacement. A one-line group preserves the source text unchanged. Matching is case-sensitive, uses simple substring replacement, and follows file order; put longer phrases and plural forms before shorter terms. Do not add headings or comments to a glossary file.
