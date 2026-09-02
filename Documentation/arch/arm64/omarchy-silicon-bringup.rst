.. SPDX-License-Identifier: GPL-2.0-only

============================================
Omarchy Silicon Apple Linux bring-up design
============================================

:Status: DESIGN ONLY. K-01 through K-06 are NOT IMPLEMENTED and none is DONE.
:Owner: ``linux-omarchy`` downstream kernel/DT design lane
:Program: ``omarchy-apple-platform/PROGRAM.md`` at ``58302d148f0e8b855578f9aa518ff1c5eb48c515``

This document is the third and final bounded design-correction round for K-01.
It is a design contract only. It does not change kernel source, device-tree
source, bindings, schema, configuration, build output, CI, qualification data,
or the program authority. It does not inspect, reproduce, describe, fetch, or
depend on the opaque human-produced bootloader boundary.

The current outcome is deliberately fail-closed. F-02 and F-03 are external
dependencies. The supplied F-02 comparison snapshot at
``c315c7e79928d0041deb582bed79a61074361b21`` is rejected, provisional, and
frozen after its three correction rounds. It is comparison input only, never
local authority. No current text below converts it into an accepted contract.
Implementation, validators, CI, signed artifacts, physical qualification,
support, merge eligibility, and release readiness remain absent.

Scope and non-negotiable boundaries
-----------------------------------

* The only edited artifact in this round is this RST document.
* ``m1n1-omarchy`` and every m1n1 path or artifact are opaque human-produced
  boundaries. This document makes no source, path, content, or qualification
  claim about them.
* The thirteen unrelated dirty kernel paths in the handoff are outside scope.
  They are not inputs to a build, test, census, or design decision.
* A recognized SoC, a compiled DTB, a desktop boot, a static count, a green
  design checker, or an opened pull request is not compatibility, support,
  qualification, release, or DONE evidence.
* No local record may copy, shadow, alias, reinterpret, or redefine an
  authoritative external record. A future ratified generated binding is
  consumable only through the typed seam below.

The downstream design has one result vocabulary: ``ALLOW`` only after every
typed gate succeeds, ``REJECT`` for a deterministic invalid input, and
``HOLD`` for unavailable or unresolved authority. There is no default board,
component, signer, relation, rollback target, mutation, profile, or evidence
set.

External import seam: ImportedPlatformBindings
----------------------------------------------

K-01 has exactly one external import seam. No K-01 consumer reads an F-02/F-03
JSON object, generated file, lock, policy, or context directly. The only
admissible value is the nominal type ``ImportedPlatformBindings`` returned by
the future ratified external verifier.

The future artifact paths are fixed here and are not present in this design
branch:

.. list-table:: Future external artifact paths
   :header-rows: 1
   :widths: 28 42 30

   * - Artifact
     - Exact path or path pattern
     - Status now
   * - Schema input lock
     - ``schemas/schema-input.lock``
     - NOT_IMPLEMENTED; external input
   * - Generated output lock
     - ``bindings/generated-output.lock``
     - NOT_IMPLEMENTED; external input
   * - Python compiled lock
     - ``bindings/compiled-locks/python/compiled-binding-lock.json``
     - NOT_IMPLEMENTED; future ratified path
   * - Swift compiled lock
     - ``bindings/compiled-locks/swift/compiled-binding-lock.json``
     - NOT_IMPLEMENTED; future ratified path
   * - Rust boot compiled lock
     - ``bindings/compiled-locks/rust-boot/compiled-binding-lock.json``
     - NOT_IMPLEMENTED; future ratified path
   * - Generated Python binding
     - ``bindings/generated/python/1.0.0/omarchy_platform.py``
     - NOT_IMPLEMENTED; external generated output
   * - Generated Swift binding
     - ``bindings/generated/swift/1.0.0/OmarchyPlatform.swift``
     - NOT_IMPLEMENTED; external generated output
   * - Generated Rust boot binding
     - ``bindings/generated/rust-boot/1.0.0/omarchy_boot_health.rs``
     - NOT_IMPLEMENTED; external generated output

The last six paths are future import requirements, not local copies. If a
future ratified contract changes one, the import is absent until a new typed
schema version binds the replacement path. K-01 does not repair that mismatch.

The exact nominal record is:

.. code-block:: text

   ImportedPlatformBindings = {
     import_schema: "k01-imported-platform-bindings/v1",
     source_document_id: DocumentId,
     source_payload_digest: Digest,
     source_content_digest: Digest,
     source_preimage_digest: Digest,
     schema_set: SchemaSetIdentity,
     generated_output_lock: GeneratedOutputLockIdentity,
     compiled_locks: ExactList<CompiledLockIdentity, language>,
     generated_bindings: ExactList<GeneratedBindingIdentity, language>,
     authority: AuthorityRoleBinding,
     expected_context: ExpectedContext,
     negotiation: VersionCapabilityNegotiation,
     freshness: FreshnessWindow,
     replay: ReplayReservation,
     import_binding_digest: Digest
   }

   SchemaSetIdentity = {
     schema_set_id: "platform-schema-set/v1",
     schema_input_lock_path: "schemas/schema-input.lock",
     schema_input_lock_document_id: DocumentId,
     schema_input_lock_payload_digest: Digest,
     schema_input_lock_content_digest: Digest,
     schema_input_lock_preimage_digest: Digest,
     schema_set_digest: Digest
   }

   GeneratedOutputLockIdentity = {
     output_lock_path: "bindings/generated-output.lock",
     output_lock_document_id: DocumentId,
     output_lock_payload_digest: Digest,
     output_lock_content_digest: Digest,
     output_lock_preimage_digest: Digest,
     schema_set_digest: Digest
   }

   CompiledLockIdentity = {
     language: "python" | "swift" | "rust-boot",
     path: RelativeBindingPath,
     document_id: DocumentId,
     payload_digest: Digest,
     content_digest: Digest,
     preimage_digest: Digest,
     lock_digest: Digest,
     schema_set_digest: Digest,
     generated_output_digest: Digest,
     binding_identity_digest: Digest,
     parser_identity_digest: Digest,
     toolchain_identity_digest: Digest
   }

   GeneratedBindingIdentity = {
     language: "python" | "swift" | "rust-boot",
     path: RelativeBindingPath,
     artifact_id: ArtifactId,
     document_id: DocumentId,
     payload_digest: Digest,
     content_digest: Digest,
     output_digest: Digest,
     binding_id: LowerAsciiToken,
     binding_version: Version,
     binding_source_digest: Digest,
     parser_id: LowerAsciiToken,
     parser_version: Version,
     parser_source_digest: Digest,
     api_id: LowerAsciiToken,
     api_version: ApiVersion,
     api_source_digest: Digest
   }

``document_id`` is a stable identifier for one immutable revision. It is not
a content digest. ``payload_digest`` is ``sha256(JCS(payload))``;
``content_digest`` is the digest of the exact artifact bytes; and
``preimage_digest`` is the digest of the domain-separated authenticated
preimage. A verifier never substitutes one for another or accepts a matching
digest with a different ID.

The schema and binding preimages are fixed as follows:

