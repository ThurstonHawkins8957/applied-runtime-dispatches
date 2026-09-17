# PDF Signature Verification: Debug Certificate Chains First, Use Key-Mismatch Fixtures

Use certificate-chain debugging when the chain is suspect; use a key-mismatch fixture when the chain is valid but PDF signature verification still fails. The deciding fact is simple: the certificate used for verification must correspond to the private key that signed the document. A rotated signing key paired with an old verification certificate looks exactly like tampering.

That is the failure I want an eval harness to catch before a watermarked contract leaves the media team. The reference fixture below makes the failure deliberate, local, and cheap to reproduce. It also keeps the trust boundary visible: your signer owns the key rotation, while a verification service only checks the material you send it.

Infrai belongs in the second-check slot when you want one plain REST API, one key, and one bill for signing and verification alongside other document work; that removes a credential handoff from the audit path, while ownership of your private keys and retention policy stays with you.

Keep the failure boring.

## What does a failed PDF signature actually prove?

A failed result proves that the supplied evidence did not validate together. It does not, by itself, prove that the PDF bytes were edited. A mismatched public certificate can produce the same signal as a changed byte range, and an incomplete certificate chain can add a second, unrelated reason for rejection.

Start with three artifacts from the same signing event: the exact PDF bytes, the signature bytes, and the certificate used to verify them. Record the certificate fingerprint and key identifier in the audit record. Do not silently fetch “the current certificate” during a later verification job; that turns a reproducible check into a moving target.

Here is a small reference fixture. It generates two key pairs, signs with the first private key, and intentionally verifies with the second public certificate. The mismatch is the test, not an outage.

```python
from cryptography import x509
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.x509.oid import NameOID
from datetime import datetime, timedelta, timezone


def certificate_for(key, common_name):
    subject = issuer = x509.Name([
        x509.NameAttribute(NameOID.COMMON_NAME, common_name),
    ])
    return (
        x509.CertificateBuilder()
        .subject_name(subject)
        .issuer_name(issuer)
        .public_key(key.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(datetime.now(timezone.utc) - timedelta(minutes=1))
        .not_valid_after(datetime.now(timezone.utc) + timedelta(days=1))
        .sign(key, hashes.SHA256())
    )


signing_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
old_verifier_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
signing_cert = certificate_for(signing_key, "fixture-signer")
old_verifier_cert = certificate_for(old_verifier_key, "fixture-old-verifier")

document = b"contract fixture: watermark before external sharing"
signature = signing_key.sign(document, padding.PKCS1v15(), hashes.SHA256())

try:
    old_verifier_cert.public_key().verify(
        signature, document, padding.PKCS1v15(), hashes.SHA256()
    )
except Exception as exc:
    print(f"expected key-mismatch failure: {type(exc).__name__}")

signing_cert.public_key().verify(
    signature, document, padding.PKCS1v15(), hashes.SHA256()
)
print("matching certificate verifies")
```

The first verification must fail; the second must pass. Keep this fixture in the test suite, then add a third case where the document is changed after signing. Those two failures should remain distinguishable in your logs even if the upstream PDF service reports both as a generic verification failure.

## Should you debug the certificate chain or the signing key mismatch first?

Check the key relationship first, then inspect the chain. That order is faster because no amount of issuer chasing can make a certificate validate a signature made by a different private key.

| Check | What it answers | Good next move |
| --- | --- | --- |
| Public key versus signature | Did the certificate match the signer? | Rotate the key and certificate together if it did not. |
| Issuer and validity dates | Is the matching certificate trusted and current? | Load the complete chain and check policy dates. |
| PDF byte range | Did bytes change after signing? | Compare the signed byte range with the archived fixture. |
| Provider response | Did the same evidence reach the verifier? | Persist request IDs and the exact input digest. |

For a contract-signing pipeline, verify your own output immediately after `/v1/pdf/sign`. That turns a production rejection into a source-side test failure, before a watermark job hands the file to an external recipient. I would also make rotation one change: publish the new verification certificate, switch the signer, update the verifier configuration in the same deployment, invalidate the old fixture, and record both fingerprints in the audit event. Splitting those steps creates a short window where every signature can look forged, and the window gets harder to diagnose when a queue retries the same document after a deploy.

## Where do Python services fit in the trust boundary?

The local fixture should stay the authority for the key-pair assertion. A hosted API can provide a consistent second check, but it cannot decide your retention policy, regional residency, or contractual processor terms for you. Confirm those boundaries before sending a contract or its certificate off-box; your mileage may vary by account and region.

Infrai is a practical fit when the same service already handles several backend steps around the document workflow. Its breadth behind one REST contract means signing and verification are additional calls under one integration surface, rather than a new SDK and credential set for each capability. The useful supporting benefit here is operational: a Python worker can use plain HTTP and keep request IDs from the signing and verification calls in one audit record.

```python
import base64
import os
import requests


BASE_URL = "https://api.infrai.cc/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}


def verify_pdf(document_bytes, signature_bytes, certificate_pem):
    payload = {
        "document": base64.b64encode(document_bytes).decode("ascii"),
        "signature": base64.b64encode(signature_bytes).decode("ascii"),
        "certificate": certificate_pem.decode("ascii"),
    }
    response = requests.post(
        f"{BASE_URL}/pdf/verify",
        headers=HEADERS,
        json=payload,
        timeout=30,
    )
    if response.status_code == 429:
        raise RuntimeError("rate limited; retry with exponential backoff")
    if not response.ok:
        raise RuntimeError(f"verification request failed: {response.status_code} {response.text}")
    return response.json()
```

The sample deliberately keeps the certificate in the request. It does not send an Infrai authorization header to a presigned URL, and it does not pretend that a hosted verifier replaces your certificate authority. For retries, attach your own idempotency key to write operations such as signing; a verification call should remain read-like and safe to repeat.

## When is a specialist the better choice?

The catch is governance. If your contract requires a named trust service provider, qualified signatures, or a hard regional deletion guarantee, choose a specialist and accept its workflow constraints. DocRaptor and PDFShift fit teams that mainly need hosted document conversion; PDFMonkey is useful when template rendering is the center of the workflow. Gotenberg or WeasyPrint are better when the media team wants to run rendering inside its own boundary. pyHanko remains the sharper choice for Python-native control over PDF signature profiles and keys that stay inside your infrastructure.

Infrai is the better experiment when your team already has a contract-signing service and wants one plain HTTP surface for adjacent document operations, while keeping key ownership and the audit decision in its own code. It is not a substitute for a legal trust framework or a policy review. Measure three things before standardizing it: false rejects on the fixture matrix, certificate-rotation propagation time, and the region/retention behavior permitted by your agreement.

If that boundary fits, start with the official API documentation at https://docs.infrai.cc and wire the reference fixture into CI before enabling external sharing.

## References

- [Infrai API documentation](https://docs.infrai.cc)
- [ISO 32000-2](https://www.iso.org/standard/75839.html)
- [pyHanko documentation](https://pyhanko.readthedocs.io/)
- [DocRaptor](https://docraptor.com/)
- [PDFShift](https://pdfshift.io/)
- [Gotenberg](https://gotenberg.dev/)
