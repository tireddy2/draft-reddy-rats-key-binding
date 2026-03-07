---
title: "A Key Binding Claim for Entity Attestation Tokens (EAT)"
abbrev: "EAT-KB"
category: std

docname: draft-reddy-rats-key-binding-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "RATS"
keyword:
 - PQC
 - COSE
 - JOSE
 - Hybrid
 - HPKE
 - Post-Quantum


venue:
  group: "RATS"
  type: "Working Group"
  mail: "rats@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/rats/"


stand_alone: yes
pi: [toc, sortrefs, symrefs, strict, comments, docmapping]

author:
 -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    city: Bangalore
    region: Karnataka
    country: India
    email: "k.tirumaleswar_reddy@nokia.com"

normative:

informative:



--- abstract

This document defines a new Entity Attestation Token (EAT) claim that cryptographically binds a private key used to sign a certificate signing request (CSR), or the private key corresponding to an end-entity certificate used for TLS authentication, to an attested execution environment.

The claim includes a proof of possession generated using the private key and carried within the EAT. Because the EAT is signed by a hardware-backed Attestation Key (AK), successful verification of both the signature on the EAT and the proof of possession establishes a cryptographic binding between the private key and the attested platform state. This mechanism addresses key substitution attacks that arise when attestation evidence and the certificate private keys are validated independently.


--- middle

# Introduction

Remote attestation enables an entity to produce attestation evidence that a verifier can use to assess whether the entity’s platform state satisfies a required policy. In certificate enrollment and authentication protocols (e.g., TLS), a common security policy requirement is that a private key used by an endpoint must be generated, stored, and protected within a trusted execution environment (TEE) or comparable hardware root of trust.

In certificate enrollment workflows, a Certification Authority (CA) may require attestation evidence demonstrating that the private key corresponding to the public key in a certificate signing request (CSR) is protected by a hardware-backed environment. The LAMPS CSR Attestation specification {{!I-D.ietf-lamps-csr-attestation}} defines mechanisms for including attestation evidence alongside a CSR. In this model, the CA verifies the CSR signature using the public key contained in the request and independently verifies the attestation evidence according to the RATS architecture, using applicable endorsements and trust anchors.
However, the attestation evidence does not inherently provide a cryptographic proof that the private key used to sign the CSR is the same key that is generated, stored, or protected within the attested environment. The CSR signature demonstrates possession of a private key, and the attestation demonstrates properties of a platform state, but there is no standardized mechanism that cryptographically binds these two validations together. An endpoint could present valid attestation evidence from a protected environment while submitting a certificate signing request (CSR) that is signed with a private key not generated or stored within that environment. In this case, the Certification Authority has no intrinsic cryptographic assurance that the private key corresponding to the CSR public key benefits from the protections described in the attestation evidence.

A similar problem exists in TLS-based scenarios. The TLS Exported Attestation specification {{!I-D.fossati-tls-exported-attestation}} and the TLS Early Attestation specification {{!I-D.fossati-seat-early-attestation}} define mechanisms for conveying attestation evidence within a TLS connection. While the attestation evidence is bound to the TLS connection in these approaches, it does not intrinsically bind the attested environment to the private key corresponding to the end-entity certificate used for TLS authentication. An endpoint could therefore obtain valid attestation evidence from a protected environment while performing certificate-based TLS authentication using a private key that is not confined to that environment. For example, the TLS private key may reside outside the trusted execution environment and lack the protections claimed by the attestation evidence.

This separation between validation of attestation evidence and validation of certificate private key creates a class of key substitution attacks. In such an attack:

* A valid attestation produced by a genuine hardware-protected environment is presented to the verifier; and
* A private key that is not generated or protected within that environment is used in the CSR or TLS authentication flow.

Because the attestation evidence and the certificate private key are validated independently, the verifier has no intrinsic cryptographic assurance that the operational private key benefits from the protections described in the attestation. This undermines security policy objectives that require the certificate private key to be generated and constrained within the attested environment.

Addressing this problem requires a mechanism that provides both proof of possession of the private key and a cryptographic binding between that key and the attested platform state.