.. code-block:: text

   schema_set_digest = sha256(ASCII("omarchy-schema-set/v1") || 0x00 ||
                              JCS(SchemaInputLock))
   generated_output_digest = sha256(ASCII("omarchy-generated-output/v1") ||
                                    0x00 || LF_normalized_output_bytes)
   compiled_lock_digest = sha256(ASCII("omarchy-compiled-binding-lock/v1") ||
                                 0x00 || JCS(CompiledLock without lock_digest))
   import_binding_digest = sha256(ASCII("omarchy-k01-import/v1") || 0x00 ||
                                  JCS(ImportedPlatformBindings without import_binding_digest))

The dependency graph is strictly one-way:

.. code-block:: text

   schema bytes
     -> schemas/schema-input.lock fields
     -> schema_set_digest
     -> bindings/generated-output.lock
     -> generated output bytes
     -> compiled lock preimage
     -> ImportedPlatformBindings
     -> K-01 consumers

The input lock cannot contain an output-lock digest, generated output path,
compiled-lock digest, current date, absolute path, or discovered file. The
output lock cannot select a second schema set. A compiled lock cannot select a
different generator, parser, API, toolchain, output path, or output digest.
The graph is rejected if an edge points backwards, if a lock is copied into a
local path, or if the same schema set resolves to two binding identities.

The imported authority and context are closed records. F-03 owns the trust
root, membership, key custody, threshold, revocation, rotation, expiry grace,
compromise, and offline-recovery policy; K-01 may not infer any of them.

.. code-block:: text

   AuthorityRoleBinding = {
     binding_schema: "authority-role-binding/v1",
     authority_id: LowerAsciiToken,
     role: "board-admission" | "manifest-release" | "installer-planner" |
           "owner-authorization" | "ci-conformance" | "qualification-lab" |
           "boot-runtime" | "dtb-authority" | "evidence-reader",
     actor_id: ActorId,
     account_id: AccountId,
     key_ids: SortedList<KeyId, key_id>,
     allowed_methods: ExactList<AuthorizationMethod, method>,
     service_policy_id: PolicyId,
     service_policy_digest: Digest,
     issued_at: Timestamp,
     expires_at: Timestamp,
     binding_digest: Digest
   }

   ExpectedContext = {
     context_schema: "expected-context/v1",
     payload_type: PayloadType,
     payload_version: Version,
     domain: Domain,
     context: Context,
     project_id: ProjectId,
     repository_id: RepositoryId,
     slice_id: SliceId,
     operation: Operation,
     board_id: BoardId,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     schema_set_digest: Digest,
     target_account_id: AccountId,
     target_account_binding: Digest,
     target_identity_digests: ExactList<Digest, value>,
     policy_digest: Digest
   }

   FreshnessWindow = {
     issued_at: Timestamp,
     expires_at: Timestamp,
     verified_clock_id: LowerAsciiToken,
     verified_now: Timestamp,
     max_clock_skew_ms: uint64
   }

   ReplayReservation = {
     replay_domain: LowerAsciiToken,
     replay_id: UUID,
     nonce_digest: Digest,
     tuple_digest: Digest,
     state: "reserved" | "committed",
     reservation_digest: Digest
   }

Version negotiation is exact. The required and supported payload sets are the
same ordered eight-member set: ``board-registry/v1``,
``platform-manifest/v1``, ``installer-plan/v1``, ``qualification-record/v1``,
``boot-health/v1``, ``owner-approval/v1``, ``boot-success-mark/v1``, and
``dtb-mutation-envelope/v1``. Consumer API versions compare numerically as
``(major, minor, patch)``. Missing, extra, duplicate, reordered, lower,
malformed, or string-compared versions reject with
``BINDING_INTEGRITY_FAILURE`` at ``$.negotiation``. There is no API downgrade,
nearby binding, mixed lock, parser substitution, or capability default.

The import verifier has exactly one public operation:

.. code-block:: text

   import_platform_bindings(
       FutureRatifiedGeneratedBindings,
       ExactSchemaInputLock,
       ExactGeneratedOutputLock,
       ExactCompiledLocks,
       Trusted<AuthorityRoleBinding>,
       ExpectedContext,
       VerifiedClock,
       ReplayStore) -> ImportedPlatformBindings | ImportError

Every K-01 entry point accepts only this result or a narrower trusted type:

* board admission consumes ``ImportedPlatformBindings`` plus trusted board
  observations and qualification records;
* manifest/component admission consumes its imported component IDs, locks,
  artifacts, firmware, DTB, and Mesa relations;
* boot evaluation consumes the imported boot binding and the sealed records in
  the next section;
* DTB mutation consumes the imported DTB policy and sealed DTB inputs;
* qualification acceptance consumes the imported profile/evidence binding; and
* delivery gates consume the exact imported artifact and validator identities.

Absent, rejected, provisional, stale, expired, replayed, copied, locally
aliased, input/output-cyclic, path-mismatched, digest-mismatched, version-
mismatched, or wrong-role/scope authority produces exactly this pre-admission
result before build, DTB mutation, boot evaluation, or qualification:

.. code-block:: text

   HOLD_TUPLE = {
     code: TRUST_BOUNDARY_FAILURE,
     path: "$.imported_platform_bindings",
     phase: P6,
     decision: HOLD,
     evidence: durable import decision with source IDs, all compared digests,
               authority binding digest, clock ID, replay ID, and reason
   }

The tuple is one result, not a family of local aliases. A rejected or
provisional F-02 snapshot cannot satisfy it. If the frozen comparison catalog
itself assigns incompatible tuples, the verifier stops before import with the
single contradiction result below, rather than choosing either row:

.. code-block:: text

   F02_CONTRACT_CONFLICT_TUPLE = {
     code: F02_CONTRACT_CONFLICT,
     path: "$.f02_catalog",
     phase: P0,
     decision: HOLD,
     evidence: durable catalog-conflict record naming both rows and tuples
   }

No K-01 document defines a replacement for either result.

Sealed boot-context seam
------------------------

The sole boot-context constructor is ``verify_boot_context``. Its input and
output are nominal private types. No caller-created, partial, parsed, copied,
or context-substituted value can cross the seam.

