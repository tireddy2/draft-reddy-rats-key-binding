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

 -
       ins: T. Fossati
       name: Thomas Fossati
       organization: Linaro
       email: thomas.fossati@linaro.org

 -
       ins: I. Mihalcea
       name: Ionut Mihalcea
       organization: Arm Limited
       email: Ionut.Mihalcea@arm.com

normative:

informative:



--- abstract

This document defines an Entity Attestation Token (EAT) profile and a new EAT claim that convey the subject public key and its protection properties within attestation evidence. Combined with protocol-level proof of possession from the surrounding protocol, this establishes a cryptographic binding between a private key and an attested execution environment.

The subject public key is conveyed using the EAT `cnf` claim defined in {{!RFC8747}} and {{!RFC7800}}, and freshness uses the EAT `eat_nonce` claim defined in {{!RFC9711}}. The proof of possession of the subject key is obtained from the surrounding protocol, such as TLS certificate-based authentication or CSR signature verification. Because the EAT is signed by a hardware-backed Attestation Key (AK), successful verification of the EAT signature together with protocol-level proof of possession establishes a cryptographic binding between the private key and the attested platform state. This mechanism addresses key substitution attacks that arise when attestation evidence and the certificate private keys are validated independently.


--- middle

# Introduction

Remote attestation enables an entity to produce attestation evidence that a verifier can use to assess whether the entity's platform state satisfies a required policy. In certificate enrollment and authentication protocols (e.g., TLS), a common security policy requirement is that a private key used by an endpoint must be generated, stored, and protected within a trusted execution environment (TEE) or comparable hardware root of trust.

In certificate enrollment workflows, a Certification Authority (CA) may require attestation evidence demonstrating that the private key corresponding to the public key in a certificate signing request (CSR) is protected by a hardware-backed environment. The LAMPS CSR Attestation specification {{!I-D.ietf-lamps-csr-attestation}} defines mechanisms for including attestation evidence alongside a CSR. In this model, the CA verifies the CSR signature using the public key contained in the request and independently verifies the attestation evidence according to the RATS architecture, using applicable endorsements and trust anchors.
However, attestation evidence does not inherently provide a cryptographic proof that the private key used to sign the CSR is the same key that is generated, stored, or protected within the attested environment. The CSR signature demonstrates possession of a private key, and the attestation demonstrates properties of a platform state, but there is no standardized mechanism that cryptographically binds these two validations together. An endpoint could present valid attestation evidence from a protected environment while submitting a certificate signing request (CSR) that is signed with a private key not generated or stored within that environment. In this case, the Certification Authority has no intrinsic cryptographic assurance that the private key corresponding to the CSR public key benefits from the protections described in the attestation evidence.

A similar problem exists in TLS-based scenarios. The TLS Exported Attestation specification {{!I-D.fossati-tls-exported-attestation}} and the TLS Early Attestation specification {{!I-D.fossati-seat-early-attestation}} define mechanisms for conveying attestation evidence within a TLS connection. While the attestation evidence is bound to the TLS connection in these approaches, it does not intrinsically bind the attested environment to the private key corresponding to the end-entity certificate used for TLS authentication. An endpoint could therefore obtain valid attestation evidence from a protected environment while performing certificate-based TLS authentication using a private key that is not confined to that environment. For example, the TLS private key may reside outside the trusted execution environment and lack the protections claimed by the attestation evidence.

This separation between validation of attestation evidence and validation of certificate private key creates a class of key substitution attacks. In such an attack:

* A valid attestation produced by a genuine hardware-protected environment is presented to the verifier; and
* A private key that is not generated or protected within that environment is used in the CSR or TLS authentication flow.

Because the attestation evidence and the certificate private key are validated independently, the verifier has no intrinsic cryptographic assurance that the operational private key benefits from the protections described in the attestation evidence. This undermines security policy objectives that require the certificate private key to be generated and constrained within the attested environment.

Addressing this problem requires a mechanism that provides both proof of possession of the private key and a cryptographic binding between that key and the attested platform state.

