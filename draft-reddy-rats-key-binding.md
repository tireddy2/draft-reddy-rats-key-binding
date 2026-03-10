---
title: "Key Attestation for Entity Attestation Tokens (EAT)"
abbrev: "EAT Key Attestation"
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

 -
    fullname: Hannes Tschofenig
    organization: University of the Bundeswehr Munich
    abbrev: UniBw M.
    city: Neubiberg
    region: Bavaria
    country: Germany
    email: "hannes.tschofenig@gmx.net"

normative:

informative:



--- abstract

This document defines a CWT-based Entity Attestation Token (EAT) profile and a new EAT claim that cryptographically bind a private key used to sign a certificate signing request (CSR), or the private key corresponding to an end-entity certificate used for TLS authentication, to an attested execution environment.

The subject public key is conveyed using the EAT `cnf` claim defined in {{!RFC8747}}, freshness uses the EAT `nonce` claim defined in {{!RFC9711}}, and proof of possession is provided by a COSE countersignature generated with the subject private key. Because the EAT is signed by a hardware-backed Attestation Key (AK), successful verification of both signatures establishes a cryptographic binding between the private key and the attested platform state. Optionally, a Platform Attestation Token (PAT) can be embedded or linked using a Conceptual Message Wrapper (CMW) as defined in {{!I-D.ietf-rats-msg-wrap}}. This mechanism addresses key substitution attacks that arise when attestation evidence and the certificate private keys are validated independently.


--- middle

# Introduction

Remote attestation enables an entity to produce attestation evidence that a verifier can use to assess whether the entity’s platform state satisfies a required policy. In certificate enrollment and authentication protocols (e.g., TLS), a common security policy requirement is that a private key used by an endpoint must be generated, stored, and protected within a trusted execution environment (TEE) or comparable hardware root of trust.

In certificate enrollment workflows, a Certification Authority (CA) may require attestation evidence demonstrating that the private key corresponding to the public key in a certificate signing request (CSR) is protected by a hardware-backed environment. The LAMPS CSR Attestation specification {{!I-D.ietf-lamps-csr-attestation}} defines mechanisms for including attestation evidence alongside a CSR. In this model, the CA verifies the CSR signature using the public key contained in the request and independently verifies the attestation evidence according to the RATS architecture, using applicable endorsements and trust anchors.
However, attestation evidence does not inherently provide a cryptographic proof that the private key used to sign the CSR is the same key that is generated, stored, or protected within the attested environment. The CSR signature demonstrates possession of a private key, and the attestation demonstrates properties of a platform state, but there is no standardized mechanism that cryptographically binds these two validations together. An endpoint could present valid attestation evidence from a protected environment while submitting a certificate signing request (CSR) that is signed with a private key not generated or stored within that environment. In this case, the Certification Authority has no intrinsic cryptographic assurance that the private key corresponding to the CSR public key benefits from the protections described in the attestation evidence.

A similar problem exists in TLS-based scenarios. The TLS Exported Attestation specification {{!I-D.fossati-tls-exported-attestation}} and the TLS Early Attestation specification {{!I-D.fossati-seat-early-attestation}} define mechanisms for conveying attestation evidence within a TLS connection. While the attestation evidence is bound to the TLS connection in these approaches, it does not intrinsically bind the attested environment to the private key corresponding to the end-entity certificate used for TLS authentication. An endpoint could therefore obtain valid attestation evidence from a protected environment while performing certificate-based TLS authentication using a private key that is not confined to that environment. For example, the TLS private key may reside outside the trusted execution environment and lack the protections claimed by the attestation evidence.

This separation between validation of attestation evidence and validation of certificate private key creates a class of key substitution attacks. In such an attack:

* A valid attestation produced by a genuine hardware-protected environment is presented to the verifier; and
* A private key that is not generated or protected within that environment is used in the CSR or TLS authentication flow.

Because the attestation evidence and the certificate private key are validated independently, the verifier has no intrinsic cryptographic assurance that the operational private key benefits from the protections described in the attestation evidence. This undermines security policy objectives that require the certificate private key to be generated and constrained within the attested environment.

Addressing this problem requires a mechanism that provides both proof of possession of the private key and a cryptographic binding between that key and the attested platform state.