A Relying Party will also require additional Claims describing key protection properties, such as non-exportability or hardware-level protection. For example, {{!I-D.ietf-rats-pkix-key-attestation}} defines an Evidence format for reporting properties of cryptographic modules and managed keys in PKIX environments, but it does not provide cryptographic proof of possession of the Subject Key by the Attester. The PKIX Key Attestation specification {{!I-D.ietf-rats-pkix-key-attestation}} defines attributes including extractable, never-extractable, sensitive, and local that describe protection properties of keys managed by cryptographic modules. These attributes can be conveyed together with the key-binding claim defined in this document to allow a verifier to evaluate security policy requirements related to key protection.

Appendix A.1.4 of {{!RFC9711}} illustrates how a key and key store may be represented in Evidence. However, the example uses private-use claim labels and does not define standardized key-protection Claims or a proof-of-possession mechanism for the attested key. The key-binding claim defined in this document provides a standardized cryptographic binding between the Subject Key and the attested platform state.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Attestation Key (AK): A cryptographic key, typically hardware-backed, used to sign attestation evidence that conveys claims about the state of a platform or trusted execution environment.

Attestation Evidence: Claims about a platform’s state, signed by an Attestation Key, and conveyed in an Entity Attestation Token (EAT).

Entity Attestation Token (EAT): A token format that conveys attestation claims, as defined in {{?RFC9711}}

Subject Key: An asymmetric key pair for which the protection of the private component within an attested execution environment is being asserted. The private component is used to generate the proof of possession (PoP), and the corresponding public component is conveyed in the claim and compared with the key used in a CSR or TLS end-entity certificate.

Proof of Possession (PoP): A digital signature generated using the private component of the Subject Key to demonstrate control of that private key at a given point in time.


Verifier: The entity that validates attestation evidence and evaluates whether the platform state and associated keys satisfy its policy.

Certification Authority (CA): An entity that issues certificates and may require attestation evidence during certificate enrollment.

Trusted Execution Environment (TEE): An isolated execution context that provides confidentiality and integrity protections for code and data, including cryptographic key material.

Key Substitution Attack: An attack in which valid attestation evidence from a protected execution environment is presented to a verifier, while a private key used in a CSR or authentication protocol is not generated or protected within that environment. Because attestation evidence and certificate private key are validated independently, the verifier may lack cryptographic assurance that the certificate private key benefits from the protections claimed in the attestation.

# Key Confirmation and Binding Claim

## Overview

This document defines a new EAT claim that establishes a cryptographic binding between a Subject Key and an attested execution environment.

The claim provides two properties:

1. Proof of Possession : Demonstration that the entity presenting the attestation controls the private component of the Subject Key.
2. Key-to-Platform Binding : Cryptographic association between the private component of the Subject Key and the attested platform state.

The mechanism operates by embedding a proof-of-possession (PoP) generated using the Subject Key within attestation evidence that is signed by the Attestation Key (AK). Because the attestation evidence is authenticated by the AK, and the PoP is contained within that evidence, successful verification establishes that:

- The attested platform state is authentic; and
- The entity producing the attestation evidence demonstrates control of the private component of the Subject Key at the time the evidence was generated.

This construction creates a cryptographic linkage between the Subject Key and the attested platform state, mitigating Key Substitution Attacks.

The key-binding claim can also include attributes describing key protection properties and permitted usage of the Subject Key. When present, attributes such as extractable, never-extractable, sensitive, and local MUST follow the semantics defined in {{!I-D.ietf-rats-pkix-key-attestation}}. These attributes enable relying parties to enforce policies such as requiring keys to be generated within the attested environment and prohibiting extraction of private key material.

## High-Level Construction

At a high level, the mechanism binds a certificate private key to an attested execution environment through a nested signature structure.

First, the private component of the Subject Key generates a signature over a verifier-provided nonce. This signature demonstrates possession of the private key at the time the attestation evidence is created.

Next, the corresponding public key and the resulting proof of possession are included as part of the Entity Attestation Token (EAT). The EAT itself is signed by the Attestation Key, which authenticates the platform state.

During verification, three relationships are checked:

- The signature on the EAT establishes that the attested platform state is authentic.
- The proof of possession establishes control of the private key.
- The public key included in the claim is compared with the public key in the CSR or TLS end-entity certificate to ensure they refer to the same operational key.