A relying party will also require additional claims describing key protection properties, such as non-exportability or hardware-level protection. For example, {{!I-D.ietf-rats-pkix-key-attestation}} defines an evidence format for reporting properties of cryptographic modules and managed keys in PKIX environments. The PKIX Key Attestation specification {{!I-D.ietf-rats-pkix-key-attestation}} defines attributes including extractable, never-extractable, sensitive, and local that describe protection properties of keys managed by cryptographic modules. These attributes can be conveyed using the `key-attributes` claim defined in this document while key confirmation itself is conveyed using `cnf` ({{!RFC8747}} and {{!RFC7800}}) and protocol-level proof of possession.

Appendix A.1.4 of {{!RFC9711}} illustrates how a key and key store may be represented in evidence. However, the example uses private-use claim labels and does not define standardized key-protection claims. This specification uses the standardized `cnf` claim from {{!RFC8747}} and {{!RFC7800}} and defines a new claim for key-protection attributes and usage constraints, while relying on protocol-level proof of possession.


The use of directly conveyed key protection properties in attestation evidence is consistent with {{!I-D.ietf-rats-pkix-key-attestation}}, which defines attributes describing protection properties of managed keys, such as extractable, never-extractable, sensitive, and local.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The reader is assumed to be familiar with the vocabulary and concepts defined in the RATS Architecture ({{!RFC9334}}) such as Attester, Relying Party, Verifier.

The reader is assumed to be familiar with the common vocabulary and concepts defined in {{!RFC5280}} such as certificate, signature, attribute, verification and validation.

The following terms are used in this document:

Attestation Key (AK): A cryptographic key, typically hardware-backed, used to sign attestation evidence that conveys claims about the state of a platform or trusted execution environment.

Attestation Evidence: Claims about a platform's state, signed by an Attestation Key, and conveyed in an Entity Attestation Token (EAT).

Entity Attestation Token (EAT): A token format that conveys attestation claims, as defined in {{?RFC9711}}.

Subject Key: An asymmetric key pair for which the protection of the private component within an attested execution environment is being asserted. The corresponding public component is conveyed in the EAT `cnf` claim and compared with the key used in a CSR or TLS end-entity certificate.

Proof of Possession (PoP): Evidence that demonstrates control of the private component of the Subject Key at a given point in time. In this profile, PoP is obtained from the surrounding protocol (for example, TLS certificate-based authentication or CSR signature verification).

Verifier: The entity that validates attestation evidence and evaluates whether the platform state and associated keys satisfy its policy.

Certification Authority (CA): An entity that issues certificates and may require attestation evidence during certificate enrollment.

Trusted Execution Environment (TEE): An isolated execution context that provides confidentiality and integrity protections for code and data, including cryptographic key material.

Key Substitution Attack: An attack in which valid attestation evidence from a protected execution environment is presented to a verifier, while a private key used in a CSR or authentication protocol is not generated or protected within that environment. Because attestation evidence and certificate private key are validated independently, the verifier may lack cryptographic assurance that the certificate private key benefits from the protections claimed in the attestation.

Key Attestation Key (KAK): An Attestation Key used specifically for signing Key Attestation Tokens in the split model (see {{architecture}}). The KAK is protected within the trusted computing base, and its public key and protection properties are conveyed in the Platform Attestation Token via the `cnf` and `key-attributes` claims.

Platform Attestation Token (PAT): An EAT signed by the platform Attestation Key that conveys the state of the trusted computing base. In this context, 'platform' refers to the Attester's full TCB. The KAK public key is conveyed via the `cnf` claim, and KAK protection properties via the `key-attributes` claim.

Key Attestation Token (KAT): An EAT signed by the KAK that conveys the Subject Public Key via the `cnf` claim and Subject Key protection properties via the `key-attributes` claim.

Combined Attestation Bundle (CAB): A structure that bundles a Key Attestation Token (KAT) and a Platform Attestation Token (PAT) together for transport in the surrounding protocol, using the CMW collection format {{!I-D.ietf-rats-msg-wrap}}.

The public key carried in the `cnf` claim is referred to as the Subject Public Key in this document, reflecting its use as the public key carried in the SubjectPublicKeyInfo field of an X.509 certificate {{!RFC5280}} in certificate enrollment and TLS authentication. In the context of the `cnf` claim defined in {{!RFC8747}} and {{!RFC7800}}, this key serves as the proof-of-possession key.

# Architecture {#architecture}