A relying party will also require additional claims describing key protection properties, such as non-exportability or hardware-level protection. For example, {{!I-D.ietf-rats-pkix-key-attestation}} defines an evidence format for reporting properties of cryptographic modules and managed keys in PKIX environments, but it does not provide cryptographic proof of possession of the subject key by the attester. The PKIX Key Attestation specification {{!I-D.ietf-rats-pkix-key-attestation}} defines attributes including extractable, never-extractable, sensitive, and local that describe protection properties of keys managed by cryptographic modules. These attributes can be conveyed using the `key-attributes` claim defined in this document while key confirmation itself is conveyed using `cnf` ({{!RFC8747}}) and a COSE countersignature ({{!RFC9338}}).

Appendix A.1.4 of {{!RFC9711}} illustrates how a key and key store may be represented in evidence. However, the example uses private-use claim labels and does not define standardized key-protection claims or a proof-of-possession mechanism for the attested key. This specification uses the standardized `cnf` claim from {{!RFC8747}} together with a COSE countersignature from {{!RFC9338}} and defines a new claim for key-protection attributes and usage constraints.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The reader is assumed to be familiar with the vocabulary and concepts defined in the RATS Architecture ({{!RFC9334}}) such as Attester, Relying Party, Verifier.

The reader is assumed to be familiar with common vocabulary and concepts defined in {{!RFC5280}} such as certificate, signature, attribute, verification and validation.

The following terms are used in this document:

Attestation Key (AK): A cryptographic key, typically hardware-backed, used to sign attestation evidence that conveys claims about the state of a platform or trusted execution environment.

Attestation Evidence: Claims about a platform's state, signed by an Attestation Key, and conveyed in an Entity Attestation Token (EAT).

Entity Attestation Token (EAT): A token format that conveys attestation claims, as defined in {{?RFC9711}}

Subject Key: An asymmetric key pair for which the protection of the private component within an attested execution environment is being asserted. The private component is used to generate the proof of possession (PoP), and the corresponding public component is conveyed in the EAT `cnf` claim and compared with the key used in a CSR or TLS end-entity certificate.

Proof of Possession (PoP): A digital signature generated using the private component of the Subject Key to demonstrate control of that private key at a given point in time.

Verifier: The entity that validates attestation evidence and evaluates whether the platform state and associated keys satisfy its policy.

Certification Authority (CA): An entity that issues certificates and may require attestation evidence during certificate enrollment.

Trusted Execution Environment (TEE): An isolated execution context that provides confidentiality and integrity protections for code and data, including cryptographic key material.

Key Substitution Attack: An attack in which valid attestation evidence from a protected execution environment is presented to a verifier, while a private key used in a CSR or authentication protocol is not generated or protected within that environment. Because attestation evidence and certificate private key are validated independently, the verifier may lack cryptographic assurance that the certificate private key benefits from the protections claimed in the attestation.

# Key Confirmation and Binding Profile

## Overview

This document defines a CWT-based EAT profile and a new EAT claim that establish a cryptographic binding between a Subject Key and an attested execution environment.
The profile is defined for EAT conveyed as a CWT Claims Set in a `COSE_Sign1` message.

The profile provides two properties:

1. Proof of Possession : Demonstration that the entity presenting the attestation controls the private component of the Subject Key.
2. Key-to-Platform Binding : Cryptographic association between the private component of the Subject Key and the attested platform state.

The mechanism operates using two signatures over the same EAT:

- an EAT signature generated by the Attestation Key (AK); and
- a COSE countersignature generated by the private component of the Subject Key.

The subject public key used to verify the countersignature is carried in the EAT `cnf` claim as a `COSE_Key`, following {{!RFC8747}}. Because the attestation evidence is authenticated by the AK, and the countersignature provides PoP for the Subject Key, successful verification establishes that:

- The attested platform state is authentic; and
- The entity producing the attestation evidence demonstrates control of the private component of the Subject Key at the time the evidence was generated.

This construction creates a cryptographic linkage between the Subject Key and the attested platform state, mitigating Key Substitution Attacks.

The `key-attributes` claim conveys attributes describing key protection properties and permitted usage of the Subject Key. The `key-attributes` claim MUST be present and MUST contain at least one member. When present, attributes such as extractable, never-extractable, sensitive, and local MUST follow the semantics defined in {{!I-D.ietf-rats-pkix-key-attestation}}. These attributes enable relying parties to enforce policies such as requiring keys to be generated within the attested environment and prohibiting extraction of private key material.

## High-Level Construction

At a high level, the mechanism binds a certificate private key to an attested execution environment through a dual-signature construction.

First, the attester constructs an EAT Claims Set including:

- the verifier-provided `nonce` claim from {{!RFC9711}};
- the `cnf` claim from {{!RFC8747}} containing the Subject Public Key as `COSE_Key`;
- the `key-attributes` claim defined in this document.