When these checks succeed, the verifier gains assurance that the certificate private key used in the protocol is the same key whose protection within the attested execution environment is being asserted.

# Proof-of-Possession

The Proof of Possession (PoP) demonstrates control of the private component of the Subject Key at the time the attestation evidence is generated.

The PoP MUST be computed as a digital signature using the private component of the Subject Key and the signature algorithm identified by the `pop-algorithm` field. The input to the signature is the value TBS defined below, which incorporates a verifier-provided nonce. The verifier-provided nonce used as input to the PoP MUST be the same nonce carried in the EAT `nonce` claim defined in {{!RFC9711}}. For the key-binding claim defined in this document, the EAT `nonce` claim MUST contain a single nonce value.

~~~
TBS = ("EAT-KB-PoP-v1" || subject_public_key || verifier_nonce)
PoP = Sign(subject_private_key, TBS, pop-algorithm)
~~~

Where:

* "EAT-KB-PoP-v1" is the Context String. 
* `subject_public_key` is the public key corresponding to the Subject Key contained in 
  the key-binding claim. For the purpose of TBS computation, the subject_public_key MUST be encoded in the same format as it is represented within the key-binding claim.
* `verifier_nonce` is the nonce supplied by the verifier and carried in the EAT `nonce` 
  claim.
* `SHA-256` is used to produce a fixed-length digest over the concatenated inputs prior 
  to signature generation.

The nonce MUST be supplied by the verifier and MUST be unpredictable and unique within the verifier’s replay window. The PoP signature MUST be included within the attestation evidence and covered by the Attestation Key signature over the EAT.

Because the PoP is embedded within AK-signed attestation evidence, successful verification establishes that:

- The attested platform state is authentic; and
- The entity producing the attestation evidence demonstrates control of the private component of the Subject Key.

The validity period of the key-binding claim is determined by the lifetime of the enclosing Entity Attestation Token. Verifiers MUST enforce the iat, nbf, and exp claims defined in {{!RFC9711}} to ensure that attestation evidence and the associated proof-of-possession are not used outside their intended validity window.

## Freshness Requirements

The verifier MUST provide a nonce with sufficient entropy to prevent replay. The nonce MUST be unpredictable and unique within the verifier’s replay window. The verifier MUST validate that the nonce contained in the PoP input matches the nonce it supplied. Failure to include verifier-provided freshness renders the mechanism vulnerable to replay of previously valid attestation evidence.

The verifier-provided nonce is the primary mechanism for ensuring freshness of the proof of possession. The EAT time-based claims (`iat`, `nbf`, and `exp`) provide an additional validity window for the attestation evidence but do not replace the requirement for a verifier-provided nonce. Verifiers MUST validate both the nonce and
the applicable time-based claims when evaluating the key-binding claim.

## Verification Procedure

Upon receipt of attestation evidence containing this claim, the Verifier
MUST perform the following checks:

1. Validate the signature on the EAT using the applicable trust anchors and endorsements for the Attestation Key. If this validation fails, the attestation evidence MUST be rejected.

2. Extract the `subject-public-key`, `pop-algorithm`, and `pop-signature` from the claim.

3. Validate that the `pop-algorithm` is compatible with the `subject-public-key`. If the algorithm is not compatible with the key type and associated key material, the binding verification MUST fail. If the algorithm is unknown or unsupported, the binding verification MUST fail.

4. Validate the `pop-signature` using the extracted `subject-public-key`, the algorithm identified by `pop-algorithm`, and the verifier-supplied nonce. If signature verification fails, the binding verification MUST fail.

5. Compare the Subject Public Key contained in the claim with the public key:
   - contained in the CSR, in certificate enrollment workflow; or
   - contained in the end-entity certificate used for TLS authentication.

   If the public key parameters do not match, the binding verification MUST fail.

Successful completion of all checks establishes a cryptographic binding between the private component corresponding to the public key used in the CSR or TLS end-entity certificate and the attested execution environment at the time the evidence was generated.

The Verifier conveys the result of the binding verification to the Relying Party as part of the attestation result.

# Claim Definition

