---
source_url: https://www.uber.com/nl/en/blog/scaling-verify-wallet/
title: Scaling Verify with Wallet for Identity Verification at Uber
authors:
  - Gabriel D'Luca (Sr Software Engineer)
  - Beatriz Rezener (Software Engineer II)
  - Felipe Figueiredo (Staff Software Engineer)
date: June 23, 2026
---

# Scaling Verify with Wallet for Identity Verification at Uber

## Introduction

Identity verification is one of the priorities of Uber's Trusted Identity organization to promote market expansion, compliance, and support legal accountability on the Uber platform. The existing portfolio of verification methods, which includes technical solutions such as [in-house document scanning](https://www.uber.com/en-BR/blog/ubers-real-time-document-check/) and selfie capture, is constantly expanding at a global scale through partnerships with technology leaders in the segment.

As technology evolved, mobile wallets such as Apple Wallet gained the ability to store mobile documents (mDocs), such as mDL (Mobile Driver's License) and photo IDs, allowing users to seamlessly present their IDs in person, in apps, or to websites for identity or age verification.

Adding to the extensive list of mechanisms for verifying a person's identity, Uber was an early adopter of Apple's [Verify with Wallet API](https://developer.apple.com/wallet/get-started-with-verify-with-wallet/)—a reliable and modern solution introduced at WWDC 2022—into multiple use cases in Uber's cross-app Identity Verification Platform.

### About Verify with Wallet

In contrast to the verification methods operating based on document scanning or user input, Verify with Wallet merges convenience, platform security, and seamless user experiences with a privacy focus, allowing Uber apps to verify digital ID data with government-backed provenance. This verification method drastically reduces user friction, improves conversion rates by reducing operational and image processing errors, and increases the number of verified users.

The API also encourages privacy-focused practices, providing full transparency into the identity information apps request and for how long. Apps are entitled to request only the specific data required to complete the transaction, preventing users from having to overshare their identity information. Neither the issuing authority nor Apple can see when and where a user shares their mDL or photo ID.

These powerful capabilities from Verify with Wallet, now incorporated into Uber's Identity Verification Platform, are opening doors to a new era of digital identity verification—one where users can verify their Uber account in seconds rather than minutes, without needing to locate a physical ID. By eliminating this friction, the platform enables a faster and more efficient experience across eligible use cases in Uber, Uber Eats, and Postmates.

### Challenge

Uber operates across multiple apps and marketplace segments, each with legitimately different identity verification needs: a driver onboarding flow requires different identity data than an age-gated ordering flow, which differs again from a car rental scenario.

While Verify with Wallet supports this breadth and scale through an app-level Entitlement configuration, covering the full set of data elements the app is permitted to request, each data element must still be individually approved by Apple through an [Entitlement Request](https://developer.apple.com/contact/request/verify-with-wallet/) for its intended use case. This means that the platform capabilities provided at Uber must be flexible enough to serve any combination of approved data elements, yet strict enough to ensure a given use case never exceeds its own approved scope.

In addition to the orchestration of these data elements, the solution had to address several architectural requirements:

- Building a platform-agnostic mapping layer to translate Uber's internal identity models into domain-specific objects
- Managing high-assurance cryptographic validations according to the ISO/IEC 18013-5 standard
- Establishing a reliable mechanism to track and update trust anchors as government issuing authorities rotate their root certificates

Integrating and orchestrating Verify with Wallet through this scalable, compliant platform—rather than a series of separate one-off integrations—was the core engineering challenge.

## Architecture

![Sequence diagram showing Uber Apps, identity verification service, and PassKit document verification flow](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy83Y2Y1Yzc4My01ZGZkLTRiYmYtODYwNi1mNWE4N2EwMzViMTIucG5n)

*Figure 1: Diagram of Uber's Verify with Wallet integration.*

#### Configuring the App for Multiple Use Cases

By design, each Uber app includes an Entitlements file encompassing the full set of required identity elements for all of its use cases. The enforcement of per-use-case scoping happens in Uber's Identity Verification Service: the back end orchestrates the identity requests, ensuring every request is strictly limited to the data elements pre-vetted for its specific use case. This approach enables platform scalability without compromising on compliance or user privacy.

Each app is also configured with an *NSIdentityUsageDescription*—a message that tells users why the app is requesting identity information—conveying the context for app-level usage. Additionally, a new [usageDescriptionKey](https://developer.apple.com/documentation/passkit/pkidentityrequest/usagedescriptionkey) API was introduced in iOS 26.2, allowing the service to tailor this message for each specific use case upon configuring the PKIdentityRequest.

#### Orchestrating Data Elements

The orchestration is centralized within the Identity Verification Service, which manages the request flow via server-side configurations. These configurations define the specific data elements associated with each use case, along with the corresponding IntentToStore—a parameter specifying whether data is accessed for a one-time verification or persisted for long-term use.

To maintain a scalable architecture, the service implements a mapping layer that translates Uber's internal identity configuration objects into PassKit-specific objects required by the Verify with Wallet API, such as PKIdentityIntentToStore. By decoupling service logic from vendor-specific APIs, the system remains ready to support future integrations compliant with the ISO/IEC 18013-5 standard.

Server-side validations are performed on every response. These checks confirm that the returned data elements match the expected configuration for the active use case. Additional cryptographic validations are then performed on each received data element to ensure origin and integrity.

#### Decrypting and Validating Data

Uber's Identity Verification service handles decryption of the Apple Wallet payload and runs validation checks based on the ISO/IEC 18013-5 standard and the [Apple Developer Documentation](https://developer.apple.com/documentation/passkit/verifying-wallet-identity-requests).

Security in transit relies on HPKE (Hybrid Public Key Encryption, RFC 9180). When the mobile app initiates a request, the response is encrypted using the app's Identity Access Certificate, which is generated from the [Apple Developer website](https://developer.apple.com/account/resources/). The Uber back end then decrypts the payload, which is encoded in CBOR (Concise Binary Object Representation)—a compact, binary format used to keep identity data payloads small and efficient.

A key part of this exchange is the session transcript. This structure binds the response to a specific transaction by combining a unique generated nonce, the requesting client's identifiers, and the public key hash. This cryptographic link prevents replay attacks by ensuring a response can't be intercepted and reused.

Once decrypted, the service executes a three-step validation:

- MSO verification: The MSO (Mobile Security Object) is checked to ensure no data has been modified since issuance and that the current date falls within the MSO's validity period.
- Device binding: The device signature is validated against the public key in the MSO. This ensures the ID is being requested by the original, authorized device and isn't a spoofed payload.
- Issuer certificate: The system verifies the signing of certificate chains to a trusted IACA (Issuing Authority Certificate Authority) root certificate.

The IACA certificate serves as the government-backed root of trust. Verifying this chain confirms the credential's authenticity before the data is passed to Uber's internal identity checks for further use-case-specific checks. Maintaining this chain of trust at scale requires a persistent strategy for managing the rotation of these root certificates.

#### Updating IACA Certificates

Reliable verification requires a consistent root of trust. The system maintains a registry of IACA certificates from every supported issuing authority. These trust anchors are sourced from official jurisdictional portals, such as DMV websites, and the [AAMVA Digital Trust Service VICAL](https://vical.dts.aamva.org/) (Verified Issuer Certificate Authority List)—a centralized trust service that aggregates certificates from North American motor vehicle administrators.

The registry is also designed for certificate rotations. Since jurisdictions often issue new root certificates or rotate authority keys, the system supports multiple active certificates per jurisdiction. This ensures IDs signed with older keys remain valid while newer ones are simultaneously accepted, preventing disruption when a state updates its infrastructure.

We work closely with key ecosystem partners to keep the registry current, handling the ingestion of new authority keys and the deprecation of revoked certificates as they are updated. This keeps our verification logic secure and uninterrupted as jurisdictions update keys and new standards emerge.

### Next Steps

As ‌adoption towards digital identity documents continues to evolve, our focus remains on expanding the reach and technical capabilities of Uber's integration with Verify with Wallet API. The service has been built as a platform to easily accommodate future use cases, with the flexibility to request additional specific data elements to meet the needs of different marketplace segments.

In addition, the following are potential focus areas to Uber's Identity Verification Platform:

- Support new digital document types. The platform is designed to accommodate new document types with high-assurance credentials, such as Photo IDs complying to the ISO/IEC 23220-2 standard. By integrating with a wider range of document types, the service can reach a more diverse user base, while maintaining a high bar for authenticity.
- Cross-platform support. Deepening integration with global identity solutions that adhere to the ISO/IEC 18013-5 standard and the broader digital credentials ecosystem. This commitment to open standards ensures that Uber's Identity Verification service remains platform-agnostic, future-proof, and interoperable with the expanding landscape of government-backed digital credentials.

Integrating Apple's Verify with Wallet marks a significant step forward for our Identity Verification Platform. This feature integration supports acceptance of reliable, government-issued digital IDs, making the identity verification process smoother, boosting successful verifications, and reinforcing our commitment to robust security and privacy.

The privacy-preserving design is built to only request the data required to complete the transaction and prevent users from having to overshare their identity information. Beyond that, the solution employs data encryption with HPKE and session transcript and validates data against the global standard ISO/IEC 18013-5.

This strong, adaptable solution reflects our ongoing dedication to user protection, privacy, and providing a straightforward experience as digital identity evolves globally.

### Acknowledgments

This product was built by a cross-functional team including areas such as design, product management, engineering, legal, marketing, business development, and partnership management.

The following people were instrumental in adopting and scaling Verify with Wallet at Uber, in addition to the authors of this post: Duncan Carey, Allan Fukasawa, Ryan Stentz, Gustavo Tiengo, Flavia Rangel, Boldtulga Ganbaatar, Victoria Reis, Shea Hoffpauer, Shalmali Rajadhyax, Gustavo Daud, Anna Gassot, and Lizzie Ross.

Cover Photo Attribution: Image by Gabriel D'Luca and Victoria Reis. Apple UI elements © Apple Inc.; licensed under the Apple Design Resources License Agreement.

AAMVA is a trademark of the American Association of Motor Vehicle Administrators.