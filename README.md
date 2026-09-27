# Calibre Ebook Translator with an LM Studio LAN Server

This guide explains how to maintain the working connection between Calibre's **Ebook Translator** plugin and an LM Studio server. It is written so the setup can be recreated after an update or repaired without guessing.

## What connects to what

```text
Calibre
  -> Ebook Translator plugin
      -> LM Studio OpenAI-compatible API
          -> local language model
```

Calibre and LM Studio may run on the same Linux computer. LM Studio can simultaneously serve other computers on the local network.

## Known working baseline

The setup verified on 2026-09-27 used:

- Linux Mint 21.3 with Calibre installed as a system Flatpak.
- Calibre 9.14.0.
- Ebook Translator 2.4.2 by BookFere.
- LM Studio's OpenAI-compatible server on port 1234.
- English to Traditional Chinese translation.
- Three independent, verified custom engines:
  - `LM Studio Local` using `openai/gpt-oss-20b`.
  - `LM Studio Muse-Glimmer` using `meta/muse-glimmer`.
  - `LM Studio Gemma 4 E4B` using `google/gemma-4-e4b`.

These version numbers are a historical baseline, not a promise that they remain the latest. Always compare with the official sources:

- Calibre releases: https://calibre-ebook.com/whats-new
- Ebook Translator releases: https://github.com/bookfere/Ebook-Translator-Calibre-Plugin/releases
- LM Studio server documentation: https://lmstudio.ai/docs/developer/core/server
- LM Studio OpenAI-compatible endpoints: https://lmstudio.ai/docs/developer/openai-compat

## Normal daily use

1. Start LM Studio.
2. Open **Developer** and start the local API server.
3. Ensure a suitable text model is available. LM Studio may load it automatically if Just-In-Time loading is enabled.
4. Start Calibre and open Ebook Translator.
5. Select the LM Studio engine you want. `LM Studio Local` is the recommended default.
6. Confirm the source and target languages before translating a book.

The model remains local to the LM Studio server. No paid cloud API is required for this engine.

LM Studio may keep only one of these models loaded, depending on available memory. The first request after changing engines can be much slower because Just-In-Time loading may unload the previous model and load the selected one.

## Updating Calibre correctly

First determine how Calibre was installed. For a Flatpak installation, this command shows the installed version:

```bash
flatpak info com.calibre_ebook.calibre
```

Check what Flathub currently offers:

```bash
flatpak remote-info flathub com.calibre_ebook.calibre
```

Update the Flatpak with Linux Mint's Update Manager or:

```bash
flatpak update com.calibre_ebook.calibre
```

Do not use Calibre's website installer over an existing Flatpak installation. Calibre's in-app update notice can announce a release, but it cannot replace the Flatpak package. Flathub may also publish a new Calibre release slightly later than the upstream website.

## Updating Ebook Translator

In Calibre:

1. Open **Preferences**.
2. Open **Plugins**.
3. Search for `Ebook Translator`.
4. Check whether Calibre offers an update.
5. Install the update and restart Calibre when requested.

To check the installed version from the terminal when using the Flatpak:

```bash
flatpak run --command=calibre-customize com.calibre_ebook.calibre -l
```

Look for the `Ebook Translator` row. Back up its settings before manually replacing the plugin ZIP.

## Preparing LM Studio for this computer and the LAN

In LM Studio, open **Developer > Server Settings**:

1. Use port `1234`, unless another application already uses it.
2. Enable **Serve on Local Network** so other computers can connect.
3. Keep the operating-system firewall limited to trusted local networks.
4. Prefer **Require Authentication** when the server is reachable by other devices.
5. If authentication is enabled, create a dedicated token for ebook translation and keep it out of Git.

For Calibre on the same Linux computer, use the loopback endpoint even while LAN service remains enabled:

```text
http://127.0.0.1:1234/v1/chat/completions
```

For another computer, replace `127.0.0.1` with the Linux server's LAN address:

```text
http://SERVER_LAN_ADDRESS:1234/v1/chat/completions
```

