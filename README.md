# max-copy-paste

Send secret messages that look like ordinary (if slightly broken) Russian text. One HTML
file, no installation, no internet — open it in a browser on a phone or a laptop and use it
offline. The interface is in Russian.

<p align="left">
  <img src="imgs/screenshot-key.png" width="255" alt="Ключ: the symmetric key">
  <img src="imgs/screenshot-message.png" width="255" alt="Сообщение: the plaintext">
  <img src="imgs/screenshot-cipher.png" width="255" alt="Шифр: the look-alike cypher text">
</p>

## Quick start

1. Open `max_copy_paste.html` in a browser — double-click it, or send it to your phone and
   open it there.
2. Set up a key once and give the same key to the person you will be talking to
   (see [Setting the key](#setting-the-key)).
3. Write a message, press `Зашифровать`, copy the result and send it like any other text
   (see [Encrypting a message](#encrypting-a-message-step-by-step)).
4. When you receive such a text, paste it and press `Расшифровать`
   (see [Decrypting a message](#decrypting-a-message-step-by-step)).

The app has three tabs and one large text field:

| Tab | What it holds |
|---|---|
| **Ключ** | your secret key |
| **Сообщение** | the message, in plain text |
| **Шифр** | the disguised text — this is what you send and receive |

## Setting the key

The key is a shared secret. Both sides must use the same key: whoever has the key and this
file can read everything encrypted with it, and without the key nobody can.

### First launch — your key is created for you

1. Open the app. On the very first launch it lands on the **Ключ** tab and shows you a
   freshly generated key: 4 lines of 4 hex groups, in red.
2. Send this key to your contact — read it over the phone, photograph it, write it down.
   Use a channel you trust.
3. Done. The key is saved in your browser and will be loaded automatically from now on.

### Entering a key you received

1. Go to the **Ключ** tab, clear the field and type or paste the key. Line breaks, spaces
   and capital letters do not matter — the app cleans them up.
2. Press `Перезаписать ключ`. The status line under the field says «Ключ сохранён».
3. If it says «Ключ должен содержать 64 hex-символа (0-9, a-f)» — the key is incomplete,
   check it with your contact.

### Getting a new key

1. On the **Ключ** tab, clear the field and press `Перезаписать ключ`. A new random key
   appears and is saved immediately.
2. Send the new key to your contact. **Messages encrypted with the old key can no longer
   be decrypted by anyone** — including you.

## Encrypting a message (step by step)

1. Open the **Сообщение** tab and type or paste your message.
2. Press `Зашифровать` — a small spinner runs for a moment.
3. The app jumps to the **Шифр** tab. The field now holds a paragraph of odd Russian
   words with punctuation — this is your disguised message.
4. Press `Копировать` — the text goes to the clipboard («Скопировано» flashes on the
   button).
5. Paste it into any messenger or e-mail and send. It looks like ordinary text and
   survives copy-pasting.

## Decrypting a message (step by step)

1. Copy the text you received.
2. In the app, open the **Шифр** tab and paste it into the field. Do not worry about
   spacing, punctuation or capital letters — if your messenger re-wrapped the text, it
   still works.
3. Press `Расшифровать` — the spinner runs for a moment.
4. The app jumps to the **Сообщение** tab and shows the message. Press `Копировать` to
   copy it to the clipboard.

`Зашифровать` is greyed out at this point on purpose, so the decrypted text is not
accidentally encrypted again. Edit the text, or press `Начать сначала`, when you want to
write something new.

If anything goes wrong, the status line under the field tells you what happened:

| Message | Meaning |
|---|---|
| «Похоже, это не шифротекст или он повреждён» | the text is not one of ours, or some words were lost on the way |
| «Не удалось расшифровать (неверный ключ?)» | the text is intact, but your key differs from the sender's |
| «Скопируйте текст вручную» | the browser refused clipboard access — select the text and copy it yourself |

## Good to know

- `Начать сначала` clears both working fields and puts the buttons back to their starting
  state. It never touches the key.
- Use a different key for a different person if you want to be able to revoke one
  conversation without losing the other.
- Losing the key means losing the messages; sharing the key means sharing the messages.
- The app needs `file://` or `https://` — browsers only provide encryption there.

---

## Technical details

### Encryption pipeline

1. The message is encoded as UTF-8 and encrypted with **AES-256-GCM** (random 12-byte IV,
   16-byte authentication tag).
2. Every byte of `IV ‖ ciphertext ‖ tag` becomes one word from **CYPHERBOOK**, a table of
   256 Russian words — word index = byte value.
3. Words from **NOISE**, a second table of 256 service words and numerals, are sprinkled
   between them at random — about 20% of the words in the result are noise.
4. Punctuation (`, ; - . ! ?`), line breaks and capitals at sentence starts are added on
   top; they carry no information.

Decryption goes in reverse: punctuation is stripped, everything is lower-cased, noise
words are dropped, the remaining words map back to bytes, and AES-GCM verifies and
decrypts. Re-typing the text — changing case, spacing or punctuation — does not hurt, but
an unknown word is rejected instead of being silently mangled.

### Word tables

Both tables are built for size: 80% of the words are short (2-5 letters, plus a handful of
one-letter service words), the remaining 20% are ordinary longer words so the text does not
look like a telegram. AES output bytes are uniformly random, so every table entry is used
equally often and only the average word length matters — there is no "short word for a
frequent byte" trick.

Invariants, checked at startup by the app itself: 256 + 256 words, every word unique
across both tables, plain Cyrillic, one word per byte value, 205 words of up to 5 letters
and 51 longer ones per table. If they break, the status line says
«Таблицы слов повреждены».

### Size

A 100-character message becomes roughly 1.4 KB of text: the camouflage costs about 14× in
size (measured; it was ~30× before the tables were trimmed to short words and the noise
ratio was cut to 20%).

### The key

- 256 bits, shown as 4 lines of 4 hex groups:
  ```
  a3f1 c02b 9d4e 7712
  00aa 5b3c d9e8 4f20
  31c0 7e4a bb19 5d63
  84fe 20a7 6cb4 0f91
  ```
- Stored only in the browser's `localStorage` (`mcp.key.v1`) and never sent anywhere. If
  the browser refuses storage, the key survives only until the page is closed — the app
  says so in the status line.
- `Перезаписать ключ` with an empty field generates a fresh random key.

### Security model and limitations

- The page carries `Content-Security-Policy: default-src 'none'; connect-src 'none'; …` —
  no fetch, no XHR, no CDN, no fonts, no images. Everything is inline in the single file,
  and the only network-capable code is none at all.
- The disguise is camouflage, **not** steganography and **not** plausible deniability. The
  security comes from AES-256-GCM; the words only make the traffic look boring. The word
  tables are part of the app, so anyone who has the file can turn the text back into bytes —
  they still need the key to read anything.
- AES-GCM provides integrity as well as secrecy: a wrong key and a tampered text both fail
  loudly instead of producing garbage.

### Development

`max_copy_paste.html` is the whole application: `<style>`, markup and one `<script>`. It is
kept human-readable and is meant to be minified by hand later. There is no build step and
no dependency.

- The two word tables sit at the top of the script as plain word lists.
- Keep every table word free of `ё`, hyphens and digits — the decoder strips everything
  that is not a Russian letter before matching.
- Open the file, read it, change it.

### License

Apache-2.0 — see [LICENSE](LICENSE).
