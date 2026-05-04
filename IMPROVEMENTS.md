# Codebase Review: Issues & Improvements

Findings from a comprehensive review of the retro.tools frontend. Covers Svelte components, JS modules, security, i18n, tests, and CI/CD. Each entry includes a file:line reference, description, and recommended fix.

---

## Security

### S1 — Weak encryption: no key derivation, no authentication (`src/encryption.js:1–20`)

**Severity: Critical**

The encryption implementation has two fundamental weaknesses:

1. **No key derivation**: The raw password string is passed directly to `AES.encrypt` as the key. crypto-js uses a non-standard, weak KDF (MD5-based EVP_BytesToKey) rather than PBKDF2/scrypt. This makes brute-force attacks significantly easier.
2. **No authentication**: crypto-js uses AES-CBC, which has no integrity/authenticity guarantee. An attacker who can modify stored ciphertext can do so undetected.

**Fix:** Replace with the Web Crypto API (built into all modern browsers, no dependency needed), using PBKDF2 for key derivation and AES-GCM for authenticated encryption:

```js
export async function encrypt(clearText, password) {
  if (!password) return clearText;
  const enc = new TextEncoder();
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const keyMaterial = await crypto.subtle.importKey("raw", enc.encode(password), "PBKDF2", false, ["deriveKey"]);
  const key = await crypto.subtle.deriveKey(
    { name: "PBKDF2", salt, hash: "SHA-256", iterations: 200_000 },
    keyMaterial,
    { name: "AES-GCM", length: 256 },
    false,
    ["encrypt"],
  );
  const ciphertext = await crypto.subtle.encrypt({ name: "AES-GCM", iv }, key, enc.encode(clearText));
  const out = new Uint8Array(salt.byteLength + iv.byteLength + ciphertext.byteLength);
  out.set(salt, 0);
  out.set(iv, salt.byteLength);
  out.set(new Uint8Array(ciphertext), salt.byteLength + iv.byteLength);
  return btoa(String.fromCharCode(...out));
}
```

Note: migrating to AES-GCM is a breaking change for existing encrypted boards — a migration path (detecting old crypto-js format vs new) is required.

---

### S2 — Silent decryption failure returns `"?"` (`src/encryption.js:15–19`)

**Severity: Medium**

```js
} catch {
  return "?";
}
```

Callers (e.g. the CSV export in `Menu.svelte`) cannot tell whether `"?"` is the actual decrypted content or a decryption failure. Wrong passwords, corrupted data, and tampering all look identical.

**Fix:** Throw a typed error instead, and let callers handle it:

```js
} catch {
  throw new Error("Decryption failed");
}
```

---