This document defines a new EAT claim that conveys a cryptographic binding between a Subject Key and the attested platform state.

## Claim Structure

The claim is defined using CDDL as follows:

~~~
key-binding = {
subject-public-key: public-key,
pop-algorithm: int / tstr,
pop-signature: bstr,

; Optional key protection attributes 
? extractable: bool,
? never-extractable: bool,
? sensitive: bool,
? local: bool,

; Optional cryptographic usage constraints
? purpose: [* oid]

}

public-key = COSE_Key / JWK
; COSE_Key as defined in RFC9052
; JWK as defined in RFC7517
oid = tstr  ; dotted-decimal OID string

~~~

* The `subject-public-key` field contains the public component corresponding to the private key used to generate the `pop-signature`.

* The `pop-algorithm` field identifies the signature algorithm used to generate the `pop-signature`.

* In CBOR-based EAT, the public key MUST be encoded as a COSE_Key, the `pop-algorithm` MUST be encoded as an integer, and the `pop-signature` MUST be a byte string. In JSON-based EAT, the public key MUST be encoded as a JWK, the `pop-algorithm` MUST be encoded as a text string, and the `pop-signature` MUST be Base64URL-encoded.

The signature algorithm identified by `pop-algorithm` MUST be cryptographically valid for the key type and key material of the `subject-public-key`. For example, an RSA public key MUST NOT be used with an ECDSA or EdDSA signature algorithm, and an elliptic curve public key MUST use a signature algorithm appropriate for its curve. If the algorithm is not compatible with the `subject-public-key`, binding verification MUST fail.


## subject-public-key

The `subject-public-key` field contains the public key corresponding to the private key used to generate the Proof of Possession (PoP).

When comparing the subject-public-key contained in the claim with the public key used in a CSR or TLS end-entity certificate, the comparison MUST be performed over the mathematical public key parameters rather than over their serialized encodings. This ensures that differences in encoding formats (e.g., ASN.1 DER versus CBOR) do not cause two equivalent public keys to be incorrectly treated as unequal.

For example:

* For RSA keys: The modulus (n) and public exponent (e) MUST match.
* For elliptic curve keys: The curve identifier and public key coordinates (e.g., x and y values) MUST match. If an implementation supports point compression, keys MUST be decompressed to a common format before the comparison is performed.
* For ML-DSA, SLH-DSA, and FN-DSA keys: The comparison MUST be performed over the raw public key byte string defined by the relevant algorithm specification (e.g., FIPS-204 for ML-DSA). In the claim, the Verifier MUST extract the raw public key bytes from the `pub` parameter of that structure. In an X.509 certificate {{!RFC5280}}, the public key is carried in the SubjectPublicKeyInfo structure. The Verifier MUST extract the contents of the `subjectPublicKey` BIT STRING and obtain the contained public key byte string. The raw public key byte string extracted from the `subject-public-key` and the byte string extracted from the certificate MUST match exactly.
* For other key types: The public key parameters defined by the relevant cryptographic specification MUST match exactly. Comparison based solely on serialized encodings (e.g., raw CBOR, JSON, or DER byte sequences) is NOT RECOMMENDED, as differences in encoding rules may cause equivalent keys to appear unequal.

If the comparison fails, the binding verification MUST fail, even if the attestation evidence itself is otherwise valid. The Verifier MUST convey the outcome of the binding verification to the Relying Party as part of its appraisal result.

## pop-signature

The `pop-signature` field contains a digital signature generated using the private component of the Subject Key over the verifier-provided nonce. The signature MUST be generated using the signature algorithm identified by `pop-algorithm`. The `pop-signature` MUST be verifiable using the corresponding `subject-public-key` and the algorithm identified by `pop-algorithm`.

## Key Protection Attributes

The key-binding claim can include attributes describing key protection properties of the private component of the Subject Key.

When present, the attributes `extractable`, `never-extractable`, `sensitive`, and `local` MUST follow the definitions and semantics specified in {{!I-D.ietf-rats-pkix-key-attestation}}.

This document does not redefine these attributes. Their interpretation and security semantics are defined in {{!I-D.ietf-rats-pkix-key-attestation}}.