This document defines two deployment models for key attestation:

**Combined Model:** The key attestation function and platform attestation function are hosted within the same trusted execution environment. A single Attestation Key (AK) signs one EAT containing both platform state claims and key protection claims (`cnf` and `key-attributes`). This is the simpler model and is suitable for platforms where separation of privilege between key attestation and platform attestation is not required.

**Split Model:** The key attestation function and platform attestation function are separate logical roles within the trusted computing base, operating at different privilege levels. Platform attestation requires higher privilege than key attestation, and the split model allows the key attestation function to operate at a lower privilege level within the TCB. A dedicated Key Attestation Key (KAK) signs the Key Attestation Token (KAT), and a separate Platform Attestation Token (PAT) is generated and signed by the platform Attestation Key. The two tokens are bundled together as a Combined Attestation Bundle (CAB) for transport.

In both models, proof of possession of the Subject Key is obtained from the surrounding protocol, as described in {{proof-of-possession}}.

# Key Confirmation and Binding Profile

## Overview {#key-confirmation-overview}

A foundational requirement of this profile is that the Subject Key MUST be generated and held within the attested execution environment. This is what gives the `key-attributes` claims their authority, the attested environment signs attestation evidence about key material it generated and controls.

This document defines an EAT profile and a new EAT claim that establish a cryptographic binding between a Subject Key and an attested execution environment.

The profile provides two properties:

1. Proof of Possession : Demonstration that the entity presenting the attestation controls the private component of the Subject Key.
2. Key-to-Platform Binding : Cryptographic association between the private component of the Subject Key and the attested platform state.

The mechanism combines:

- an EAT signature generated by the Attestation Key (AK); and
- protocol-level proof of possession for the Subject Key.

The subject public key used for protocol-level PoP verification is carried in the EAT `cnf` claim. Because the attestation evidence is authenticated by the AK, and PoP is verified in the surrounding protocol, successful verification establishes that:

- The attested platform state is authentic; and
- The entity participating in the surrounding protocol demonstrates control of the private component of the Subject Key during protocol execution.

This construction creates a cryptographic linkage between the Subject Key and the attested platform state, mitigating Key Substitution Attacks.

The `key-attributes` claim conveys attributes describing key protection properties and permitted usage of the Subject Key. The `key-attributes` claim MUST be present and MUST contain at least one member. When present, attributes such as extractable, never-extractable, sensitive, and local MUST follow the semantics defined in {{!I-D.ietf-rats-pkix-key-attestation}}. These attributes enable relying parties to enforce policies such as requiring keys to be generated within the attested environment and prohibiting extraction of private key material.

## High-Level Construction {#high-level-construction}

At a high level, the mechanism binds a certificate private key to an attested execution environment by combining AK-signed attestation evidence and protocol-level proof of possession.

First, the attester constructs an EAT Claims Set including:

- the verifier-provided `eat_nonce` claim from {{!RFC9711}};
- the `cnf` claim containing the Subject Public Key;
- the `key-attributes` claim defined in this document.

Second, the EAT is signed using the Attestation Key.

During verification, three relationships are checked:

- The digitally signed and protected EAT establishes that the platform state is authentic.
- The protocol-level PoP check establishes control of the private component of the Subject Key.
- The public key in `cnf` is compared with the public key in the CSR or TLS end-entity certificate to ensure they refer to the same operational key.

When these checks succeed, the verifier gains assurance that the certificate private key used in the protocol is the same key whose protection within the attested execution environment is being asserted.

# Proof-of-Possession {#proof-of-possession}

The Proof of Possession (PoP) demonstrates control of the private component of the Subject Key.

In this profile, PoP MUST be verified by the surrounding protocol:

- In certificate enrollment workflows, by validating the CSR signature.
- In TLS workflows, by validating certificate-based TLS authentication.

The public key used for protocol-level PoP verification MUST correspond to the Subject Public Key in EAT `cnf`.

For this profile, the EAT `eat_nonce` claim defined in {{!RFC9711}} is the mandatory freshness mechanism. The EAT `eat_nonce` claim MUST be present and MUST contain a single nonce value supplied by the verifier.

The nonce MUST be supplied by the verifier and MUST be unpredictable and unique within the verifier's replay window.