Second, the EAT is signed using the Attestation Key.

Third, a COSE countersignature is generated using the private component of the Subject Key and attached to the EAT as specified by {{!RFC9338}}.

During verification, three relationships are checked:

- The digitally signed and protected EAT establishes that the platform state is authentic.
- The countersignature establishes control of the private component of the Subject Key.
- The public key in `cnf` is compared with the public key in the CSR or TLS end-entity certificate to ensure they refer to the same operational key.

When these checks succeed, the verifier gains assurance that the certificate private key used in the protocol is the same key whose protection within the attested execution environment is being asserted.

## Optional Linkage to Platform Attestation Token (PAT) via CMW

This profile optionally allows carrying a Platform Attestation Token (PAT) using the CWT `cmw` claim defined in {{!I-D.ietf-rats-msg-wrap}}.

When the `cmw` claim is present:

- The `cmw` claim value MUST be a CMW Collection encoded as `cbor-collection`, following Section 4.3.1 of {{!I-D.ietf-rats-msg-wrap}}.
- The Collection MUST contain at least one CMW entry that conveys platform attestation Evidence (PAT).
- The PAT and the key attestation evidence in this profile MUST be cryptographically bound to the same appraisal context.

This profile uses nonce linkage for that binding: The PAT nonce claim MUST be present, and its value MUST match the top-level EAT `nonce`.

# Proof-of-Possession

The Proof of Possession (PoP) demonstrates control of the private component of the Subject Key at the time the attestation evidence is generated.

PoP is provided by a COSE countersignature as defined in {{!RFC9338}} over the EAT object. The countersignature MUST be generated using the private component of the Subject Key, and the corresponding public key MUST be carried in the EAT `cnf` claim as a `COSE_Key` according to {{!RFC8747}}.

For this profile, the EAT `nonce` claim defined in {{!RFC9711}} is the mandatory freshness mechanism. The EAT `nonce` claim MUST be present and MUST contain a single nonce value supplied by the verifier.

The nonce MUST be supplied by the verifier and MUST be unpredictable and unique within the verifier's replay window.

Because PoP is conveyed as a countersignature on the AK-signed EAT, successful verification establishes that:

- The attested platform state is authentic.
- The entity producing the attestation evidence demonstrates control of the private component of the Subject Key.

The validity period of the key attestation evidence is determined by the lifetime of the enclosing EAT. Verifiers MUST enforce the `iat`, `nbf`, and `exp` claims defined in {{!RFC9711}} to ensure that attestation evidence and the associated proof of possession are not used outside their intended validity window.

## Freshness Requirements

The verifier MUST provide a nonce with sufficient entropy to prevent replay. The nonce MUST be unpredictable and unique within the verifier's replay window. The verifier MUST validate that the nonce claim in the EAT matches the nonce it supplied. Failure to include verifier-provided freshness renders the mechanism vulnerable to replay of previously valid attestation evidence.

The verifier-provided nonce is the primary mechanism for ensuring freshness of the proof of possession. The EAT time-based claims (`iat`, `nbf`, and `exp`) provide an additional validity window for the attestation evidence but do not replace the requirement for a verifier-provided nonce. Verifiers MUST validate both the nonce and the applicable time-based claims when evaluating this profile.

## Verification Procedure

Upon receipt of attestation evidence for this profile, the Verifier MUST perform the following checks:

1. Validate the signature on the EAT using the applicable trust anchors and endorsements for the Attestation Key. If this validation fails, the attestation evidence MUST be rejected.

2. Validate the `key-attributes` claim. The `key-attributes` claim MUST be present and MUST contain at least one member.

3. Validate the EAT `nonce` claim. The EAT `nonce` claim MUST be present, MUST contain a single nonce value, and MUST match the verifier-supplied nonce.

4. Extract the Subject Public Key from the EAT `cnf` claim. This profile requires the `cnf` claim defined in {{!RFC8747}} and requires `cnf` to contain `COSE_Key`. Use of `Encrypted_COSE_Key` or a `kid`-only representation is outside the scope of this profile.

5. Validate the COSE countersignature using the extracted Subject Public Key. A countersignature MUST be present. If countersignature verification fails, the binding verification MUST fail.

6. Compare the Subject Public Key contained in `cnf` with the public key:
   - contained in the CSR, in certificate enrollment workflow; or
   - contained in the end-entity certificate used for TLS authentication.

   If the public key parameters do not match, the binding verification MUST fail.