.. code-block:: text

   Trusted<SourceEvidence> = {
     evidence_schema: "source-evidence/v1",
     source_kind: LowerAsciiToken,
     source_id: LowerAsciiToken,
     adapter_id: LowerAsciiToken,
     adapter_api_version: Version,
     source_generation: uint64,
     evidence_digest: Digest,
     captured_at: Timestamp,
     expires_at: Timestamp,
     nonce: Nonce,
     source_record_digest: Digest
   }

   Trusted<AtomicBootRecord> = {
     record_schema: "atomic-boot-record/v1",
     record_id: UUID,
     project_id: ProjectId,
     repository_id: RepositoryId,
     slice_id: SliceId,
     board_id: BoardId,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     lineage_id: UUID,
     slot_id: SlotId,
     slot_generation: uint64,
     attempt_counter: uint64,
     source_generation: uint64,
     commit_state: "committed",
     bytes_digest: Digest,
     source: Trusted<SourceEvidence>,
     replay_id: UUID,
     record_digest: Digest
   }

   Trusted<VerifiedDtbInputs> = {
     inputs_schema: "verified-dtb-inputs/v1",
     project_id: ProjectId,
     repository_id: RepositoryId,
     slice_id: SliceId,
     board_id: BoardId,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     policy_id: PolicyId,
     policy_digest: Digest,
     tool_id: LowerAsciiToken,
     tool_version: Version,
     tool_digest: Digest,
     dtb_artifact_id: ArtifactId,
     dtb_artifact_version: Version,
     dtb_artifact_digest: Digest,
     firmware_bundle_id: LowerAsciiToken,
     firmware_bundle_version: Version,
     firmware_bundle_digest: Digest,
     dt_schema_id: LowerAsciiToken,
     dt_schema_version: Version,
     dt_schema_digest: Digest,
     source_generation: uint64,
     source_dtb_bytes_digest: Digest,
     post_dtb_bytes_digest: Digest,
     source: Trusted<SourceEvidence>,
     replay_id: UUID,
     recomputation_digest: Digest
   }

   Trusted<BootContext> = {
     context_schema: "boot-context/v1",
     board_id: BoardId,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     lineage_id: UUID,
     slot_id: SlotId,
     slot_generation: uint64,
     attempt_counter: uint64,
     source_generation: uint64,
     atomic_record_digest: Digest,
     lineage_source_digest: Digest,
     provenance: {
       source_kind: "atomic-boot-journal/v1",
       source_api_version: Version,
       storage_generation: uint64
     },
     context_digest: Digest
   }

   Trusted<BootHealthCore> = {
     schema: "boot-health/v1",
     schema_set_digest: Digest,
     document_id: DocumentId,
     issuer: AuthorityId,
     issued_at: Timestamp,
     expires_at: Timestamp,
     board_id: BoardId,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     profile_id: ProfileId,
     profile_digest: Digest,
     lineage_id: UUID,
     source_generation: uint64,
     slot: {slot_id: SlotId, slot_generation: uint64,
            boot_artifact_digest: Digest},
     attempt: {counter: uint64, started_at: Timestamp,
               finished_at: Timestamp | null, previous_slot: SlotId | null,
               boot_context_generation: uint64},
     checks: ExactList<BootCheck, check_id>,
     checks_digest: Digest,
     success: boolean,
     fallback: {decision: "hold" | "recover", target_slot: SlotId,
                rollback_set_digest: Digest, failure_code: FailureCode | null}
   }
   Trusted<BootSuccessMark> = {
     schema: "boot-success-mark/v1",
     schema_set_digest: Digest,
     document_id: DocumentId,
     issuer: AuthorityId,
     issued_at: Timestamp,
     expires_at: Timestamp,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     profile_id: ProfileId,
     profile_digest: Digest,
     core_digest: Digest,
     board_id: BoardId,
     lineage_id: UUID,
     source_generation: uint64,
     slot_id: SlotId,
     slot_generation: uint64,
     attempt_counter: uint64,
     marker_generation: uint64,
     marked_at: Timestamp,
     checks_digest: Digest,
     rollback_set_digest: Digest,
     marker_replay_id: UUID
   }

``verify_boot_context(Trusted<AtomicBootRecord>, Trusted<TrustContext>,
VerifiedClock) -> Trusted<BootContext>`` is the only constructor. It verifies
the complete committed record, source bytes and ``bytes_digest``, board,
manifest, slot, lineage, counter, source generation, role, expiry, clock, and
replay reservation before returning the sealed value. The public API exposes no
initializer, cast, deserializer, map conversion, or nullable partial record.

``source_generation`` and ``attempt_counter`` are unsigned 64-bit integers
from 0 through 18,446,744,073,709,551,615. Generation zero means no committed
source snapshot; the first committed snapshot is one. Counter zero means an
unstarted attempt; the first executed attempt is one. A changed committed
snapshot increments generation once. A changed slot or manifest allocates a
new lineage ID and tombstones the previous tuple. Reset, reuse, decrease,
wrap, overflow, torn record, duplicate replay, or missing persistence is a
hold with ``BOOT_COUNTER_FAILURE`` or ``TRUST_BOUNDARY_FAILURE`` and never a
success.

The preimages are separate and acyclic:

.. code-block:: text

   D_core = sha256(ASCII("omarchy-boot-health-core/v1") || 0x00 || JCS(C))
   D_mark = sha256(ASCII("omarchy-boot-success-mark/v1") || 0x00 || JCS(M))
   D_checks = sha256(ASCII("omarchy-boot-check-set/v1") || 0x00 || JCS(checks))
   D_rollback = sha256(ASCII("omarchy-boot-rollback-set/v1") || 0x00 ||
                       JCS({manifest_ids, artifact_ids, slot_ids,
                            retention_count, failure_attempt_limit}))
   D_atomic = sha256(ASCII("omarchy-boot-atomic-record/v1") || 0x00 ||
                      JCS(AtomicBootRecord without record_digest))
   D_context = sha256(ASCII("omarchy-boot-context/v1") || 0x00 ||
                      JCS(BootContext without context_digest))

The core, marker, checks, rollback, atomic-record, and context digests are
never inserted into their own preimages. A marker authenticates ``D_core``;
the core never authenticates ``D_mark``. ``evaluate_boot_health`` has exactly
this signature and consumes the sealed context:

.. code-block:: text

   evaluate_boot_health(
       Trusted<BootHealthCore>,
       Trusted<BootSuccessMark> | None,
       Trusted<PlatformManifest>,
       Trusted<BootContext>,
       VerifiedClock) -> SlotDecision

The evaluator checks source generation, attempt counter, slot and slot
generation, lineage, last-known-good record, rollback set and retention,
profile and profile digest, core digest, checks digest, marker generation,
marker clock, marker replay, signature context, and every required check before
success. ``Trusted<BootHealthCore>`` and ``Trusted<BootSuccessMark>`` are
consumed exactly; a raw health object, raw marker, raw ``BootContext``, or
caller-supplied context is not accepted.

Typed component relations and rollback
---------------------------------------

The five component IDs are closed and are the only relation endpoints:
``linux-kernel``, ``dtb-set``, ``firmware-bundle``, ``mesa-stack``, and
``boot-stack``. An endpoint is never an ABI object, nested schema object,
artifact array, document ID, or caller-provided structure.

Every component record is closed and binds ``component_id`` to its source
commit and source digest, provenance and recipe digest, normalized config
inputs, ordered patch lock, toolchain lock, report lock, DTB source/schema
identity, firmware schema identity, boot profile identity, package/artifact
records, rollback coordinates, and relation records. Source, configuration,
toolchain, artifact, firmware schema, DTB, boot ABI, and Mesa compatibility
therefore resolve through the five component IDs and the exact manifest scope.
A nested source object, artifact array, ABI object, filename, or document ID
cannot be used as a relation endpoint or as an implicit compatibility claim.

.. code-block:: text

   TypedCompatibilityRelation = {
     relation_schema: "typed-compatibility-relation/v1",
     left_component_id: ComponentId,
     relation: "boot-protocol/v1" | "kernel-abi/v1" |
               "firmware-schema/v1" | "package-architecture/v1" |
               "component-interface/v1",
     right_component_id: ComponentId,
     contract_id: LowerAsciiToken,
     evidence_digest: Digest,
     evidence_preimage_digest: Digest,
     scope: RelationScope,
     issued_at: Timestamp,
     expires_at: Timestamp
   }

   RelationScope = {
     project_id: ProjectId,
     repository_id: RepositoryId,
     slice_id: SliceId,
     board_id: BoardId,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     schema_set_digest: Digest
   }

The relation table is the complete K-01 projection. Each row has component
IDs, one relation kind, a contract ID, evidence digest and preimage, scope,
freshness, and deterministic rejection:

.. list-table:: Component relation closure
   :header-rows: 1
   :widths: 24 20 20 36

   * - Relation ID
     - Left endpoint
     - Right endpoint and kind
     - Meaning and rejection
   * - ``K01-REL-001``
     - ``linux-kernel``
     - ``dtb-set`` / ``kernel-abi/v1``
     - Kernel ABI and DTB source/config lock agree; substitution is ``CROSS_DOCUMENT_MISMATCH``.
   * - ``K01-REL-002``
     - ``linux-kernel``
     - ``firmware-bundle`` / ``firmware-schema/v1``
     - Firmware ABI is bound to the kernel component ID; nested-object endpoints reject.
   * - ``K01-REL-003``
     - ``firmware-bundle``
     - ``dtb-set`` / ``firmware-schema/v1``
     - DTB firmware schema and bundle digest agree; cross-board values reject.
   * - ``K01-REL-004``
     - ``linux-kernel``
     - ``boot-stack`` / ``boot-protocol/v1``
     - Boot artifact protocol is bound to kernel ABI; document-only binding rejects.
   * - ``K01-REL-005``
     - ``boot-stack``
     - ``firmware-bundle`` / ``component-interface/v1``
     - Opaque boot artifact and firmware bundle use one manifest scope.
   * - ``K01-REL-006``
     - ``linux-kernel``
     - ``mesa-stack`` / ``kernel-abi/v1``
     - DRM ABI and Mesa artifact use one kernel component ID.
   * - ``K01-REL-007``
     - ``mesa-stack``
     - ``firmware-bundle`` / ``component-interface/v1``
     - GPU firmware ABI and Mesa artifact use one firmware component ID.

Component relation lists are owned by exactly one component, in the declared
lexicographic owner order. The top-level manifest relation projection is the
exact sorted union, with no winner, merge, or last-writer behavior. Missing,
extra, duplicate, reordered, orphaned, wrong-scope, stale, or mismatched rows
reject at the first differing JSON path.

Rollback is closed at every component and at the manifest projection:

.. code-block:: text

   RollbackCoordinates = {
     coordinate_schema: "component-rollback/v1",
     previous_component_ids: SortedList<ComponentId, component_id>,
     previous_manifest_ids: SortedList<DocumentId, document_id>,
     artifact_ids: SortedList<ArtifactId, artifact_id>,
     retention_count: uint16,
     rollback_policy_id: PolicyId,
     rollback_policy_digest: Digest,
     last_known_good: LastKnownGoodReference,
     ordering: ExactList<RollbackStep, sequence>
   }

   LastKnownGoodReference = {
     manifest_id: DocumentId,
     manifest_digest: Digest,
     component_ids: ExactList<ComponentId, component_id>,
     artifact_ids: ExactList<ArtifactId, artifact_id>,
     slot_id: SlotId,
     slot_generation: uint64,
     record_digest: Digest
   }

   RollbackStep = {
     sequence: uint16,
     component_id: ComponentId,
     artifact_id: ArtifactId,
     action: "select-lkg/v1" | "restore/v1" | "verify/v1",
     evidence_digest: Digest
   }

``previous_component_ids`` is an exact sorted set of the five typed IDs or a
closed empty set for a first release. ``retention_count`` is inclusive of the
current manifest and must satisfy the signed rollback policy. The LKG record,
manifest IDs, component IDs, artifact IDs, slot, sequence, retention, policy
ID, and policy digest must all agree. A downgrade outside this policy, missing
LKG, duplicate step, orphan component/artifact, omitted previous component,
extra rollback target, cross-board or cross-manifest reference, and reordered
step yields ``BOOT_FALLBACK_FAILURE`` at the first rollback path and selects
HOLD or recovery. No consumer infers LKG from a filename or current slot.

Exact failure mapping and contradiction hold
---------------------------------------------

The imported error is a closed redacted tuple. Every hostile or boundary
condition has exactly one code, JSON path, phase, decision, and durable
evidence record. The following is the K-01 consumption table; it is a mapping
table, not a new local authority or a second F-02 catalog.

.. list-table:: K-01 failure tuple closure
   :header-rows: 1
   :widths: 27 36 10 12 15

   * - Condition
     - Imported failure code and exact JSON path
     - Phase
     - Result
     - Durable evidence
   * - Missing, rejected, provisional, stale, copied, aliased, cyclic, or wrong authority import
     - ``TRUST_BOUNDARY_FAILURE`` at ``$.imported_platform_bindings``
     - P6
     - HOLD
     - Import decision
   * - Frozen F-02 catalog contradiction
     - ``F02_CONTRACT_CONFLICT`` at ``$.f02_catalog``
     - P0
     - HOLD
     - Both row IDs and tuples
   * - Invalid UTF-8, BOM, control byte, truncated input
     - ``PARSE_SCHEMA_FAILURE`` at ``$.transport``
     - P0
     - REJECT
     - Parser receipt
   * - Duplicate JSON name or semantic key
     - ``DUPLICATE_SEMANTIC_KEY`` at the second name/key
     - P0/P1
     - REJECT
     - Parser receipt
   * - Unknown property, extra component alias, unknown enum
     - ``UNKNOWN_FIELD`` at the first unknown property
     - P1
     - REJECT
     - Schema receipt
   * - Missing required field, malformed ID, wrong collection order
     - ``PARSE_SCHEMA_FAILURE`` at the missing/invalid field
     - P1
     - REJECT
     - Schema receipt
   * - Over-bound bytes, depth, count, integer, or 4,097-byte marker
     - ``RESOURCE_LIMIT`` at the first over-bound value
     - P0/P1
     - REJECT
     - Limit receipt
   * - Non-canonical number, JCS mismatch, or digest preimage mismatch
     - ``CANONICALIZATION_FAILURE`` at ``$`` or first scalar
     - P2
     - REJECT
     - Canonical receipt
   * - Wrong envelope domain, context, payload type, role, or signature context
     - ``SIGNATURE_CONTEXT_MISMATCH`` at the first mismatched envelope/signature path
     - P3
     - REJECT
     - Signature receipt
   * - Unknown key, revoked key, invalid signature, or authority binding mismatch
     - ``TRUST_FAILURE`` at ``$.signatures[0]`` or authority path
     - P3
     - REJECT
     - Trust receipt
   * - Expired or replayed signed object
     - ``EXPIRY_OR_REPLAY_FAILURE`` at ``$.payload.expires_at`` or ``$.replay_id``
     - P3
     - REJECT
     - Replay receipt
   * - Source bytes, source generation, or source tuple mismatch
     - ``DTB_INPUT_VERIFICATION_FAILURE`` at ``$.source_identity``
     - P4
     - REJECT
     - Source receipt
   * - Missing, forged, stale, partial, or replayed local trusted record
     - ``TRUST_BOUNDARY_FAILURE`` at the first local record field
     - P6
     - HOLD
     - Trust-seam receipt
   * - Wrong board, manifest, schema set, artifact, firmware, or scope
     - ``CROSS_DOCUMENT_MISMATCH`` at the first unequal bound path
     - P5
     - REJECT
     - Binding receipt
   * - Stable-ID omission, malformed capacity, or invalid source identity
     - ``IDENTITY_INCOMPLETE`` at the identity field
     - P4
     - REJECT
     - Identity receipt
   * - Stable-ID endpoint substitution or conflicting parent/tuple
     - ``AMBIGUOUS_IDENTITY`` at the second identity path
     - P4
     - REJECT
     - Identity receipt
   * - Component relation endpoint, kind, contract, evidence, scope, or freshness mismatch
     - ``CROSS_DOCUMENT_MISMATCH`` at ``$.payload.compatibility[i]``
     - P5
     - REJECT
     - Relation receipt
   * - Rollback omission, duplicate, orphan, downgrade, LKG, retention, or policy mismatch
     - ``BOOT_FALLBACK_FAILURE`` at ``$.payload.rollback``
     - P5/P6
     - HOLD
     - Rollback receipt
   * - Plan, scope, owner proof, target account, or operation mismatch
     - ``PLAN_SCOPE_OR_APPROVAL_FAILURE`` at the first scope path
     - P6
     - REJECT
     - Scope receipt
   * - Boot source generation or attempt reset/wrap/decrease
     - ``BOOT_COUNTER_FAILURE`` at ``$.atomic_record.attempt_counter``
     - P4/P6
     - HOLD
     - Counter receipt
   * - Boot context substitution or core/marker lineage mismatch
     - ``BOOT_CONTEXT_MISMATCH`` at ``$.boot_context`` or first differing field
     - P5
     - HOLD
     - Context receipt
   * - Missing, forged, expired, replayed, or wrong-role success marker
     - ``BOOT_MARKER_AUTH_FAILURE`` at ``$.signatures[0]`` or marker replay path
     - P3/P6
     - HOLD
     - Marker receipt
   * - Missing, extra, duplicate, failed, or out-of-profile boot check
     - ``BOOT_REQUIRED_CHECK_FAILURE`` at ``$.payload.checks[i]``
     - P5
     - HOLD
     - Check receipt
   * - Unknown, missing, unauthorized, out-of-order, or wrong-operation DTB rule
     - ``UNKNOWN_MUTATION`` at ``$.payload.authorized_mutations[i]``
     - P5
     - REJECT
     - Mutation receipt
   * - DTB before/after value or preimage mismatch
     - ``DTB_INPUT_VERIFICATION_FAILURE`` at the first before/after path
     - P4/P5
     - REJECT
     - DTB recomputation receipt
   * - DTB policy, tool, schema, artifact, firmware, nonce, source, or post digest mismatch
     - ``CROSS_DOCUMENT_MISMATCH`` at the first differing identity path
     - P5
     - REJECT
     - DTB binding receipt
   * - DTB expiry or replay reservation conflict
     - ``DTB_EXPIRY_FAILURE`` at ``$.payload.expires_at`` or ``EXPIRY_OR_REPLAY_FAILURE`` at ``$.replay_identity``
     - P3/P6
     - REJECT
     - Reservation receipt
   * - Generated binding, compiled lock, parser, API, or artifact output drift
     - ``BINDING_INTEGRITY_FAILURE`` at the first lock/output path
     - P1/P5/P6
     - REJECT
     - Binding receipt
   * - Qualification profile, evidence, unit, board, manifest, or attempt mismatch
     - ``CROSS_DOCUMENT_MISMATCH`` at the first evidence binding path
     - P5
     - REJECT
     - Qualification receipt
   * - Qualification evidence absent, stale, pooled, reused, omitted failure, or missing artifact
     - ``TRUST_BOUNDARY_FAILURE`` at ``$.qualification.evidence``
     - P6
     - HOLD
     - Evidence receipt