The validity period of the key attestation evidence is determined by the lifetime of the enclosing EAT. Verifiers MUST enforce the `iat`, `nbf`, and `exp` claims defined in {{!RFC9711}} to ensure that attestation evidence is not used outside its intended validity window.

## Freshness Requirements

The verifier MUST provide a nonce with sufficient entropy to prevent replay. The nonce is conveyed to the Attester by the Relying Party through the surrounding protocol.
The nonce MUST be unpredictable and unique within the verifier's replay window. The verifier MUST validate that the nonce claim in the EAT matches the nonce it supplied. Failure to include verifier-provided freshness renders the mechanism vulnerable to replay of previously valid attestation evidence. Mechanisms for obtaining and conveying such nonces in certificate enrollment protocols are described in {{!I-D.ietf-lamps-attestation-freshness}}.

The verifier-provided nonce is the primary mechanism for ensuring freshness of the attestation evidence. The EAT time-based claims (`iat`, `nbf`, and `exp`) provide an additional validity window for the attestation evidence but do not replace the requirement for a verifier-provided nonce. Verifiers MUST validate both the nonce and the applicable time-based claims when evaluating this profile.

## Verification Procedure {#verification-procedure}

Upon receipt of attestation evidence for this profile, the Verifier MUST perform the following checks:

1. Validate the signature on the EAT using the applicable trust anchors for the Attestation Key. If this validation fails, the attestation evidence MUST be rejected.

2. Validate the `key-attributes` claim. The `key-attributes` claim MUST be present and MUST contain at least one member. Trust in the `key-attributes` claim depends on successful appraisal of the attestation evidence for the target environment in which the Subject Key is generated and protected. Such appraisal includes evaluation of measurements in the attestation evidence against the applicable reference values as described in the RATS Architecture {{!RFC9334}} and EAT {{!RFC9711}}.

3. Validate the EAT `eat_nonce` claim. The EAT `eat_nonce` claim MUST be present, MUST contain a single nonce value, and MUST match the verifier-supplied nonce.

4. Extract the Subject Public Key from the EAT `cnf` claim. 

5. Compare the Subject Public Key contained in `cnf` with the public key used for protocol-level PoP verification. This public key is either obtained directly from the protocol or supplied to the Verifier by the Relying Party.
   - In certificate enrollment, the public key is obtained from the CSR.
   - In TLS, the public key is obtained from the end-entity certificate used for TLS authentication.

The public key provided to the Verifier MUST correspond to the same key for which protocol-level PoP verification was performed. If the public key parameters do not match, the binding verification MUST fail.

Successful completion of all checks establishes a cryptographic binding between the private component corresponding to the public key used in the CSR or TLS end-entity certificate and the attested execution environment at the time the evidence was generated.

The Verifier conveys the result of the binding verification to the Relying Party as part of the attestation result.

# Split Model Profile

In the split model, the key attestation function and platform attestation function are separate logical roles within the trusted computing base. Two EATs are generated and bundled together as a Combined Attestation Bundle (CAB) using the CMW collection format {{!I-D.ietf-rats-msg-wrap}}:

- A Platform Attestation Token (PAT), signed by the platform Attestation Key, that conveys the state of the trusted computing base, the KAK public key via the `cnf` claim, and KAK protection properties via the `key-attributes` claim.
- A Key Attestation Token (KAT), signed by the Key Attestation Key (KAK), that conveys the Subject Public Key via the `cnf` claim and Subject Key protection properties via the `key-attributes` claim.

The KAT is signed by the KAK whose public key and protection properties are attested in the PAT. This provides cryptographic binding between the two tokens without requiring a separate linkage mechanism.

## Verification Procedure {#split-verification-procedure}

Upon receipt of a CAB for this profile, the Verifier MUST perform the following checks:

1. Validate the signature on the PAT using the applicable trust anchors for the platform Attestation Key. If this validation fails, the attestation evidence MUST be rejected.

2. Validate the EAT `eat_nonce` claim in the PAT. The `eat_nonce` claim MUST be present, MUST contain a single nonce value, and MUST match the verifier-supplied nonce.

3. Validate the `key-attributes` claim in the PAT. The `key-attributes` claim MUST be present and MUST contain at least one member. Trust in the `key-attributes` claim depends on successful appraisal of the PAT against applicable reference values as described in {{!RFC9334}}.