7. If the `cmw` claim is present, validate CMW processing according to {{!I-D.ietf-rats-msg-wrap}}, including:
   - confirmation that `cmw` is a CMW Collection (`cbor-collection`);
   - verification of the PAT Evidence conveyed in the Collection according to the PAT format and trust model; and
   - verification that PAT and key attestation evidence are bound to the same appraisal context by matching nonce values.

Successful completion of all checks establishes a cryptographic binding between the private component corresponding to the public key used in the CSR or TLS end-entity certificate and the attested execution environment at the time the evidence was generated.

The Verifier conveys the result of the binding verification to the Relying Party as part of the attestation result.

# Claim Definition

This document defines a new EAT claim named `key-attributes` that conveys key protection attributes and key-usage constraints for the Subject Key.

## Claim Structure

The claim is defined using CDDL as follows:

~~~
key-attributes = {
; Optional key protection attributes
? extractable: bool,
? never-extractable: bool,
? sensitive: bool,
? local: bool,

; Optional cryptographic usage constraints
? purpose: [* oid]

}

oid = tstr  ; dotted-decimal OID string

~~~

The `key-attributes` claim MUST contain at least one member.

## Subject Key in cnf

This profile uses the EAT `cnf` claim defined in {{!RFC8747}} to carry the Subject Public Key. The `cnf` claim MUST be present and MUST contain a `COSE_Key` member.

When comparing the Subject Public Key contained in `cnf` with the public key used in a CSR or TLS end-entity certificate, the comparison MUST be performed over the public key parameters rather than over their serialized encodings. This ensures that differences in encoding formats (e.g., ASN.1 DER versus CBOR) do not cause two equivalent public keys to be incorrectly treated as unequal.

For example:

* For RSA keys: The modulus (n) and public exponent (e) MUST match.
* For elliptic curve keys: The curve identifier and public key coordinates (e.g., x and y values) MUST match. If an implementation supports point compression, keys MUST be decompressed to a common format before the comparison is performed.
* For ML-DSA, SLH-DSA, and FN-DSA keys: The comparison MUST be performed over the raw public key byte string defined by the relevant algorithm specification (e.g., FIPS-204 for ML-DSA). In `cnf`, the Verifier MUST extract the raw public key bytes from the `pub` parameter of that structure. In an X.509 certificate {{!RFC5280}}, the public key is carried in the SubjectPublicKeyInfo structure. The Verifier MUST extract the contents of the `subjectPublicKey` BIT STRING and obtain the contained public key byte string. The raw public key byte string extracted from `cnf` and the byte string extracted from the certificate MUST match exactly.
* For other key types: The public key parameters defined by the relevant cryptographic specification MUST match exactly. Comparison based solely on serialized encodings (e.g., raw CBOR, JSON, or DER byte sequences) is NOT RECOMMENDED, as differences in encoding rules may cause equivalent keys to appear unequal.

If the comparison fails, the binding verification MUST fail, even if the attestation evidence itself is otherwise valid. The Verifier MUST convey the outcome of the binding verification to the Relying Party as part of its appraisal result.

## Optional cmw Claim Usage

This document does not define a new `cmw` claim. If present, `cmw` MUST follow the syntax and semantics defined in {{!I-D.ietf-rats-msg-wrap}}.

## COSE Countersignature

This profile requires a COSE countersignature as defined in {{!RFC9338}} to provide proof of possession of the Subject Key. The countersignature MUST be verifiable using the Subject Public Key from `cnf`.

## Key Protection Attributes

The `key-attributes` claim includes attributes describing key protection properties of the private component of the Subject Key.

When present, the attributes `extractable`, `never-extractable`, `sensitive`, and `local` MUST follow the definitions and semantics specified in {{!I-D.ietf-rats-pkix-key-attestation}}.

This document does not redefine these attributes. Their interpretation and security semantics are defined in {{!I-D.ietf-rats-pkix-key-attestation}}.

A Verifier will evaluate these attributes as part of its security policy when determining whether the Subject Key satisfies requirements for key generation, storage, or exportability.

## Purpose

The `purpose` parameter identifies the key capabilities associated with the Subject Key.

The value of this parameter is a list of object identifiers (OIDs) identifying the key capabilities defined in {{!I-D.ietf-rats-pkix-key-attestation}}.

These OIDs correspond to the key capability identifiers defined in Section 5.2.5 of {{!I-D.ietf-rats-pkix-key-attestation}}.

When the `key-attributes` claim is used in certificate enrollment workflows, the reported key capabilities MUST be compatible with the KeyUsage extensions requested in the CSR and included in the issued certificate.

# Security Considerations