A Verifier will evaluate these attributes as part of its security policy when determining whether the Subject Key satisfies requirements for key generation, storage, or exportability.

## Purpose

The `purpose` parameter identifies the key capabilities associated with the Subject Key.

The value of this parameter is a list of object identifiers (OIDs) identifying the key capabilities defined in {{!I-D.ietf-rats-pkix-key-attestation}}.

These OIDs correspond to the key capability identifiers defined in Section 5.2.5 of {{!I-D.ietf-rats-pkix-key-attestation}}.

When the key-binding claim is used in certificate enrollment workflows, the reported key capabilities MUST be compatible with the KeyUsage extensions requested in the CSR and included in the issued certificate.

# Security Considerations

## Threat Model

This document addresses Key Substitution Attacks. In such attacks, valid attestation evidence from a protected execution environment is presented to a verifier while a private key used in certificate enrollment or TLS authentication is not generated or protected within that environment.

The mechanism defined in this document assumes:

- The Attestation Key (AK) is securely provisioned and protected.
- The AK correctly signs attestation evidence reflecting the platform state.
- The Subject Key private component is accessible only within the protected execution environment at the time the Proof of Possession (PoP) is generated.
- The verifier provides an unpredictable nonce to ensure freshness.

If these assumptions do not hold, the security guarantees of this mechanism do not apply.

## Proof-of-Possession Rationale

The PoP prevents reliance on self-reported claims about the presence of the Subject Key by requiring the attested environment to demonstrate its ability to perform a cryptographic operation using the private component of that key.

## Key Substitution Attack Illustration

The following example illustrates how the mechanism detects a Key Substitution Attack.

1. An attacker generates a private key outside the trusted execution environment, denoted as K_bad.
2. The attacker also possesses or obtains attestation evidence for a different key protected within the trusted execution environment, denoted as K_good.
3. The attacker submits a certificate signing request (CSR) signed using K_bad while attaching an EAT containing a claim for K_good.
4. During verification, the Subject Public Key contained in the claim (corresponding to K_good) is compared with the public key contained in the CSR (corresponding to K_bad).

Because the public keys do not match, the binding verification fails. Although the attestation evidence and the CSR signature may each be valid independently, the mismatch prevents the establishment of a cryptographic binding between the certificate private key and the attested execution environment.


## Mitigation of Key Substitution Attacks

This mechanism mitigates Key Substitution Attacks by requiring cryptographic proof that the private key used in a CSR or TLS authentication flow corresponds to the key whose protection within the attested execution environment is being asserted.

Because the proof of possession is included within the EAT and validated together with the EAT signature, and because the claimed Subject Public Key must match the public key contained in the CSR or in the TLS end-entity certificate, substitution of a private key that is not generated or protected within the attested execution environment is detected.

If any of these validations fail, the binding is not established.

## Claim Omission and Downgrade

If a security policy requires that the private key corresponding to a certificate be generated or protected within an attested execution environment, the Relying Party MUST ensure that the key-binding claim defined in this document is present and that binding verification succeeds.

In deployments using a separate Verifier, the Relying Party MUST require the Verifier to enforce the presence and successful validation of the key-binding claim as part of attestation appraisal.


## Scope of Guarantees

This mechanism provides cryptographic evidence that the entity producing the attestation evidence demonstrated control of the private component of the Subject Key at the time the evidence was generated.

It does not guarantee:

- That the key remains protected after attestation;
- That the key cannot later be exported or migrated;
- That the platform remains in the same state after evidence generation.

Such guarantees depend on platform-specific properties and lifecycle management outside the scope of this document.

# IANA Considerations {#IANA}

This document requests registration of a new claim in the following registries:

* "CBOR Web Token (CWT) Claims" registry (established by {{!RFC8392}})
* "JSON Web Token Claims" registry (established by {{!RFC7519}})

The following value is to be added to both registries:

Claim Name: Key Binding
JWT Claim Name: key-binding
CWT Claim Key: TBD
Claim Description: Cryptographic binding between a Subject Key and the attested platform state, including a proof-of-possession signature generated by the Subject Key.
Claim Value Type: CBOR map
Change Controller: IETF
Reference: RFCXXXX

# Acknowledgments
{: numbered="false"}

TODO
