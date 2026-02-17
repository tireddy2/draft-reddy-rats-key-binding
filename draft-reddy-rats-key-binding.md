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

The claim includes a proof of possession generated using the private key and carried within the EAT. Because the EAT is signed by a hardware-backed Attestation Key (AK), successful verification of both the signature on the EAT and the proof of possession establishes a cryptographic binding between the private key and the attested platform state. This mechanism addresses key substitution attacks that arise when attestation evidence and certificate private key are validated independently.


--- middle

# Introduction

## 1. Introduction

Remote attestation enables an entity to produce attestation evidence that a verifier can use to assess whether the entity’s platform state satisfies a required policy. In certificate enrollment and authentication protocols (e.g., TLS), a common security policy requirement is that a private key used by an endpoint must be generated, stored, and protected within a trusted execution environment (TEE) or comparable hardware root of trust.

In certificate enrollment workflows, a Certification Authority (CA) may require attestation evidence demonstrating that the private key corresponding to the public key in a certificate signing request (CSR) is protected by a hardware-backed environment. The LAMPS CSR Attestation specification {{!I-D.ietf-lamps-csr-attestation}} defines mechanisms for including attestation evidence alongside a CSR. In this model, the CA verifies the CSR signature using the public key contained in the request and independently verifies the attestation evidence according to the RATS architecture, using applicable endorsements and trust anchors.
However, the attestation evidence does not inherently provide a cryptographic proof that the private key used to sign the CSR is the same key that is generated, stored, or protected within the attested environment. The CSR signature demonstrates possession of a private key, and the attestation demonstrates properties of a platform state, but there is no standardized mechanism that cryptographically binds these two validations together. An endpoint could present valid attestation evidence from a protected environment while submitting a certificate signing request (CSR) that is signed with a private key not generated or stored within that environment. In this case, the Certification Authority has no intrinsic cryptographic assurance that the private key corresponding to the CSR public key benefits from the protections described in the attestation evidence.

A similar structural gap exists in transport-layer scenarios. The TLS Exported Attestation specification 
{{!I-D.fossati-tls-exported-attestation}} defines mechanisms for conveying attestation evidence within a TLS session. While the attestation evidence is bound to the TLS handshake transcript, it does not intrinsically bind the attested environment to the private key corresponding to the end-entity certificate used for TLS authentication. An endpoint could therefore obtain valid attestation evidence from a protected environment while performing certificate-based TLS authentication using a private key that is not confined to that environment. For example, the TLS private key may reside outside the trusted execution environment and lack the protections claimed by the attestation evidence.

This separation between validation of attestation evidence and validation of certificate private key creates a class of key substitution attacks. In such an attack:

* A valid attestation produced by a genuine hardware-protected environment is presented to the verifier; and  
* A private key that is not generated or protected within that environment is used in the CSR or TLS authentication flow.

Because the attestation evidence and the certificate private key are validated independently, the verifier has no intrinsic cryptographic assurance that the operational private key benefits from the protections described in the attestation. This undermines security policy objectives that require the certificate private key to be generated and constrained within the attested environment.

Addressing this problem requires a mechanism that provides both proof of possession of the private key and a cryptographic binding between that key and the attested platform state.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Attestation Key (AK): A cryptographic key, typically hardware-backed, used to sign attestation evidence that conveys claims about the state of a platform or trusted execution environment.

Attestation Evidence: Claims about a platform’s state, signed by an Attestation Key, and conveyed in an Entity Attestation Token (EAT).

Entity Attestation Token (EAT): A token format that conveys attestation claims, as defined in {{?RFC9334}}

Subject Key: An asymmetric key pair for which the protection of the private component within an attested execution environment is being asserted. The private component is used to generate the proof of possession (PoP), and the corresponding public component is conveyed in the claim and compared with the key used in a CSR or TLS end-entity certificate.

Proof of Possession (PoP): A digital signature generated using the private component of the Subject Key to demonstrate control of thatprivate key at a given point in time.


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
- The private component of the Subject Key is under the control of the attested environment at the time the evidence was generated.

This construction creates a cryptographic linkage between the Subject Key and the attested platform state, mitigating Key Substitution Attacks.

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

The PoP MUST be computed as a digital signature over a verifier-provided nonce.

PoP = Sign_subject_key(verifier_nonce)

The nonce MUST be supplied by the verifier and MUST be unpredictable and unique within the verifier’s replay window. The PoP signature MUST be included within the attestation evidence and covered by the Attestation Key signature over the EAT.

Because the PoP is embedded within AK-signed attestation evidence, successful verification establishes that:

- The attested platform state is authentic; and  
- The Subject Key was under control of the attested environment at the time the evidence was created.

## Freshness Requirements

The verifier MUST provide a nonce with sufficient entropy to prevent replay. The nonce MUST be unpredictable and unique within the verifier’s replay window. The verifier MUST validate that the nonce contained in the PoP input matches the nonce it supplied. Failure to include verifier-provided freshness renders the mechanism vulnerable to replay of previously valid attestation evidence.

## Verification Procedure

Upon receipt of attestation evidence containing this claim, the entity responsible for binding verification (Verifier or Relying Party, depending on deployment architecture) MUST perform the following checks:

1. Validate the signature on the EAT using the applicable trust anchors and endorsements for the Attestation Key. If this validation fails, the attestation evidence MUST be rejected.

2. Extract the Subject Public Key and the Proof of Possession (PoP) from the claim.

3. Validate the PoP signature using the extracted Subject Public Key and the verifier-supplied nonce. If signature verification fails, the binding verification MUST fail.

4. Confirm that the nonce used in the PoP matches the verifier-provided freshness value. If the nonce does not match, the binding verification MUST fail.