4. Extract the KAK public key from the `cnf` claim in the PAT.

5. Validate the signature on the KAT using the KAK public key extracted in step 4. If this validation fails, the attestation evidence MUST be rejected.

6. Validate the `key-attributes` claim in the KAT. The `key-attributes` claim MUST be present and MUST contain at least one member.

7. Validate the EAT `eat_nonce` claim in the KAT. The `eat_nonce` claim MUST be present, MUST contain a single nonce value, and MUST match the verifier-supplied nonce.

8. Extract the Subject Public Key from the `cnf` claim in the KAT and compare it with the public key used for protocol-level PoP verification, following the same procedure defined in steps 4 and 5 of {{verification-procedure}}.

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

This profile uses the EAT `cnf` claim defined in {{!RFC8747}} and {{!RFC7800}} to carry the Subject Public Key. 

When comparing the Subject Public Key contained in `cnf` with the public key used in a CSR or TLS end-entity certificate, the comparison MUST be performed over the public key parameters rather than over their serialized encodings. This ensures that differences in encoding formats (e.g., ASN.1 DER versus CBOR) do not cause two equivalent public keys to be incorrectly treated as unequal.

For example:

* For RSA keys: The modulus (n) and public exponent (e) MUST match.
* For elliptic curve keys: The curve identifier and public key coordinates (e.g., x and y values) MUST match. If an implementation supports point compression, keys MUST be decompressed to a common format before the comparison is performed.
* For ML-DSA, SLH-DSA, and FN-DSA keys: The comparison MUST be performed over the raw public key byte string defined by the relevant algorithm specification (e.g., FIPS-204 for ML-DSA). In `cnf`, the Verifier MUST extract the raw public key bytes from the `pub` parameter of that structure. In an X.509 certificate {{!RFC5280}}, the public key is carried in the SubjectPublicKeyInfo structure. The Verifier MUST extract the contents of the `subjectPublicKey` BIT STRING and obtain the contained public key byte string. The raw public key byte string extracted from `cnf` and the byte string extracted from the certificate MUST match exactly.
* For other key types: The public key parameters defined by the relevant cryptographic specification MUST match exactly. Comparison based solely on serialized encodings (e.g., raw CBOR, JSON, or DER byte sequences) is NOT RECOMMENDED, as differences in encoding rules may cause equivalent keys to appear unequal.

If the comparison fails, the key-to-platform binding is not established.

## Protocol-Based PoP

This profile relies on protocol-level proof of possession for the Subject Key. This document does not define a new in-token PoP container or signature format.

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
- The surrounding protocol correctly verifies proof of possession of the Subject Key private component.
- The verifier provides an unpredictable nonce to ensure freshness.
- The Attester generates the Subject Key within the attested execution environment and does not export the private component in cleartext. If this assumption does not hold, the `key-attributes` claims do not accurately reflect the protection properties of the Subject Key, and the security guarantees of this profile do not apply.

For the split model, the following additional assumption applies:

- The platform Attester generates and protects the KAK within the trusted computing base, and the KAK private component is never exported in cleartext. If this assumption does not hold, the `key-attributes` claims in the PAT do not accurately reflect the protection properties of the KAK.

If these assumptions do not hold, the security guarantees of this mechanism do not apply.

## Proof-of-Possession Rationale

The AK signature over the EAT provides evidence about the Subject Key and its asserted protection properties. Protocol-level PoP verification provides direct evidence of control of the Subject Key.

By requiring both checks, the profile prevents reliance solely on self-reported claims about the presence of the Subject Key in the attested environment.

## Binding of Key Protection Claims to the Attested Environment

The `key-attributes` claim conveys protection properties of the Subject Key, such as non-exportability and hardware-level protection. For these claims to be trustworthy, they must be asserted by the attested environment that generated and holds the Subject Key, as it is the only entity with direct knowledge of and authority over the key protection properties.

An alternative construction sometimes used in attestation protocols is to build an unsigned claims set (UCCS) containing the Subject Public Key and associated attributes, hash it, and supply the hash as a challenge to an existing attestation interface. In such constructions, the attestation evidence provides an indirect cryptographic binding between the claims set and the attested environment. However, a TEE-bound process acting as a proxy could forward a fabricated UCCS on behalf of an untrusted caller, causing the attested environment to sign claims it did not generate and cannot verify.