The imported failure catalog must keep one row per code/path/phase/result.
``decision = HOLD`` is permitted only for the five boot codes
``BOOT_MARKER_AUTH_FAILURE``, ``BOOT_CONTEXT_MISMATCH``,
``BOOT_COUNTER_FAILURE``, ``BOOT_REQUIRED_CHECK_FAILURE``,
``BOOT_FALLBACK_FAILURE``, and the trust-boundary hold
``TRUST_BOUNDARY_FAILURE``. ``F02_CONTRACT_CONFLICT`` is a pre-admission
catalog control and is not selected for an ordinary input. All other failures
are REJECT. No consumer uses “first applicable” among competing tuples.

The frozen provisional catalog contains a contradiction that K-01 must not
resolve locally. The exact conflicting rows are:

.. list-table:: Frozen catalog contradictions
   :header-rows: 1
   :widths: 28 34 34

   * - Frozen row
     - Tuple in one catalog location
     - Conflicting tuple in another catalog location
   * - ``dtb-wrong-before-digest``
     - ``CROSS_DOCUMENT_MISMATCH`` at ``$.payload.authorized_mutations[0].before_value_digest``, P5
     - ``DTB_INPUT_VERIFICATION_FAILURE`` for the same wrong-before class, DTB input verification phase
   * - ``dtb-add-real-before``
     - ``DTB_INPUT_VERIFICATION_FAILURE`` for an add sentinel with a real before value, P5
     - Generic wrong-before mapping overlaps the preceding row without a disambiguating path rule
   * - expired DTB input
     - ``DTB_INPUT_BOUNDARY_FAILURE`` at ``$.dtb_inputs``, P6 in a generic K-01 mapping
     - ``DTB_EXPIRY_FAILURE`` at ``$.payload.expires_at``, P3 in the frozen catalog
   * - expired/replayed boot material
     - grouped as a boot input boundary failure in the prior K-01 prose
     - ``EXPIRY_OR_REPLAY_FAILURE`` at the expiry/replay path, P3 in the frozen catalog

Until an external future ratified catalog provides one tuple for each row,
these rows produce ``F02_CONTRACT_CONFLICT_TUPLE`` before any K-01 admission.

Exact DTB mutation policy
-------------------------

The DTB consumer accepts one signed ``dtb-mutation-envelope/v1`` and one
future imported policy. The policy is closed, ordered, and bound to board,
manifest, source generation, signer, tool, schema, firmware, artifact, and
post-digest identities. It does not accept a path prefix, wildcard, regular
expression, operation default, or locally widened set.

.. list-table:: K-01 DTB rule set
   :header-rows: 1
   :widths: 18 32 15 20

   * - Rule ID
     - Exact JSON/property path
     - Allowed operation
     - Before-value semantics
   * - ``K01-DTB-001``
     - ``/soc/gpu@203000000/compatible``
     - replace
     - real canonical value digest
   * - ``K01-DTB-002``
     - ``/soc/gpu@203000000/power-domains``
     - replace
     - real canonical value digest
   * - ``K01-DTB-003``
     - ``/soc/gpu@203000000/memory-region``
     - add
     - ``absent_digest`` only
   * - ``K01-DTB-004``
     - ``/soc/gpu@203000000/apple,firmware-abi``
     - replace
     - real canonical value digest
   * - ``K01-DTB-005``
     - ``/soc/gpu@203000000/firmware-name``
     - remove
     - real canonical value digest

The exact future policy record is:

.. code-block:: text

   DtbMutationPolicy = {
     policy_schema: "dtb-mutation-policy/v1",
     board_id: BoardId,
     manifest_id: DocumentId,
     manifest_digest: Digest,
     policy_id: PolicyId,
     policy_digest: Digest,
     tool_id: LowerAsciiToken,
     tool_version: Version,
     tool_digest: Digest,
     dtb_artifact_id: ArtifactId,
     dtb_artifact_digest: Digest,
     firmware_bundle_id: LowerAsciiToken,
     firmware_bundle_digest: Digest,
     dt_schema_id: LowerAsciiToken,
     dt_schema_digest: Digest,
     source_generation: uint64,
     signer_authority: AuthorityRoleBinding,
     rules: ExactList<DtbMutationRule, authorization_rule_id>,
     policy_record_digest: Digest
   }

   DtbMutationRule = {
     authorization_rule_id: LowerAsciiToken,
     property_path: DtbPropertyPath,
     operation: "add" | "replace" | "remove",
     allowed: true
   }

   AuthorizedMutation = {
     sequence: uint16,
     mutation_id: LowerAsciiToken,
     property_path: DtbPropertyPath,
     operation: "add" | "replace" | "remove",
     before_value_digest: Digest,
     after_value_digest: Digest,
     authorization_rule_id: LowerAsciiToken,
     before_preimage_digest: Digest,
     after_preimage_digest: Digest
   }

   absent_digest = sha256(ASCII("omarchy-dtb-absent/v1") || 0x00)
   value_digest(v) = sha256(ASCII("omarchy-dtb-value/v1") || 0x00 || JCS(v))
   before_preimage(add) = absent_digest
   after_preimage(remove) = absent_digest
   before_preimage(replace) = value_digest(current_value)
   after_preimage(add | replace) = value_digest(new_value)
   D_dtb_pre = sha256(ASCII("omarchy-dtb-pre/v1") || 0x00 || source_dtb_bytes)
   D_dtb_post = sha256(ASCII("omarchy-dtb-post/v1") || 0x00 || post_dtb_bytes)
   nonce_digest = sha256(ASCII("omarchy-dtb-nonce/v1") || 0x00 || nonce)

The exact envelope paths are ``$.payload.board_identity``,
``$.payload.source_identity``, ``$.payload.platform_manifest_document_id``,
``$.payload.platform_manifest_payload_digest``,
``$.payload.pre_mutation_dtb_digest``,
``$.payload.post_mutation_dtb_digest``, ``$.payload.policy_identity``,
``$.payload.tool_identity``, ``$.payload.artifact_identity``,
``$.payload.firmware_bundle_identity``, ``$.payload.dt_schema_identity``,
``$.payload.authorized_mutations``, ``$.payload.signer_authority``,
``$.payload.nonce``, and ``$.payload.replay_identity``. The signer role is
``dtb-authority`` and the scope is the exact imported project, repository,
slice, board, manifest, policy, tool, DTB artifact, firmware, DT schema, and
source generation.

The consumer checks expiry and the durable replay cache's atomic reservation, then identity and
scope, then rule ID/path/operation/order, then before and after values and
preimages, then source bytes, every intermediate mutation, and post bytes.
Unknown path, operation, order, rule, before value, after value, nonce domain,
stale clock, replay, transplant, cross-firmware input, or locally widened
policy has one exact tuple in the failure table and produces no DTB mutation.
The post digest is committed only with the durable replay reservation. A
crash re-reads the reservation and postcondition; it never guesses or retries
twice.

Authenticated qualification binding
------------------------------------

Qualification is a future signed input, not a numeric claim in this document.
The exact future paths are:

.. list-table:: Future qualification artifacts
   :header-rows: 1
   :widths: 30 43 27

   * - Input
     - Exact path
     - Status now
   * - Signed profile
     - ``qualification/profiles/<profile_id>.json``
     - NOT_IMPLEMENTED
   * - Signed record
     - ``qualification/records/<record_id>.json``
     - NOT_IMPLEMENTED
   * - Unit run history
     - ``qualification/runs/<record_id>/<unit_id>/<run_id>.json``
     - NOT_IMPLEMENTED
   * - Retained evidence
     - ``qualification/evidence/<evidence_id>``
     - NOT_IMPLEMENTED
   * - Evidence manifest
     - ``qualification/evidence/<record_id>/manifest.json``
     - NOT_IMPLEMENTED

The profile and record are independently signed and bound to the imported
schema-set, generated/compiled binding, board registry, platform manifest,
component artifact IDs/digests, firmware baseline, DT schema, source
generation, policy digest, and qualification-lab ``AuthorityRoleBinding``.
Each record carries document ID, payload digest, content digest, preimage
digest, profile ID/digest, board ID, manifest ID/digest, exact component and
artifact set, clock window, attempt inclusion, unit ID, run history ID, and
evidence IDs. A stable ID is not a content digest.

For every materially distinct board/profile tuple the future lab must serialize
at least two independently identified physical units. Each unit independently
requires:

* 3 clean installs with preserved install journals and failure attempts;
* 50 cold boots and a separate 50 warm boots, each with unique run histories,
  unique attempt counters, source generation, slot, lineage, and artifact
  binding;
* 5 cycles for every applicable port and peripheral path;
* 10 update/rollback cycles, including failure-triggered rollback;
* every applicable capability in the signed profile, with no omitted failure;
* retained raw evidence by content digest and a signed evidence manifest; and
* all profile thresholds: 100% clean-install success, 100% applicable
  port/peripheral success, 100% update/rollback success, zero critical
  kernel faults, zero unsafe speaker/thermal state, zero silently disabled
  applicable capability, and zero untracked qualification exception.

An evidence row is accepted only when its profile, schema and binding locks,
board, manifest, artifact, firmware, clock, source generation, unit, run,
attempt, evidence digest, and failure-preservation fields compare exactly.
Reused or pooled units, reused run histories, stale clocks, stale profiles,
cross-board or cross-manifest records, missing artifacts, omitted failures,
synthetic evidence, and count-only summaries reject or hold at the exact
qualification tuple. Three clean installs or 100 boot counts on one pooled
record cannot satisfy two independent units.

Executable-delivery acceptance design
--------------------------------------

The following is the exact future delivery acceptance shape. Every absent item
is explicitly NOT_IMPLEMENTED or TOOLING_BLOCK; this design document is not a
runtime PASS.

.. list-table:: Future delivery gates
   :header-rows: 1
   :widths: 24 46 30

   * - Gate
     - Required future artifact or command
     - Current state
   * - Schema manifest
     - ``schemas/manifest.json`` naming all eleven Draft 2020-12 inputs, IDs, paths, and digests
     - NOT_IMPLEMENTED
   * - Fixture manifest
     - ``fixtures/manifest.json`` naming accepted and hostile fixture IDs, one mutation, tuple, and order
     - NOT_IMPLEMENTED
   * - Generated outputs
     - ``bindings/generated-output.lock`` plus the three exact language outputs and digests
     - NOT_IMPLEMENTED
   * - Validators
     - ``tools/schema/omarchy-platform-validate`` with strict JSON, JCS, signature, lock, relation, boot, and DTB checks
     - NOT_IMPLEMENTED
   * - Fixture runner
     - ``tools/schema/omarchy-platform-fixtures`` emitting the closed hostile report schema
     - NOT_IMPLEMENTED
   * - Clean-checkout CI
     - ``.github/workflows/schema-conformance.yml`` and ``.github/workflows/linux-k01.yml`` with no dirty paths
     - NOT_IMPLEMENTED
   * - DT validation
     - ``make dtbs_check`` with pinned ``dtc``/``dt-schema`` container and report digest
     - TOOLING_BLOCK until tools and clean checkout exist
   * - Kernel build
     - pinned ``make ARCH=arm64`` argv, config digest, compiler/toolchain lock, and reproducibility comparison
     - NOT_IMPLEMENTED
   * - Signed artifacts
     - immutable kernel, DTB, firmware, Mesa, boot, package, and manifest signatures with content digests
     - NOT_IMPLEMENTED
   * - Physical evidence
     - signed profile, two-unit records, run histories, evidence manifests, and retained failure artifacts
     - NOT_IMPLEMENTED

