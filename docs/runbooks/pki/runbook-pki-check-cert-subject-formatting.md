# Runbook: Check a Certificate's Subject Fields for Literal Formatting Issues

**Category:** Internal PKI
**When to use:** A certificate's CN (or other Subject field) looks visually odd in a browser or UI — extra spacing, unexpected characters — and you need to confirm whether that's a real, literal encoding in the certificate itself versus just a display/rendering artifact of whatever's showing it to you.

## Why this matters

Browser certificate viewers and some tools apply their own formatting/whitespace normalization when displaying Subject fields, which can hide or fabricate the appearance of issues that aren't actually in the certificate — or the reverse, hide a real one. Don't trust the rendered display; check the raw encoding.

## Steps

**1. Pull the Subject line with strict encoding, no cosmetic normalization:**
```bash
openssl x509 -in <cert-file> -noout -subject -nameopt RFC2253
```
`RFC2253` format doesn't apply the kind of display spacing a browser or default OpenSSL output might, so whatever comes back is what's actually encoded.

**2. Interpret the output.** A backslash before a character (e.g. `CN=\ Homelab Intermediate CA`) is RFC2253's way of escaping a literal character that would otherwise be ambiguous — in this example, a literal leading space that really is part of the encoded CN, not a rendering artifact.

**3. If it's confirmed real and needs fixing**, the fix is reissuing the certificate with the corrected value in the CSR/signing command — there's no way to edit a Subject field on an already-issued certificate in place. Locate whatever config drives the default value for that field (e.g. `commonName_default` in an OpenSSL config file) if the issue is likely to recur on future certs from the same CA, not just this one.

## Notes

- This applies to any Subject field (CN, O, OU, etc.), not just CN specifically — swap in `-nameopt RFC2253` output and look for the same escaping pattern.
- Worth checking *before* committing to reissuing anything downstream that depends on the value (e.g. before using a cert's CN as a bootstrap name for a new system) — confirms whether you're about to propagate an existing typo or not.