This profile instead requires the Attestation Key (AK) to directly sign the EAT Claims Set containing the `key-attributes` claim and the Subject Public Key in `cnf`. This ensures that key protection attributes are conveyed as claims directly attested by the Attestor, eliminating the risk of fabricated claims being indirectly bound to attestation evidence.

## Key Protection Properties

The `cnf` claim establishes a cryptographic binding between the Subject Key and the attestation evidence. However, the `cnf` claim alone does not convey information about the protection properties of the private component of that key.

Some deployments require assurances regarding how the private key is generated, stored, and protected (for example, whether the key is non-exportable or generated within a hardware-protected environment). The `key-attributes` claim enables the Verifier to evaluate such properties and determine whether the Subject Key satisfies the security requirements of the deployment.

## Key Substitution Attack Illustration

The following example illustrates how the mechanism detects a Key Substitution Attack.

1. An attacker generates a private key outside the trusted execution environment, denoted as K_bad.
2. The attacker also possesses or obtains attestation evidence for a different key protected within the trusted execution environment, denoted as K_good.
3. The attacker submits a certificate signing request (CSR) signed using K_bad while attaching an EAT containing `cnf` for K_good.
4. During verification, the Subject Public Key contained in `cnf` (corresponding to K_good) is compared with the public key contained in the CSR (corresponding to K_bad).

Because the public keys do not match, the binding verification fails. Although the attestation evidence and the CSR signature may each be valid independently, the mismatch prevents the establishment of a cryptographic binding between the certificate private key and the attested execution environment.

## Mitigation of Key Substitution Attacks

This mechanism mitigates Key Substitution Attacks by requiring cryptographic proof that the private key used in a CSR or TLS authentication flow corresponds to the key whose protection within the attested execution environment is being asserted.

Because protocol-level proof of possession is validated together with the EAT signature, and because the claimed Subject Public Key in `cnf` must match the public key used in the protocol, substitution of a private key that is not generated or protected within the attested execution environment is detected.

If any of these validations fail, the binding is not established.

## Claim Omission and Downgrade

If a security policy requires that the private key corresponding to a certificate be generated or protected within an attested execution environment, the Relying Party MUST ensure that the `key-attributes` claim defined in this document is present, that the EAT `eat_nonce` claim is present, that the `cnf` claim is present with the subject public key, that protocol-level PoP verification succeeds, and that binding verification succeeds.

In deployments using a separate Verifier, the Relying Party MUST require the Verifier to enforce the presence and successful validation of the `key-attributes` claim, EAT `eat_nonce`, `cnf` with the subject public key, and protocol-level PoP verification as part of attestation appraisal.

## Scope of Guarantees

This mechanism provides cryptographic evidence that the entity participating in the surrounding protocol demonstrated control of the private component of the Subject Key during protocol execution.

It does not guarantee:

- That the key remains protected after attestation;
- That the key cannot later be exported or migrated;
- That the platform remains in the same state after evidence generation.

Such guarantees depend on platform-specific properties and lifecycle management outside the scope of this document.

# IANA Considerations {#IANA}

This document requests registration of a new claim in the "CBOR Web Token (CWT) Claims" registry (established by {{!RFC8392}}).

The following value is to be added to this registry:

*  Claim Name: key-attributes
*  CWT Claim Key: TBD
*  Claim Description: Key protection attributes and key-usage constraints associated
   with the Subject Key identified by the EAT `cnf` claim.
*  Claim Value Type: CBOR map
*  Change Controller: IETF
*  Reference: RFCXXXX

This document also requests registration of a new claim in the "JSON Web Token (JWT) Claims" registry (established by {{!RFC7519}}).

The following value is to be added to this registry:

* Claim Name: key-attributes
* Claim Description: Key protection attributes and key-usage constraints associated with the Subject Key identified by the EAT `cnf` claim.
* Change Controller: IETF
* Reference: RFCXXXX


# Acknowledgments
{: numbered="false"}

The authors thank Paul Wouters and Nathanael Ritz for the discussion and comments.

--- back