The future validator command/argv is fixed for the handoff:

.. code-block:: text

   omarchy-platform validate --type TYPE --input FILE --schema-lock schemas/schema-input.lock
   omarchy-platform bindings check --input-lock schemas/schema-input.lock \
       --output-lock bindings/generated-output.lock \
       --compiled-lock bindings/compiled-locks/LANGUAGE/compiled-binding-lock.json
   omarchy-platform fixtures run --manifest fixtures/manifest.json --report REPORT.json
   make ARCH=arm64 LLVM=1 dtbs_check DT_SCHEMA_FILES=schemas/omarchy
   make ARCH=arm64 olddefconfig
   make ARCH=arm64 Image.gz dtbs modules
   omarchy-platform artifacts verify --manifest SIGNED_MANIFEST --at VERIFIED_TIME
   omarchy-platform qualification verify --profile PROFILE --record RECORD \
       --evidence-manifest EVIDENCE_MANIFEST --at VERIFIED_TIME

The pinned future runtime identities are CPython 3.12.8 with the locked
strict-json/JCS implementation, Swift 6.0.3 with the locked Swift API, and
rustc 1.84.1 stable for ``aarch64-unknown-none`` with the locked no-std boot
API. The container image, digest, ``dtc``, dt-schema, compiler, generator,
parser, API, locale, line-ending policy, argv, and environment allowlist are
lock fields. Network fetch, mutable tags, absolute paths, host discovery,
locale-dependent output, and dirty-checkout execution fail closed.

The hostile report schema is:

.. code-block:: text

   HostileFixtureReport = {
     report_schema: "k01-hostile-fixture-report/v1",
     fixture_id: LowerAsciiToken,
     order: uint16,
     executor: LowerAsciiToken,
     consumer: LowerAsciiToken,
     implementation_status: "NOT_IMPLEMENTED" | "TOOLING_BLOCK" | "PASS" | "FAIL",
     mutation_digest: Digest,
     expected_code: FailureCode,
     expected_path: JsonPath,
     expected_phase: Phase,
     expected_decision: "ALLOW" | "REJECT" | "HOLD",
     observed_signal: "DESIGN_MODEL" | "RUNTIME",
     evidence_digest: Digest
   }

No design/source scan, static source count, generated-looking file, VM, or
desktop boot can populate a runtime ``PASS`` report.

Structural source/design census
-------------------------------

The verified bounded source/design facts that this note preserves are:

.. list-table:: Structural census
   :header-rows: 1
   :widths: 35 15 20 30

   * - Census
     - Total
     - Accepted/quarantined split
     - Meaning
   * - Apple AGX compatible literals
     - 18
     - 8 accepted / 10 unsupported
     - Source/document set equality only; unsupported values remain quarantined.
   * - AGX GPU properties
     - 61
     - 8 accepted / 53 unsupported
     - Source/document set equality only; no DT schema or hardware PASS.
   * - Kconfig symbols
     - 8
     - 13 depends clauses
     - Four direct edges, four operational edges, zero-cycle model.
   * - Qualification minima
     - 2 units/profile
     - 3 installs, 50 cold, 50 warm, 5 port, 10 update/rollback
     - Prose minimum only; evidence remains NOT_IMPLEMENTED.

The AGX accepted literals are exactly 8 of the 18 source literals, and the
remaining 10 are unsupported/quarantined: ``apple,agx-g13x``,
``apple,agx-g14x``, ``apple,agx-t6000``, ``apple,agx-t6001``,
``apple,agx-t6002``, ``apple,agx-t6020``, ``apple,agx-t6021``,
``apple,agx-t6022``, ``apple,agx-t8103``, and ``apple,agx-t8112``. The 61/8/53
property census is the verified source/design split, not an executable
binding result. The Kconfig graph is a model only: no current Kconfig guard
or ordered bring-up checker exists.

Hostile closure and deterministic design models
-----------------------------------------------

Every row below has an executor, consumer, implementation status, expected
tuple, evidence, and unique order/ID. The future executor must plant exactly
one mutation in a throwaway input and emit one report row. In this round all
rows are ``NOT_IMPLEMENTED`` and the observed signal is ``DESIGN_MODEL``.