## Threat Model

This document addresses Key Substitution Attacks. In such attacks, valid attestation evidence from a protected execution environment is presented to a verifier while a private key used in certificate enrollment or TLS authentication is not generated or protected within that environment.

The mechanism defined in this document assumes:

- The AK is securely provisioned and protected.
- The AK correctly signs attestation evidence reflecting the platform state.
- The Subject Key private component is accessible only within the protected execution environment at the time the PoP is generated.
- The verifier provides an unpredictable nonce to ensure freshness.

If these assumptions do not hold, the security guarantees of this mechanism do not apply.

## Proof-of-Possession Rationale

The AK signature over the EAT provides indirect evidence about the Subject Key. The attested environment reports that the key exists and that it is protected within the environment, and the AK signature ensures the integrity and authenticity of those claims.

The PoP defined in this document provides direct evidence of control of the Subject Key. By generating a COSE countersignature using the private component of the Subject Key over the EAT object, the attested environment demonstrates its ability to perform a cryptographic operation with that key.

By requiring this demonstration, the PoP prevents reliance solely on self-reported claims about the presence of the Subject Key in the attested environment.

## Key Substitution Attack Illustration

The following example illustrates how the mechanism detects a Key Substitution Attack.

1. An attacker generates a private key outside the trusted execution environment, denoted as K_bad.
2. The attacker also possesses or obtains attestation evidence for a different key protected within the trusted execution environment, denoted as K_good.
3. The attacker submits a certificate signing request (CSR) signed using K_bad while attaching an EAT containing `cnf` for K_good.
4. During verification, the Subject Public Key contained in `cnf` (corresponding to K_good) is compared with the public key contained in the CSR (corresponding to K_bad).

Because the public keys do not match, the binding verification fails. Although the attestation evidence and the CSR signature may each be valid independently, the mismatch prevents the establishment of a cryptographic binding between the certificate private key and the attested execution environment.

## Mitigation of Key Substitution Attacks

This mechanism mitigates Key Substitution Attacks by requiring cryptographic proof that the private key used in a CSR or TLS authentication flow corresponds to the key whose protection within the attested execution environment is being asserted.

Because proof of possession is conveyed by a countersignature and validated together with the EAT signature, and because the claimed Subject Public Key in `cnf` must match the public key contained in the CSR or in the TLS end-entity certificate, substitution of a private key that is not generated or protected within the attested execution environment is detected.

If any of these validations fail, the binding is not established.

## CMW Linkage Security

If the optional `cmw` claim is used to carry a PAT, Verifiers MUST ensure that the PAT cannot be mixed with an unrelated key attestation token. Verifiers MUST validate nonce linkage using the mandatory nonce claims.

Inclusion of PAT in a CMW Collection does not, by itself, establish trust in PAT contents. Verifiers MUST perform full PAT signature and appraisal verification according to the PAT format and trust model.

## Claim Omission and Downgrade

If a security policy requires that the private key corresponding to a certificate be generated or protected within an attested execution environment, the Relying Party MUST ensure that the `key-attributes` claim defined in this document is present, that the EAT `nonce` claim is present, that the `cnf` claim is present with `COSE_Key`, that a valid countersignature is present, and that binding verification succeeds.

If a security policy additionally requires platform attestation linkage via `cmw`, the Relying Party MUST ensure that a valid PAT is present in `cmw` and that the linkage checks defined in this profile succeed.

In deployments using a separate Verifier, the Relying Party MUST require the Verifier to enforce the presence and successful validation of the `key-attributes` claim, EAT `nonce`, `cnf` with `COSE_Key`, and the countersignature as part of attestation appraisal.


## Scope of Guarantees

This mechanism provides cryptographic evidence that the entity producing the attestation evidence demonstrated control of the private component of the Subject Key at the time the evidence was generated.

It does not guarantee:

- That the key remains protected after attestation;
- That the key cannot later be exported or migrated;
- That the platform remains in the same state after evidence generation.

Such guarantees depend on platform-specific properties and lifecycle management outside the scope of this document.

# IANA Considerations {#IANA}

This document requests registration of a new claim in the "CBOR Web Token (CWT) Claims" registry (established by {{!RFC8392}}).

The following value is to be added to this registry:

Claim Name: key-attributes
CWT Claim Key: TBD
Claim Description: Key protection attributes and key-usage constraints associated with the Subject Key identified by the EAT `cnf` claim.
Claim Value Type: CBOR map
Change Controller: IETF
Reference: RFCXXXX

# Acknowledgments
{: numbered="false"}

TODO