Test the model list locally:

```bash
curl http://127.0.0.1:1234/v1/models
```

From another computer, test the LAN address instead. A successful response contains a `data` list with model identifiers.

## Creating the Ebook Translator custom engine

Ready-to-paste templates are included with this skill:

- [LM Studio Local — GPT-OSS 20B](assets/engine-lm-studio-gpt-oss-20b.json)
- [LM Studio Muse-Glimmer](assets/engine-lm-studio-muse-glimmer.json)
- [LM Studio Gemma 4 E4B](assets/engine-lm-studio-gemma-4-e4b.json)

Open Ebook Translator's settings and find its custom translation-engine manager. For each model you want, add a **new** custom engine, paste the corresponding JSON, and save it. Do not replace another engine: the different names make all three choices available in Ebook Translator's engine menu.

Before pasting, query `/v1/models` and confirm that the template's model identifier is available. The GPT-OSS template is reproduced below as the baseline example.


```json
{
  "name": "LM Studio Local",
  "languages": {
    "source": {
      "English": "English",
      "Chinese (Traditional)": "Traditional Chinese"
    },
    "target": {
      "Chinese (Traditional)": "Traditional Chinese",
      "English": "English"
    }
  },
  "request": {
    "url": "http://127.0.0.1:1234/v1/chat/completions",
    "method": "POST",
    "headers": {
      "Content-Type": "application/json"
    },
    "data": {
      "model": "openai/gpt-oss-20b",
      "messages": [
        {
          "role": "system",
          "content": "Translate the following text from <source> to <target>. Preserve meaning, names, paragraph structure, and formatting. Return only the translation, with no explanation."
        },
        {
          "role": "user",
          "content": "<text>"
        }
      ],
      "temperature": 0.1,
      "max_tokens": 2048,
      "stream": false
    }
  },
  "response": "response['choices'][0]['message']['content']"
}
```

Recommended plugin settings:

- Source language: `English`
- Target language: `Chinese (Traditional)`
- Concurrent requests: `1`
- Request timeout: `180` seconds
- Attempts: `2`
- Request interval: `0.5` seconds

One concurrent request is slower but avoids overloading a local model and is easier to recover when a long chapter contains difficult text.

### Choosing among the three engines

| Engine | LM Studio model identifier | Suggested timeout | Use it when |
| --- | --- | ---: | --- |
| `LM Studio Local` | `openai/gpt-oss-20b` | 180 seconds | You want the tested default and predictable text output. |
| `LM Studio Muse-Glimmer` | `meta/muse-glimmer` | 300 seconds | You prefer Muse-Glimmer and can allow extra loading and reasoning time. |
| `LM Studio Gemma 4 E4B` | `google/gemma-4-e4b` | 240 seconds | You want an independent Gemma option with a longer first-load allowance. |

The timeout is an Ebook Translator setting, not part of the engine JSON. Keep concurrent requests at `1`, especially when switching large models. After selecting a different engine, test one sentence and wait for LM Studio to finish loading before deciding that the request has timed out.

### When LM Studio authentication is enabled

Add an Authorization header to the engine's `headers` object:

```json
{
  "Content-Type": "application/json",
  "Authorization": "Bearer REPLACE_WITH_YOUR_LM_STUDIO_TOKEN"
}
```

Use the real token only in Calibre's local configuration. Never save the real token in this guide, a Git commit, a screenshot, or a support message.

## Creating a Translation Glossary

The ready-to-edit example is [glossary-en-zh-tw-example.txt](assets/glossary-en-zh-tw-example.txt).

Ebook Translator expects a UTF-8 plain-text file:

1. Put the exact English source term on the first line.
2. Put the Taiwan Traditional Chinese replacement on the second line.
3. Leave one blank line before the next term pair.
4. Do not add a title row, column names, commas, tabs, or comments.

Example:

```text
artificial intelligence
人工智慧

software
軟體

information
資訊
```

If a term must remain unchanged, repeat it on the second line or use a one-line group:

```text
LM Studio
LM Studio

EPUB
```