.. list-table:: K-01 hostile rows
   :header-rows: 1
   :widths: 8 22 18 25 27

   * - Order
     - Fixture ID and executor
     - Consumer
     - Expected tuple
     - Evidence
   * - 001
     - ``H01-import-absent`` / ``schema-fixture-runner``
     - import seam
     - ``TRUST_BOUNDARY_FAILURE``, ``$.imported_platform_bindings``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 002
     - ``H02-import-provisional`` / ``schema-fixture-runner``
     - import seam
     - ``TRUST_BOUNDARY_FAILURE``, ``$.imported_platform_bindings``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 003
     - ``H03-lock-drift`` / ``schema-fixture-runner``
     - lock verifier
     - ``BINDING_INTEGRITY_FAILURE``, ``$.schema_set_digest``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 004
     - ``H04-lock-cycle`` / ``schema-fixture-runner``
     - lock verifier
     - ``BINDING_INTEGRITY_FAILURE``, ``$.schema_input_lock``, P1, REJECT
     - DESIGN_MODEL receipt
   * - 005
     - ``H05-local-alias`` / ``schema-fixture-runner``
     - import seam
     - ``TRUST_BOUNDARY_FAILURE``, ``$.imported_platform_bindings``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 006
     - ``H06-wrong-role`` / ``signature-fixture-runner``
     - trust verifier
     - ``SIGNATURE_CONTEXT_MISMATCH``, ``$.signatures[0].signer_role``, P3, REJECT
     - DESIGN_MODEL receipt
   * - 007
     - ``H07-wrong-scope`` / ``signature-fixture-runner``
     - ExpectedContext
     - ``SIGNATURE_CONTEXT_MISMATCH``, ``$.context``, P3, REJECT
     - DESIGN_MODEL receipt
   * - 008
     - ``H08-relation-endpoint`` / ``manifest-fixture-runner``
     - relation verifier
     - ``CROSS_DOCUMENT_MISMATCH``, ``$.payload.compatibility[0].left_component_id``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 009
     - ``H09-relation-nested-object`` / ``manifest-fixture-runner``
     - relation verifier
     - ``PARSE_SCHEMA_FAILURE``, ``$.payload.compatibility[0].left_component_id``, P1, REJECT
     - DESIGN_MODEL receipt
   * - 010
     - ``H10-rollback-omission`` / ``manifest-fixture-runner``
     - rollback verifier
     - ``BOOT_FALLBACK_FAILURE``, ``$.payload.rollback.previous_component_ids``, P5, HOLD
     - DESIGN_MODEL receipt
   * - 011
     - ``H11-rollback-policy`` / ``manifest-fixture-runner``
     - rollback verifier
     - ``BOOT_FALLBACK_FAILURE``, ``$.payload.rollback.rollback_policy_digest``, P5, HOLD
     - DESIGN_MODEL receipt
   * - 012
     - ``H12-boot-context-substitution`` / ``boot-fixture-runner``
     - boot evaluator
     - ``TRUST_BOUNDARY_FAILURE``, ``$.boot_context``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 013
     - ``H13-boot-lineage`` / ``boot-fixture-runner``
     - boot evaluator
     - ``BOOT_CONTEXT_MISMATCH``, ``$.payload.lineage_id``, P5, HOLD
     - DESIGN_MODEL receipt
   * - 014
     - ``H14-counter-reset`` / ``boot-fixture-runner``
     - boot constructor
     - ``BOOT_COUNTER_FAILURE``, ``$.atomic_record.attempt_counter``, P4, HOLD
     - DESIGN_MODEL receipt
   * - 015
     - ``H15-marker-cycle`` / ``boot-fixture-runner``
     - boot parser
     - ``BOOT_MARKER_DIGEST_CYCLE``, ``$.payload.canonical_payload_digest``, P2, REJECT
     - DESIGN_MODEL receipt
   * - 016
     - ``H16-dtb-unknown-path`` / ``dtb-fixture-runner``
     - DTB consumer
     - ``UNKNOWN_MUTATION``, ``$.payload.authorized_mutations[0].property_path``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 017
     - ``H17-dtb-operation`` / ``dtb-fixture-runner``
     - DTB consumer
     - ``UNKNOWN_MUTATION``, ``$.payload.authorized_mutations[0].operation``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 018
     - ``H18-dtb-before`` / ``dtb-fixture-runner``
     - DTB consumer
     - ``DTB_INPUT_VERIFICATION_FAILURE``, ``$.payload.authorized_mutations[0].before_value_digest``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 019
     - ``H19-dtb-policy-widen`` / ``dtb-fixture-runner``
     - policy verifier
     - ``TRUST_BOUNDARY_FAILURE``, ``$.dtb_policy.rules``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 020
     - ``H20-dtb-replay`` / ``dtb-fixture-runner``
     - replay store
     - ``EXPIRY_OR_REPLAY_FAILURE``, ``$.payload.replay_identity``, P3, REJECT
     - DESIGN_MODEL receipt
   * - 021
     - ``H21-dtb-cross-firmware`` / ``dtb-fixture-runner``
     - DTB binding
     - ``CROSS_DOCUMENT_MISMATCH``, ``$.payload.firmware_bundle_identity``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 022
     - ``H22-reused-unit`` / ``qualification-fixture-runner``
     - qualification verifier
     - ``TRUST_BOUNDARY_FAILURE``, ``$.qualification.unit_id``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 023
     - ``H23-pooled-unit`` / ``qualification-fixture-runner``
     - qualification verifier
     - ``CROSS_DOCUMENT_MISMATCH``, ``$.qualification.unit_id``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 024
     - ``H24-stale-evidence`` / ``qualification-fixture-runner``
     - evidence verifier
     - ``EXPIRY_OR_REPLAY_FAILURE``, ``$.qualification.evidence[0].expires_at``, P3, REJECT
     - DESIGN_MODEL receipt
   * - 025
     - ``H25-omitted-failure`` / ``qualification-fixture-runner``
     - qualification verifier
     - ``TRUST_BOUNDARY_FAILURE``, ``$.qualification.evidence``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 026
     - ``H26-missing-artifact`` / ``qualification-fixture-runner``
     - qualification verifier
     - ``CROSS_DOCUMENT_MISMATCH``, ``$.qualification.artifacts``, P5, REJECT
     - DESIGN_MODEL receipt
   * - 027
     - ``H27-false-build-pass`` / ``delivery-fixture-runner``
     - delivery gate
     - ``BINDING_INTEGRITY_FAILURE``, ``$.delivery.clean_checkout``, P6, REJECT
     - DESIGN_MODEL receipt
   * - 028
     - ``H28-false-dt-pass`` / ``delivery-fixture-runner``
     - delivery gate
     - ``BINDING_INTEGRITY_FAILURE``, ``$.delivery.dt_validation``, P6, REJECT
     - DESIGN_MODEL receipt
   * - 029
     - ``H29-false-physical-pass`` / ``delivery-fixture-runner``
     - qualification gate
     - ``TRUST_BOUNDARY_FAILURE``, ``$.qualification.evidence``, P6, HOLD
     - DESIGN_MODEL receipt
   * - 030
     - ``H30-failure-tuple-duplicate`` / ``catalog-fixture-runner``
     - failure catalog
     - ``DUPLICATE_SEMANTIC_KEY``, ``$.failure_catalog[1]``, P1, REJECT
     - DESIGN_MODEL receipt
   * - 031
     - ``H31-f02-conflict`` / ``catalog-fixture-runner``
     - pre-admission gate
     - ``F02_CONTRACT_CONFLICT``, ``$.f02_catalog``, P0, HOLD
     - DESIGN_MODEL receipt
   * - 032
     - ``H32-unknown-field`` / ``schema-fixture-runner``
     - strict parser
     - ``UNKNOWN_FIELD``, ``$.payload.components.kernel``, P1, REJECT
     - DESIGN_MODEL receipt

The hostile design census is therefore exactly 32 unique rows, orders 001
through 032, with 32 ``DESIGN_MODEL`` signals and zero runtime signals in this
branch. The future 154-row external F-02 catalog is not copied or claimed as
K-01 evidence; K-01's 32 rows are its consumer-specific closure and remain
unimplemented.

The deterministic scratch model used for this correction must execute each
guard in memory, plant the single listed violation, and print one of these
signals: ``DESIGN_MODEL:HOLD_TUPLE``, ``DESIGN_MODEL:REJECT_TUPLE``, or
``DESIGN_MODEL:F02_CONTRACT_CONFLICT``. It must never report runtime PASS from
absence of a validator. A scratch result is evidence that the model exercised
the design, not evidence that a consumer exists.

Delivery and release honesty
----------------------------

The present repository state has no schema package, generated binding,
compiled lock, lock validator, hostile fixture runner, consumer guard, CI
workflow, clean-checkout K-01 build, available complete DT-schema toolchain,
signed artifact set, signed qualification profile, physical unit records, or
release evidence. Missing ``dtc``, ``dt-validate``, ``dt_binding_check``,
Sphinx/docutils, PyYAML, JCS, or equivalent tools are ``TOOLING_BLOCK`` and
never PASS. The design does not run a build or DT validator through the dirty
checkout.

The exact status census is:

.. list-table:: Final honesty census
   :header-rows: 1
   :widths: 32 20 48

   * - Claim
     - Status
     - Boundary
   * - Design contract and structural counts
     - PASS, design-only
     - Exact paths, seams, tuples, counts, and future acceptance gates are specified.
   * - Scratch hostile models
     - DESIGN_MODEL
     - In-memory planted violations only; no runtime implementation.
   * - Schema, bindings, locks, validators, fixtures, consumer guards
     - NOT_IMPLEMENTED
     - No executable artifact exists in this lane.
   * - DT validation and full kernel reproducibility
     - TOOLING_BLOCK / NOT_IMPLEMENTED
     - Required tools, clean checkout, and locked reports are absent.
   * - Signed artifacts and qualification evidence
     - NOT_IMPLEMENTED
     - No authenticated future records or physical evidence exist.
   * - Support, merge eligibility, release readiness, K-01 DONE
     - REJECTED / NOT ELIGIBLE
     - This document cannot promote any board or slice.

The honest verdict is ``DESIGN_CONTRACT: PASS`` only for this design contract.
It is not an implementation PASS, qualification PASS, support claim, merge
approval, release claim, or DONE signal. F-02/F-03 remain provisional/HOLD;
implementation, validation, CI, DT validation, signed artifacts, physical
qualification, support, merge eligibility, and release readiness remain
absent. The coordinator alone may later change program status.