5. Compare the Subject Public Key contained in the claim with the public key:
   - contained in the CSR, in certificate enrollment workflows; or  
   - contained in the end-entity certificate used for TLS authentication.

   This comparison MUST be performed using a canonical representation suitable for unambiguous equality checking. If the keys do not match exactly, the binding verification MUST fail.

Successful completion of all checks establishes a cryptographic binding between the private component corresponding to the operational certificate key and the attested execution environment at the time the evidence was generated.


# Claim Definition

This document defines a new EAT claim that conveys a cryptographic binding between a Subject Key and the attested platform state.

## Claim Structure

The claim is defined using CDDL as follows:

key-binding-claim = {
subject-public-key: public-key,
pop-signature: bstr
}

public-key = cose-key / jwk

In CBOR-based EAT, the public key MUST be encoded as a COSE_Key and the pop-signature MUST be a byte string. In JSON-based EAT, the public key MUST be encoded as a JWK and the pop-signature MUST be a Base64URL-encoded string.

To ensure unambiguous verification, the subject-public-key MUST include an algorithm identifier (alg parameter in COSE/JWK) specifying the signature scheme used to generate the pop-signature. 

## subject-public-key

The `subject-public-key` field contains the public key corresponding to the private key used to generate the Proof of Possession (PoP).

When comparing the subject-public-key contained in the claim with the public key used in a CSR or TLS end-entity certificate, the comparison MUST be performed over the mathematical public key parameters rather than over their serialized encodings. This ensures that differences in wire formats (e.g., ASN.1 DER vs. CBOR) do not result in false negatives.

For example:

* For RSA keys: The modulus (n) and public exponent (e) MUST match.
* For elliptic curve keys: The curve identifier and public key coordinates (e.g., x and y values) MUST match. If an implementation supports point compression, keys MUST be decompressed to a common format before the comparison is performed.
* For other key types: The public key parameters defined by the relevant specification (e.g., base points, polynomial coefficients) MUST match exactly. Comparison based solely on serialized encodings (e.g., raw CBOR, JSON, or DER byte sequences) is NOT RECOMMENDED, as it is prone to failure due to different encoding rules or metadata across protocols.

If the comparison fails, the binding verification MUST fail, even if the attestation evidence itself is otherwise valid. The Verifier MUST convey the outcome of the binding verification to the Relying Party as part of its appraisal result.

## pop-signature

The `pop-signature` field contains a digital signature generated using the private component of the Subject Key.
The signature MUST be computed over the verifier-provided nonce and MUST be verifiable using the corresponding `subject-public-key` included in the claim.


# Security Considerations

## Threat Model

This document addresses Key Substitution Attacks. In such attacks, valid attestation evidence from a protected execution environment is presented to a verifier while a private key used in certificate enrollment or TLS authentication is not generated or protected within that environment.

The mechanism defined in this document assumes:

- The Attestation Key (AK) is securely provisioned and protected.
- The Attestation Key correctly signs attestation evidence reflecting the platform state.
- The Subject Key private component is accessible only within the protected execution environment at the time the Proof of Possession (PoP) is generated.
- The verifier provides an unpredictable nonce to ensure freshness.

If these assumptions do not hold, the security guarantees of this mechanism do not apply.

## Key Substitution Attack Illustration

The following example illustrates how the mechanism detects a Key Substitution Attack.

1. An attacker generates a private key outside the trusted execution environment, denoted as K_bad.
2. The attacker also possesses or obtains attestation evidence for a different key protected within the trusted execution environment, denoted as K_good.
3. The attacker submits a certificate signing request (CSR) signed using K_bad while attaching an EAT containing a claim for K_good.
4. During verification, the Subject Public Key contained in the claim (corresponding to K_good) is compared with the public key contained in the CSR (corresponding to K_bad).

Because the public keys do not match, the binding verification fails. Although the attestation evidence and the CSR signature may each be valid independently, the mismatch prevents the establishment of a cryptographic binding between the operational certificate key and the attested execution environment.

## Mitigation of Key Substitution Attacks

This mechanism mitigates Key Substitution Attacks by requiring cryptographic proof that the private key used in a CSR or TLS authentication flow corresponds to the key whose protection within the attested execution environment is being asserted.

Because the proof of possession is included within the EAT and validated together with the EAT signature, and because the claimed Subject Public Key must match the operational certificate key, substitution of a private key that is not generated or protected within the attested execution environment is detected.

If any of these validations fail, the binding is not established.

## Freshness and Replay

The PoP signature MUST be computed over a verifier-provided nonce. If the nonce is predictable, reused, or omitted, an attacker could replay a previously valid attestation token and PoP signature.

Verifiers MUST ensure that nonces are unpredictable and unique within an appropriate replay window.

## Claim Omission and Downgrade

If a security policy requires that the private key corresponding to a certificate be protected within an attested execution environment, the entity responsible for enforcing that policy MUST require the presence of this claim.

An attacker could attempt a downgrade by omitting the claim while still presenting otherwise valid attestation evidence. If the presence of the claim is not enforced when required by policy, the key substitution vulnerability described in this document remains.

## Scope of Guarantees

This mechanism provides cryptographic evidence that the private component of the Subject Key is under control of the attested execution environment at the time the attestation evidence was generated.

It does not guarantee:

- That the key remains protected after attestation;
- That the key cannot later be exported or migrated;
- That the platform remains in the same state after evidence generation.

Such guarantees depend on platform-specific properties and lifecycle management outside the scope of this document.

# IANA Considerations {#IANA}

TODO

# Acknowledgments
{: numbered="false"}

TODO