Important matching behavior:

- Matching is case-sensitive. Add separate entries when both `Calibre` and `calibre` occur.
- Matching uses direct substring replacement, not whole-word matching.
- Entries are processed from top to bottom. Put longer phrases and plural forms before shorter terms, such as `local area network` before `network`.
- Avoid overly broad entries when the English term can have different meanings in different contexts.
- Save the file as UTF-8 with a `.txt` extension.

To use it in Calibre:

1. Open Ebook Translator settings.
2. Open the **Content** tab.
3. Under **Translation Glossary**, select **Enable**.
4. Click **Choose** and select the glossary `.txt` file.
5. Save the settings and test one short paragraph before translating a book.

## Testing before translating a whole book

Use Ebook Translator's test feature with one short sentence:

```text
Hello, this is a translation test from the Calibre plugin.
```

A successful Traditional Chinese result should be similar to:

```text
你好，這是來自 Calibre 外掛的翻譯測試。
```

The exact wording can differ. What matters is that the result is non-empty, contains only the translation, and does not contain an explanation or hidden reasoning.

Do not begin with an entire book. First test a short selection, then one chapter, and only then translate the complete EPUB.

## Backing up the plugin configuration

For the Calibre Flatpak, the main Ebook Translator configuration is normally:

```text
~/.var/app/com.calibre_ebook.calibre/config/calibre/plugins/ebook_translator.json
```

Close Calibre before making a manual backup. Then copy the file to a clearly named backup:

```bash
cp -p ~/.var/app/com.calibre_ebook.calibre/config/calibre/plugins/ebook_translator.json ~/.var/app/com.calibre_ebook.calibre/config/calibre/plugins/ebook_translator.json.backup-before-change
```

The plugin ZIP and additional settings are in the same `plugins` directory. Do not commit that live directory to Git because it may contain credentials, translation preferences, and caches.

### Restoring a backup

Restoring overwrites the current plugin settings. Close Calibre, make a new backup of the current file, and only then copy the chosen backup over `ebook_translator.json`. Reopen Calibre and run the short translation test again.

## Troubleshooting

### Calibre says an update is available but installation fails

If Calibre is a Flatpak, update it through Flatpak or Linux Mint's Update Manager. Do not run the upstream installer over the Flatpak.

### Connection refused

- Confirm LM Studio is open and the API server is started.
- Confirm the port in LM Studio and the custom engine match.
- From the same computer, test `http://127.0.0.1:1234/v1/models`.
- From another computer, use the server's LAN address and check the firewall.

### Unauthorized or HTTP 401

LM Studio authentication is enabled. Add a valid `Authorization: Bearer ...` header using a dedicated token.

### Model not found

Request `/v1/models` and copy the exact model identifier into the engine JSON. Display names in the LM Studio interface are not always identical to API identifiers.

### The test is blank or parsing fails

Inspect a direct `/v1/chat/completions` response. Ebook Translator expects the final text at:

```text
choices[0].message.content
```

Some reasoning or vision models return text only in `reasoning_content`. Choose a text model that returns a normal `content` field rather than changing the plugin to treat hidden reasoning as the translation.

### Translation is very slow or times out

- Wait for LM Studio to finish loading the model before testing.
- After changing engines, watch LM Studio for an unload/load cycle; the first request is expected to take longer.
- Keep concurrency at `1`.
- Use the suggested timeout in the engine table, or increase it slightly when model loading is slow.
- Use a smaller text model when speed matters more than maximum quality.
- Test one short selection to separate model-loading delay from translation speed.

### The custom engine disappeared

Verify that Calibre is using the expected installation. Flatpak and native Calibre installations have different configuration directories. Restore the correct backup or recreate the engine with the JSON template above.

## Information to record after future changes

When an update succeeds, record these details without including tokens:

- Calibre installation type and version.
- Ebook Translator version.
- LM Studio version and server port.
- Model API identifier.
- Source and target languages.
- Whether the test sentence succeeded.
- Backup filename created before the change.
