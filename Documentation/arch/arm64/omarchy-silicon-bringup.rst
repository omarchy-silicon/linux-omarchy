.. SPDX-License-Identifier: GPL-2.0-only

============================================
Omarchy Silicon Apple Linux bring-up design
============================================

:Status: DESIGN ONLY. Nothing in K-01 through K-06 is DONE.
:Owner: ``linux-omarchy`` downstream kernel/DT design lane
:Program: ``omarchy-apple-platform/PROGRAM.md`` (2026-09-02)

This document defines the downstream kernel and device-tree plan for K-01
through K-06. It is a design note, not a support declaration, a release note,
or a compiled-kernel qualification. It does not change kernel or device-tree
code, configuration, bindings, or build output. A board is not supported merely
because this tree recognizes its SoC, because a DTB compiles, or because a
kernel boots.

The design uses the candidate immutable signed ref ``asahi-7.1.9-1`` peeled to
``77cb8f24c2381a8abb7272d7bbdec548d6426a8a`` (2026-05-30), together with the
current Apple bindings, ``arch/arm64/configs/asahi.config``, the Apple DT
inventory, the kernel KUnit/kselftest and documentation guidance, and the
Apple/Asahi entries in ``MAINTAINERS``. The repository has no ``.github``
workflow tree at that snapshot. The candidate's tag object and GPG evidence
are explicitly unavailable locally, so this source tuple is not authenticated
or release-ready. CI described here is therefore a proposed
Omarchy/platform boundary, not an observed property of the source repository.

The bootloader artifact is an opaque, human-produced boundary. This note
defines only the external artifact and handoff contract needed by the kernel
and release manifest; it does not inspect, reproduce, or make source-level
claims about that boundary.

Design invariants
-----------------

* The coordinator owns support states, the board registry, the platform
  manifest, qualification records, and promotion. This document cannot mark a
  slice DONE.
* A kernel source SHA, config digest, DTB digest, firmware bundle, Mesa build,
  boot artifact, and userspace image are one compatibility tuple. No consumer
  may silently select a different member of the tuple.
* Unknown board IDs, SoC IDs, DT compatible strings, firmware schemas, or
  manifest versions fail closed at admission or probe. A missing required
  component is an error, never a warning followed by success.
* Downstream work is a small, ordered patch queue with an upstreaming record.
  A fork is an ownership and release boundary, not permission for permanent
  divergence.
* Physical board evidence is required for every promotion. Build, static DT
  validation, emulation, chip recognition, and desktop boot are lower gates
  only.
* Release artifacts are built from pinned source and signed by the platform
  release process. Debug artifacts are separately named and never satisfy a
  release or support gate.

Canonical manifest contract and the F-02 dependency
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

F-02 is the platform-manifest acceptance slice. It has not been accepted at
this reviewed tip. K-01 consumes the frozen F-02 contract and does not define
a parallel manifest, registry, owner registry, or digest authority. Until an
accepted, signed F-02 schema revision and its signed context exist, K-01 is
``FAIL_CLOSED`` and cannot be reported as PASS, DONE, or implementation-ready.

Every authenticated object is strict UTF-8 closed JSON under the one common
``omarchy-signed/v1`` envelope. The envelope has exactly ``format``,
``payload_type``, ``payload_version``, ``domain``, ``context``,
``schema_set_digest``, ``payload``, and ``signatures[]``. The payload has its
exact common fields ``schema``, ``schema_set_digest``, ``document_id``,
``issuer``, ``issued_at``, and ``expires_at``. ``Trusted<T>`` is produced only
by the verifier after the envelope, signature context, schema set, key policy,
validity window, and replay checks pass. RFC 8785 JCS canonicalizes the payload
and ``payload_digest = SHA-256(JCS(payload))`` is computed outside the payload;
it is carried in the authenticated signature preimage and verifier metadata,
never inserted into the payload. ``document_id`` is a stable identifier and is
never a content digest. Unknown keys, duplicate keys, non-UTF-8 input,
non-canonical numbers, digest mismatch, signature failure, expiry, replay, or
an unknown schema rejects the object.

The authenticated payload vocabulary is the five primary documents
``board-registry/v1``, ``platform-manifest/v1``, ``installer-plan/v1``,
``qualification-record/v1``, and ``boot-health/v1``, plus
``owner-approval/v1``, ``boot-success-mark/v1``, and
``dtb-mutation-envelope/v1``. K-01 consumes these typed payloads only through
the common verifier; ``owner-approval/v1`` does not create an owner registry,
and a missing or untrusted payload is a hard rejection.

The authoritative K-01 manifest paths are under
``Trusted<PlatformManifest>.payload``. The component names and relationship
paths below are the complete K-01 closure. A path not listed here cannot
override one that is listed.

.. list-table:: Frozen platform-manifest/v1 paths consumed by K-01
   :header-rows: 1
   :widths: 35 50 15

   * - K-01 fact
     - Exact F-02 canonical path
     - Required result
   * - Signed manifest metadata
     - ``Trusted<PlatformManifest>.payload.schema``;
       ``Trusted<PlatformManifest>.payload.schema_set_digest``;
       ``Trusted<PlatformManifest>.payload.document_id``;
       ``Trusted<PlatformManifest>.payload.issuer``;
       ``Trusted<PlatformManifest>.payload.issued_at``;
       ``Trusted<PlatformManifest>.payload.expires_at``;
       ``Trusted<PlatformManifest>.payload_digest``;
       ``Trusted<PlatformManifest>.signatures[]``
     - One verified platform-manifest/v1 document; the stable ID and payload
       digest are distinct and no local envelope is substituted.
   * - Manifest targets and schema closure
     - ``Trusted<PlatformManifest>.payload.board_registry_digest``;
       ``Trusted<PlatformManifest>.payload.board_targets[]``;
       ``Trusted<PlatformManifest>.payload.qualification_bindings[]``;
       ``Trusted<PlatformManifest>.payload.firmware_schema``;
       ``Trusted<PlatformManifest>.payload.consumer_schema_set``;
       ``Trusted<PlatformManifest>.payload.minimum_consumer_api``
     - Explicit board targets, registry/schema identity, qualification
       bindings, and consumer compatibility are required.
   * - Manifest identity and board identity
     - ``Trusted<PlatformManifest>.payload.board_id``;
       ``Trusted<PlatformManifest>.payload.identity_match``
     - Board selection uses one complete typed identity match.
   * - Identity compatible tuple
     - ``Trusted<PlatformManifest>.payload.identity_match.ordered_linux_compatible``;
       ``Trusted<PlatformManifest>.payload.identity_match.soc_compatible``;
       ``Trusted<PlatformManifest>.payload.identity_match.product_model``;
       ``Trusted<PlatformManifest>.payload.identity_match.firmware_identity``;
       ``Trusted<PlatformManifest>.payload.identity_match.provenance``
     - All five source values agree exactly; no inference from one token.
   * - Linux source and provenance
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue``
     - One closed source/patch-queue record defined below.
   * - Linux kernel ABI
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.abi``
     - Kernel userspace and DRM ABI identity is typed and signed.
   * - Linux configuration
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.required_symbols[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.initramfs_module_closure[]``
     - Profile, preimage, normalized config, symbol closure, and initramfs
       closure are all locked.
   * - Linux toolchain and reports
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock``
     - Commands, versions, report schema, status, and artifacts are locked.
   * - DTB source and schema
     - ``Trusted<PlatformManifest>.payload.components.dtb_set.source``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.binding_schema``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.abi``
     - DTS inventory, binding-set identity, DT ABI, and source digest agree.
   * - DTB artifacts and mutation
     - ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.mutation_envelope``
     - Pre/post DTB digests and the authenticated mutation envelope are
       coupled to this manifest.
   * - Firmware bundle and ABI
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.bundle``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.tuning``
     - Generation, compatibility, tuning, schema, and signed bundle digest
       are explicit.
   * - Mesa stack
     - ``Trusted<PlatformManifest>.payload.components.mesa_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.abi``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.artifacts[]``
     - GPU generation, kernel DRM ABI, firmware ABI, and Mesa artifact agree.
   * - Boot stack
     - ``Trusted<PlatformManifest>.payload.components.boot_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.abi``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.slots``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.boot_health``
     - The opaque artifact boundary is identified without source inspection.
   * - Kernel, DTB, firmware, Mesa, boot, and userspace artifacts
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.artifacts[]``
     - Every artifact has typed ID, producer, content digest, role, and report
       link; only this closed set may be selected.
   * - Compatibility relations
     - ``Trusted<PlatformManifest>.payload.compatibility_relations[]``
     - Each relation has typed endpoints, relation kind, predicate, and
       evidence digest; all required relations are bidirectional where stated.
   * - Rollback closure
     - ``Trusted<PlatformManifest>.payload.rollback.set[]``;
       ``Trusted<PlatformManifest>.payload.rollback.last_known_good``
     - The set is atomic and points to complete accepted component records.
   * - Package, lock, and evidence closure
     - ``Trusted<PlatformManifest>.payload.package_set``;
       ``Trusted<PlatformManifest>.payload.locks.schema_set``;
       ``Trusted<PlatformManifest>.payload.locks.toolchain``;
       ``Trusted<PlatformManifest>.payload.locks.report``;
       ``Trusted<PlatformManifest>.payload.evidence``
     - Schema, toolchain, report, raw-log, signature, and retention identities
       are part of the same manifest.

The canonical source/patch-queue record at
``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue``
is closed and contains
``authoritative_url``, ``immutable_signed_ref``, ``tag_object_id``,
``peeled_commit``, ``signer_fingerprint``,
``signature_verification_evidence_digest``,
``signature_verification_result``, ``fetch_ref_advertisement_digest``,
``previous_base``, ``ordered_patch_ledger``, ``queue_tip``, and
``range_diff_digest`` plus the closed ``command_arrays`` object. The current
candidate values are immutable tag
``asahi-7.1.9-1`` and peeled commit
``77cb8f24c2381a8abb7272d7bbdec548d6426a8a``. The tag object ID, signer
fingerprint, verification evidence digest, and verification result are split
between known and unknown evidence: the locally resolved tag object is
``f3bed724fe7160d4a1f9dfb35a6e68f55153d41a``, while GPG verification is
unavailable and therefore the signer fingerprint, verification evidence
digest, and verification result remain explicit ``UNKNOWN`` candidate gaps.
The tag object and peeled commit do not imply a passed signature check.

The config lock at
``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock``
records the exact generated ``olddefconfig`` preimage bytes,
its digest, the normalized sorted ``CONFIG_SYMBOL=value`` serialization and
digest, the base defconfig and fragment identities, every required symbol's
value and reason, the built-in/module classification, and the complete ordered
initramfs dependency/signature closure. A missing, unexpectedly modular,
unsigned, extra, or unexplained dependency fails admission. The report lock
retains the raw input and output digests and the report schema/status contract.

The aggregate rows above are closed typed objects, not shorthand aliases. The
following leaf paths make the closure mechanically addressable:

.. list-table:: Canonical K-01 leaf paths
   :header-rows: 1
   :widths: 32 53 15

   * - Record
     - Exact canonical leaf paths
     - Missing result
   * - Source provenance
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.authoritative_url``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.immutable_signed_ref``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.tag_object_id``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.peeled_commit``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.signer_fingerprint``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.signature_verification_evidence_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.signature_verification_result``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.fetch_ref_advertisement_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.previous_base``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.queue_tip``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.range_diff_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.fetch_ref``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.verify_ref``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.apply_queue``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.range_diff``
     - ``PROVENANCE_BLOCK``
   * - Ordered patch ledger
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].ordinal``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].commit_id``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].patch_blob_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].subject``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].author``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].committer``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].source_or_review_ref``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].upstream_status``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].dependency``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.ordered_patch_ledger[].retirement_condition``
     - ``QUEUE_BLOCK``
   * - Config preimage and symbols
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.profile``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.preimage.bytes_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.preimage.normalized_bytes_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.base_defconfig.digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.fragments[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.required_symbols[].name``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.required_symbols[].value``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.required_symbols[].reason``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.required_symbols[].linkage``
     - ``CONFIG_CLOSURE_FAIL``
   * - Initramfs closure
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.initramfs_module_closure[].module``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.initramfs_module_closure[].dependency_closure[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.initramfs_module_closure[].vermagic``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.initramfs_module_closure[].signature_key_id``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.initramfs_module_closure[].digest``
     - ``CONFIG_CLOSURE_FAIL``
   * - Toolchain lock
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.recipe_id``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.recipe_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.compiler``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.compiler_version``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.compiler_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.rust``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.llvm``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.dtc``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.dt_schema``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.python``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.sphinx``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.builder_image``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.locale``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.make_variables``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays[]``
     - ``TOOLING_BLOCK``
   * - Report lock
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock.schemas[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock.status_values[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock.retention``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock.expected_artifacts[]``
     - ``REPORT_BLOCK``
   * - Component records
     - The exact component paths are
       ``Trusted<PlatformManifest>.payload.components.linux_kernel``,
       ``Trusted<PlatformManifest>.payload.components.dtb_set``,
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle``,
       ``Trusted<PlatformManifest>.payload.components.mesa_stack``, and
       ``Trusted<PlatformManifest>.payload.components.boot_stack``. Each has
       ``component_schema``, ``component_id``, ``source.source_kind``,
       ``source.repository_id``, ``source.source_commit``,
       ``source.upstream_commit``, ``source.source_digest``,
       ``source.provenance_report_digest``, ``provenance.provenance_schema``,
       ``provenance.source_observation_digest``,
       ``provenance.build_input_digest``, ``provenance.attestation_digest``,
       ``provenance.producer_binding_digest``, ``recipe_digest``,
       ``abi_contract_id``, ``config_inputs[]``, ``policy_inputs[]``,
       ``patch_lock``, ``toolchain_lock``, ``report_lock``, ``artifacts[]``,
       ``rollback``, and ``compatibility_relations[]``
     - ``COMPONENT_TUPLE_QUARANTINE``
   * - Artifact records
     - ``Trusted<PlatformManifest>.payload.artifacts[].artifact_id``;
       ``Trusted<PlatformManifest>.payload.artifacts[].kind``;
       ``Trusted<PlatformManifest>.payload.artifacts[].component_id``;
       ``Trusted<PlatformManifest>.payload.artifacts[].media_type``;
       ``Trusted<PlatformManifest>.payload.artifacts[].size_bytes``;
       ``Trusted<PlatformManifest>.payload.artifacts[].content_digest``;
       ``Trusted<PlatformManifest>.payload.artifacts[].artifact_version``;
       ``Trusted<PlatformManifest>.payload.artifacts[].signature_policy_id``;
       the same artifact leaves are required at each of the five exact
       component artifact paths named in the preceding component row
     - ``ARTIFACT_TUPLE_QUARANTINE``
   * - Component inputs and patch lock
     - On each of the five exact component paths,
       ``config_inputs[].input_id``; ``config_inputs[].input_kind``;
       ``config_inputs[].source_digest``;
       ``config_inputs[].normalized_content_digest``;
       ``config_inputs[].policy_digest``; ``policy_inputs[].policy_id``;
       ``policy_inputs[].policy_version``; ``policy_inputs[].policy_digest``;
       ``policy_inputs[].source_digest``; ``patch_lock.mode``;
       ``patch_lock.entries[].patch_id``; ``patch_lock.entries[].source_digest``;
       ``patch_lock.entries[].patch_digest``; ``patch_lock.entries[].order``;
       ``patch_lock.lock_digest``
     - ``COMPONENT_LOCK_BLOCK``
   * - Toolchain and report entries
     - On ``Trusted<PlatformManifest>.payload.components.linux_kernel``,
       ``toolchain_lock.mode``; ``toolchain_lock.entries[].toolchain_id``;
       ``toolchain_lock.entries[].toolchain_version``;
       ``toolchain_lock.entries[].toolchain_digest``;
       ``toolchain_lock.entries[].flags_digest``; ``toolchain_lock.lock_digest``;
       ``report_lock.mode``; ``report_lock.entries[].report_id``;
       ``report_lock.entries[].report_kind``;
       ``report_lock.entries[].report_digest``;
       ``report_lock.entries[].producer_toolchain_digest``;
       ``report_lock.lock_digest``
     - ``TOOLING_BLOCK`` or ``REPORT_BLOCK``
   * - Component rollback
     - On each of the five exact component paths,
       ``rollback.coordinate_schema``; ``rollback.previous_component_ids[]``;
       ``rollback.artifact_ids[]``; ``rollback.retention_count``;
       ``rollback.rollback_policy_id``; ``rollback.rollback_policy_digest``
     - ``ROLLBACK_SET_QUARANTINE``
   * - Compatibility relations
     - ``Trusted<PlatformManifest>.payload.compatibility_relations[].left_component_id``;
       ``Trusted<PlatformManifest>.payload.compatibility_relations[].relation``;
       ``Trusted<PlatformManifest>.payload.compatibility_relations[].right_component_id``;
       ``Trusted<PlatformManifest>.payload.compatibility_relations[].contract_id``;
       ``Trusted<PlatformManifest>.payload.compatibility_relations[].evidence_digest``.
       Each of the five exact component records named above repeats the same
       closed relation-member set under its own
       ``compatibility_relations[]`` path. ``contract_id`` is the closed typed
       predicate; no shell expression or free-form relation is accepted.
     - ``COMPATIBILITY_QUARANTINE``
   * - Rollback records
     - ``Trusted<PlatformManifest>.payload.rollback.set[]``;
       ``Trusted<PlatformManifest>.payload.rollback.last_known_good``;
       ``Trusted<PlatformManifest>.payload.rollback.last_known_good.document_id``;
       ``Trusted<PlatformManifest>.payload.rollback.last_known_good.payload_digest``;
       ``Trusted<PlatformManifest>.payload.rollback.failure_attempt_limit``
     - ``ROLLBACK_SET_QUARANTINE``

Every listed array is closed, ordered, and duplicate-rejected by the accepted
F-02 schema. A consumer resolves leaf paths from the single
``Trusted<PlatformManifest>`` value and rejects a missing leaf, unknown nested
key, alternate relation, or aggregate-only record; it never reconstructs a
leaf from an abbreviated member name.

Line-oriented records and any path outside this table have no authority in this
contract. A consumer must reject them rather than translate them into the
canonical paths above. There is exactly one manifest document ID and one
separately computed payload digest; no component, artifact, board, or report
may introduce a second identity or digest.

K-01: source, queue, configuration, and interface contract
-----------------------------------------------------------

Upstream sync and minimal patch queue
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The queue starts at the immutable signed ref and peeled commit recorded in
``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue``
and is rebased or recreated from that recorded upstream reference. The
downstream branch must not become a second unreviewable kernel history.

.. list-table:: Queue layers
   :header-rows: 1
   :widths: 18 24 58

   * - Layer
     - Owner and review
     - Rule
   * - Upstream base
     - Kernel coordinator; Asahi maintainers
     - Record ``authoritative_url``, immutable signed ref, tag object ID, peeled
       commit, fetch/ref advertisement digest, signature-verification evidence,
       and ``range_diff_digest`` against the previous base. Do not use a moving
       branch, unverified tag, or unrecorded local merge.
   * - Accepted upstream work
     - Relevant Linux/Asahi subsystem maintainers
     - Prefer the upstream commit or immutable reviewed series. Preserve
       author, review, ``Fixes``, and upstream link metadata.
   * - Omarchy integration
     - Kernel owner plus platform-manifest owner
     - Carry only the smallest compatibility, release, or board contract that
       is needed before upstream acceptance. State why it cannot yet be
       upstream and name its retirement condition.
   * - Board DT/config
     - DT owner plus subsystem owner
     - Keep board-specific facts in the board layer and keep common SoC facts
       in the SoC layer. Do not use a board patch to mask an unowned driver or
       firmware gap.

Every downstream commit has one purpose, one owner, a source or evidence
reference, and a test record. A patch that changes a binding, driver, DT, and
config is split when the dependency graph allows it; otherwise the commit
message names the dependency and the smallest independently reviewable unit.
Generated DTBs, merged configs, logs, and firmware blobs are not committed to
the source queue. Their digests belong in the release manifest and evidence
record.

The source/patch-queue record is self-contained at the frozen F-02 path
``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue``.
Its
``ordered_patch_ledger`` contains each ordinal, commit ID, patch-blob digest,
subject, author, committer, source or review reference, upstream status,
dependency link, and retirement condition. ``queue_tip`` is the exact commit
reached by applying that ledger. A local remote name, a moving branch, a date,
or a source SHA without the signed-ref evidence is not provenance.

Reconstruction starts from ``authoritative_url``, fetches only
``immutable_signed_ref``, verifies the tag object and its peeled commit,
checks the advertised-ref digest, applies ``ordered_patch_ledger`` in order,
checks ``queue_tip``, and recomputes ``range_diff_digest`` against
``previous_base``. The report retains the exact fetch/ref command array,
advertisement, verification output, patch order, range-diff, environment,
toolchain recipe, and output digests. An unavailable signed ref, tag object,
fingerprint, GPG result, patch entry, source record, or range-diff is
``FAIL_CLOSED`` rather than best effort.

The source operations are themselves complete lock-derived argv arrays at
these canonical paths:
``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.fetch_ref``;
``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.verify_ref``;
``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.apply_queue``;
and
``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue.command_arrays.range_diff``.
Each array contains every executable, argument, repository URL/ref, output
path, and environment assignment needed for that operation. The runner passes
the arrays directly with no shell expansion, caller-supplied remote, moving
branch, or appended option.

The sync job should fetch the allowlisted upstream remote, verify the expected
ref, apply the queue in order, run the source and DT gates, and publish a
machine-readable report containing:

* upstream base SHA and downstream tip SHA;
* queue commit SHAs, patch subjects, upstream status, and retirement links;
* changed-file census, DTB/config inventories, and generated artifact digests;
* range-diff from the preceding sync;
* every failed gate and every board or firmware capability that needs
  requalification.

The queue is rebased before a new release candidate, not during a physical
qualification run. A candidate is frozen at a source SHA. If an upstream sync
changes a driver, binding, DT, ABI, firmware interface, or GPU interface, the
affected board matrix is invalidated until the compatibility report says which
rows can be reused.

Configs and ABI
~~~~~~~~~~~~~~~

The current ``arch/arm64/configs/asahi.config`` is a useful platform fragment,
not a complete Omarchy release contract. It currently enables, among other
things, ``CONFIG_ARCH_APPLE``, ``CONFIG_ARM64_16K_PAGES``, Rust, Apple
CPUFreq/CPU idle, mailbox/RTKit/SMC/PMGR/DART/SART, NVMe/PCIe/USB, DCP/AGX,
audio, camera/ISP, wireless, and Apple input support. The fragment must be
kept aligned with the driver and DT dependency graph.

K-01 should establish three generated, digest-addressed configuration
profiles, without copying a second authoritative config into the platform
repository:

* ``release``: the smallest production profile assembled from the selected
  arm64 base and the reviewed Asahi platform fragment;
* ``bringup``: release plus console, persistent logs, tracing, fault reporting,
  and board diagnostics needed for lab work;
* ``debug``: bringup plus deliberately expensive developer options such as
  extra DRM/AGX debugging, allocator guards, debug info, and optional
  instrumentation. It is not a supported kernel profile.

The generated result records the base defconfig, fragment SHAs, toolchain,
``olddefconfig`` output, and a normalized ``CONFIG_`` digest. A profile may not
turn a missing required driver into a module merely to obtain a successful
boot. Module versus built-in is decided by the boot dependency graph: console,
storage, required DART/SART, firmware transports, and the root filesystem path
must be available before userspace can report health.

The ABI policy has four layers:

* Kernel userspace ABI follows the kernel's stable ABI rules. New board support
  uses existing DRM, input, sound, hwmon, thermal, power-supply, networking,
  V4L2, and sysfs interfaces where possible. A board-specific private ioctl or
  sysfs file needs a separately reviewed ABI proposal and owner.
* Device-tree bindings are the firmware/kernel ABI. Existing properties are
  not repurposed, a compatible string is never broadened to hide a new
  hardware revision, and new properties are schema-described before DT use.
  Validate with ``dt_binding_check`` and ``dtbs_check``. The rules in
  ``Documentation/devicetree/bindings/ABI.rst`` and
  ``Documentation/process/maintainer-soc.rst`` apply.
* Firmware ABI is explicit. Firmware version, schema, required memory regions,
  mailbox protocol, crash behavior, and reset requirements are named in the
  manifest. The kernel does not infer compatibility from a marketing name.
* The release ABI is the platform manifest. It binds source, config, DTB,
  firmware, boot artifacts, Mesa, initramfs, and userspace to exact digests.
  The kernel document consumes that contract; it does not define a competing
  manifest schema.

Device-tree organization
~~~~~~~~~~~~~~~~~~~~~~~~

The current tree follows the Linux SoC naming model and provides a good shape
for future work:

.. list-table:: Current Apple DT layers
   :header-rows: 1
   :widths: 28 30 42

   * - Layer
     - Current examples
     - Contents and rule
   * - SoC
     - ``t8103.dtsi``, ``t8112.dtsi``, ``t8122.dtsi``, ``t8132.dtsi``
     - CPU topology, interrupt controller, timers, MMIO blocks, clocks,
       power domains, and SoC-level firmware clients. No chassis-specific
       panel, fan, port, or board ID.
   * - SoC family/common
     - ``t600x-common.dtsi``, ``t602x-common.dtsi``,
       ``t6031-base.dtsi``
     - Shared topology and registers for closely related parts. A cut-down
       or multi-die variant disables only proven absent blocks and names the
       reason in the source.
   * - Board common
     - ``t8103-jxxx.dtsi``, ``t8122-jxxx.dtsi``,
       ``t8132-jxxx.dtsi``
     - Common integration for a board class, including framebuffer handoff,
       common sensors, NVRAM, and shared peripheral wiring.
   * - Board
     - ``t8103-j274.dts`` through the current ``t8132-*.dts`` files
     - Exact target type, exact SoC ID, model, panel/audio/camera/USB-PD
       choices, disabled blocks, GPIOs, and board-specific regulators.
   * - Build inventory
     - ``arch/arm64/boot/dts/apple/Makefile``
     - Explicit DTB list. A DT source is not in the support inventory until it
       is listed, schema-checked, built, and mapped to a board-registry record.

Apple root nodes use the existing three-part compatible contract:

.. code-block:: text

   compatible = "apple,<target-type>", "apple,<soc-id>", "apple,arm-platform";

The target type and SoC ID above are Linux compatible strings, not canonical
platform registry identifiers. A family or marketing-compatible fallback
cannot replace either value. New DT files must follow
``Documentation/devicetree/bindings/arm/apple.yaml`` and the DT coding style.

Board identity is admitted only from
``Trusted<PlatformManifest>.payload.board_id`` and its complete typed
``Trusted<PlatformManifest>.payload.identity_match`` tuple. The tuple has
exactly ``ordered_linux_compatible``, ``soc_compatible``,
``product_model``, ``firmware_identity``, and ``provenance``. Every source
contributing a tuple member is named and signed; source order is preserved. A
compatible string is accepted only when the full tuple maps to exactly one
board ID. A product name,
SoC token, serial label, or compatible string cannot fill another member by
inference.

Any missing, duplicate, conflicting, expired, or unverifiable tuple member
quarantines the board and prevents DTB, firmware, config, boot, or physical
profile selection. The j713 case is an explicit hostile fixture: the M3 source
and M4 source both contain the same ``j713`` token, while their typed SoC,
product/model, firmware, or provenance members conflict. Token equality does
not reconcile those records; the board is quarantined until a new signed
reconciliation names both sources, the resolved board ID, every tuple member,
the owner, and the approval. Family or generation inference cannot lift it.

Common includes are additive and auditable. A common file may not silently
override a board's safety-critical power, thermal, audio, or display behavior.
New family files first land with the binding, minimal SoC description, and a
single observed board; additional boards follow as separate, reviewable
patches. Merging two similar SoCs into one DTSI is allowed only when the
register layout, firmware ABI, power topology, and applicable DT properties
are proven compatible.

AGX binding acceptance gate
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The accepted AGX baseline is exactly the binding currently named by
``Documentation/devicetree/bindings/gpu/apple,agx.yaml``. Its covered
compatibles are ``apple,agx-g13g``, ``apple,agx-g13s``,
``apple,agx-g14g``, and ``apple,agx-g14s``; the two-element forms are
``apple,agx-g13c`` or ``apple,agx-g13d`` followed by ``apple,agx-g13s``, and
``apple,agx-g14c`` or ``apple,agx-g14d`` followed by ``apple,agx-g14s``.
The covered properties are ``compatible``, ``reg``, ``reg-names`` with
``asc``/``sgx``, ``power-domains``, ``mboxes``, ``memory-region`` with its six
named regions, ``memory-region-names``, and ``apple,firmware-abi``. The
binding requires ``compatible``, ``reg``, ``mboxes``, ``memory-region``, and
``apple,firmware-abi`` and sets ``additionalProperties: false``. No later AGX
compatible or property is admitted by analogy.

The pinned clean source census is a gate input, not a support claim. The
census was mechanically derived from every clean AGX base node and ``&gpu``
overlay in the pinned source tree. Source locations below are relative to
``arch/arm64/boot/dts/apple/``. The ten current compatible literals not
accepted by the binding are:

.. list-table:: Unsupported AGX compatible census
   :header-rows: 1
   :widths: 34 42 24

   * - Exact literal
     - Clean source path and line
     - One disposition
   * - ``apple,agx-t6000``
     - ``arch/arm64/boot/dts/apple/t6000.dtsi:40:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-t6001``
     - ``arch/arm64/boot/dts/apple/t6001.dtsi:86:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-t6002``
     - ``arch/arm64/boot/dts/apple/t6002.dtsi:381:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-t6020``
     - ``arch/arm64/boot/dts/apple/t6020.dtsi:30:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-t6021``
     - ``arch/arm64/boot/dts/apple/t6021.dtsi:77:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-t6022``
     - ``arch/arm64/boot/dts/apple/t6022.dtsi:383:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-t8103``
     - ``arch/arm64/boot/dts/apple/t8103.dtsi:478:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-t8112``
     - ``arch/arm64/boot/dts/apple/t8112.dtsi:509:/gpu``
     - DTS removal/correction; retain only a validated binding tuple.
   * - ``apple,agx-g13x``
     - ``arch/arm64/boot/dts/apple/t6000.dtsi:40:/gpu``
     - DTS removal/correction; use an explicitly supported ordered form.
   * - ``apple,agx-g14x``
     - ``arch/arm64/boot/dts/apple/t6020.dtsi:30:/gpu``;
       ``t6021.dtsi:77:/gpu``; ``t6022.dtsi:383:/gpu``
     - DTS removal/correction; use an explicitly supported ordered form.

The mechanically enumerated unsupported property paths are below. Each row is
an exact property under the named ``/gpu`` node or ``&gpu`` overlay. Every row
has one disposition: a binding extension with a typed schema, source
references, owner approval, and a retirement/upstreaming record. The extension
must land before the property is retained; until then the affected board is
quarantined and its DT warning cannot be waived.

.. list-table:: Unsupported AGX property census
   :header-rows: 1
   :widths: 35 45 20

   * - Exact property path
     - Clean source locations
     - One disposition
   * - ``/gpu/apple,firmware-version``
     - ``t600x-die0.dtsi:753``; ``t602x-die0.dtsi:968``;
       ``t8103.dtsi:491``; ``t8112.dtsi:525``
     - Binding extension.
   * - ``/gpu/apple,firmware-compat``
     - ``t600x-die0.dtsi:754``; ``t602x-die0.dtsi:969``;
       ``t8103.dtsi:492``; ``t8112.dtsi:526``
     - Binding extension.
   * - ``/gpu/interrupt-parent``
     - ``t602x-die0.dtsi:952``
     - Binding extension.
   * - ``/gpu/interrupts``
     - ``t602x-die0.dtsi:953``
     - Binding extension.
   * - ``/gpu/operating-points-v2``
     - ``t600x-die0.dtsi:756``; ``t602x-die0.dtsi:971``;
       ``t8103.dtsi:494``; ``t8112.dtsi:528``
     - Binding extension.
   * - ``/gpu/apple,afr-opp``
     - ``t602x-die0.dtsi:973``
     - Binding extension.
   * - ``/gpu/apple,cs-opp``
     - ``t602x-die0.dtsi:972``
     - Binding extension.
   * - ``/gpu/apple,csafr-min-sram-microvolt``
     - ``t602x-die0.dtsi:976``
     - Binding extension.
   * - ``/gpu/apple,min-sram-microvolt``
     - ``t600x-die0.dtsi:758``; ``t602x-die0.dtsi:975``;
       ``t8103.dtsi:496``; ``t8112.dtsi:530``
     - Binding extension.
   * - ``/gpu/apple,avg-power-filter-tc-ms``
     - ``t600x-die0.dtsi:759``; ``t6020.dtsi:32``; ``t6021.dtsi:79``;
       ``t6022.dtsi:385``; ``t8103.dtsi:497``; ``t8112.dtsi:531``
     - Binding extension.
   * - ``/gpu/apple,avg-power-ki-only``
     - ``t6001-j375c.dts:54``; ``t6002-j375d.dts:222``;
       ``t600x-die0.dtsi:760``; ``t6020.dtsi:33``; ``t6021.dtsi:80``;
       ``t6022.dtsi:386``; ``t8103.dtsi:498``; ``t8112.dtsi:532``
     - Binding extension.
   * - ``/gpu/apple,avg-power-kp``
     - ``t6001-j375c.dts:55``; ``t6002-j375d.dts:223``;
       ``t600x-die0.dtsi:761``; ``t6020.dtsi:34``; ``t6021.dtsi:81``;
       ``t6022.dtsi:387``; ``t8103.dtsi:499``; ``t8112.dtsi:533``
     - Binding extension.
   * - ``/gpu/apple,avg-power-min-duty-cycle``
     - ``t600x-die0.dtsi:762``; ``t602x-die0.dtsi:979``;
       ``t8103.dtsi:500``; ``t8112.dtsi:534``
     - Binding extension.
   * - ``/gpu/apple,avg-power-target-filter-tc``
     - ``t6001-j375c.dts:56``; ``t6002-j375d.dts:224``;
       ``t600x-die0.dtsi:763``; ``t602x-die0.dtsi:980``;
       ``t8103.dtsi:501``; ``t8112.dtsi:535``
     - Binding extension.
   * - ``/gpu/apple,fast-die0-integral-gain``
     - ``t600x-die0.dtsi:764``; ``t6020.dtsi:35``; ``t6021.dtsi:82``;
       ``t6022.dtsi:388``; ``t8103.dtsi:502``; ``t8112.dtsi:536``
     - Binding extension.
   * - ``/gpu/apple,fast-die0-proportional-gain``
     - ``t600x-die0.dtsi:765``; ``t6022.dtsi:389``;
       ``t602x-die0.dtsi:981``; ``t8103.dtsi:503``; ``t8112.dtsi:537``
     - Binding extension.
   * - ``/gpu/apple,idleoff-standby-timer``
     - ``t6021-j475c.dts:120``; ``t6022.dtsi:390``
     - Binding extension.
   * - ``/gpu/apple,perf-base-pstate``
     - ``t6001-j375c.dts:57``; ``t6002-j375d.dts:225``;
       ``t600x-die0.dtsi:757``; ``t6020-j474s.dts:125``;
       ``t6021-j475c.dts:121``; ``t6022.dtsi:391``;
       ``t602x-die0.dtsi:977``; ``t8103-j274.dts:146``;
       ``t8103-j456.dts:164``; ``t8103-j457.dts:145``;
       ``t8103.dtsi:495``; ``t8112-j473.dts:214``; ``t8112.dtsi:529``
     - Binding extension.
   * - ``/gpu/apple,perf-boost-ce-step``
     - ``t600x-die0.dtsi:766``; ``t6021-j475c.dts:122``;
       ``t6022.dtsi:392``; ``t602x-die0.dtsi:982``; ``t8112.dtsi:538``
     - Binding extension.
   * - ``/gpu/apple,perf-boost-min-util``
     - ``t600x-die0.dtsi:767``; ``t6021-j475c.dts:123``;
       ``t6022.dtsi:393``; ``t602x-die0.dtsi:983``; ``t8112.dtsi:539``
     - Binding extension.
   * - ``/gpu/apple,perf-filter-drop-threshold``
     - ``t600x-die0.dtsi:768``; ``t602x-die0.dtsi:984``;
       ``t8103.dtsi:504``; ``t8112.dtsi:540``
     - Binding extension.
   * - ``/gpu/apple,perf-filter-time-constant``
     - ``t600x-die0.dtsi:769``; ``t602x-die0.dtsi:985``;
       ``t8103.dtsi:505``; ``t8112.dtsi:541``
     - Binding extension.
   * - ``/gpu/apple,perf-filter-time-constant2``
     - ``t600x-die0.dtsi:770``; ``t602x-die0.dtsi:986``;
       ``t8103.dtsi:506``; ``t8112.dtsi:542``
     - Binding extension.
   * - ``/gpu/apple,perf-integral-gain``
     - ``t600x-die0.dtsi:771``; ``t602x-die0.dtsi:987``;
       ``t8112.dtsi:543``
     - Binding extension.
   * - ``/gpu/apple,perf-integral-gain2``
     - ``t600x-die0.dtsi:772``; ``t602x-die0.dtsi:988``;
       ``t8103.dtsi:507``; ``t8112.dtsi:544``
     - Binding extension.
   * - ``/gpu/apple,perf-integral-min-clamp``
     - ``t600x-die0.dtsi:773``; ``t602x-die0.dtsi:989``;
       ``t8103.dtsi:508``; ``t8112.dtsi:545``
     - Binding extension.
   * - ``/gpu/apple,perf-proportional-gain``
     - ``t600x-die0.dtsi:774``; ``t602x-die0.dtsi:991``;
       ``t8112.dtsi:546``
     - Binding extension.
   * - ``/gpu/apple,perf-proportional-gain2``
     - ``t600x-die0.dtsi:775``; ``t602x-die0.dtsi:990``;
       ``t8103.dtsi:509``; ``t8112.dtsi:547``
     - Binding extension.
   * - ``/gpu/apple,perf-tgt-utilization``
     - ``t600x-die0.dtsi:776``; ``t6021-j475c.dts:124``;
       ``t6022.dtsi:394``; ``t602x-die0.dtsi:992``;
       ``t8103.dtsi:510``; ``t8112.dtsi:548``
     - Binding extension.
   * - ``/gpu/apple,power-sample-period``
     - ``t600x-die0.dtsi:777``; ``t602x-die0.dtsi:993``;
       ``t8103.dtsi:511``; ``t8112.dtsi:549``
     - Binding extension.
   * - ``/gpu/apple,power-zones``
     - ``t8103.dtsi:512``
     - Binding extension.
   * - ``/gpu/apple,ppm-filter-time-constant-ms``
     - ``t600x-die0.dtsi:778``; ``t6020.dtsi:36``; ``t6021.dtsi:83``;
       ``t602x-die0.dtsi:994``; ``t8103.dtsi:513``; ``t8112.dtsi:550``
     - Binding extension.
   * - ``/gpu/apple,ppm-ki``
     - ``t6001-j375c.dts:58``; ``t6002-j375d.dts:226``;
       ``t600x-die0.dtsi:779``; ``t6020.dtsi:37``; ``t6021.dtsi:84``;
       ``t6022.dtsi:395``; ``t602x-die0.dtsi:995``;
       ``t8103.dtsi:514``; ``t8112.dtsi:551``
     - Binding extension.
   * - ``/gpu/apple,ppm-kp``
     - ``t6001-j375c.dts:59``; ``t6002-j375d.dts:227``;
       ``t600x-die0.dtsi:780``; ``t6022.dtsi:396``;
       ``t602x-die0.dtsi:996``; ``t8103.dtsi:515``; ``t8112.dtsi:552``
     - Binding extension.
   * - ``/gpu/apple,pwr-filter-time-constant``
     - ``t600x-die0.dtsi:781``; ``t602x-die0.dtsi:997``;
       ``t8103.dtsi:516``
     - Binding extension.
   * - ``/gpu/apple,pwr-integral-gain``
     - ``t600x-die0.dtsi:782``; ``t602x-die0.dtsi:998``;
       ``t8103.dtsi:517``
     - Binding extension.
   * - ``/gpu/apple,pwr-integral-min-clamp``
     - ``t600x-die0.dtsi:783``; ``t602x-die0.dtsi:999``;
       ``t8103.dtsi:518``
     - Binding extension.
   * - ``/gpu/apple,pwr-min-duty-cycle``
     - ``t600x-die0.dtsi:784``; ``t602x-die0.dtsi:1000``;
       ``t8103.dtsi:519``; ``t8112.dtsi:553``
     - Binding extension.
   * - ``/gpu/apple,pwr-proportional-gain``
     - ``t600x-die0.dtsi:785``; ``t602x-die0.dtsi:1001``;
       ``t8103.dtsi:520``
     - Binding extension.
   * - ``/gpu/apple,pwr-sample-period-aic-clks``
     - ``t602x-die0.dtsi:1002``
     - Binding extension.
   * - ``/gpu/apple,se-engagement-criteria``
     - ``t602x-die0.dtsi:1003``
     - Binding extension.
   * - ``/gpu/apple,se-filter-time-constant``
     - ``t602x-die0.dtsi:1004``
     - Binding extension.
   * - ``/gpu/apple,se-filter-time-constant-1``
     - ``t602x-die0.dtsi:1005``
     - Binding extension.
   * - ``/gpu/apple,se-inactive-threshold``
     - ``t602x-die0.dtsi:1006``
     - Binding extension.
   * - ``/gpu/apple,se-ki``
     - ``t602x-die0.dtsi:1007``
     - Binding extension.
   * - ``/gpu/apple,se-ki-1``
     - ``t602x-die0.dtsi:1008``
     - Binding extension.
   * - ``/gpu/apple,se-kp``
     - ``t602x-die0.dtsi:1009``
     - Binding extension.
   * - ``/gpu/apple,se-kp-1``
     - ``t602x-die0.dtsi:1010``
     - Binding extension.
   * - ``/gpu/apple,se-reset-criteria``
     - ``t602x-die0.dtsi:1011``
     - Binding extension.
   * - ``/gpu/apple,core-leak-coef``
     - ``t600x-die0.dtsi:787``; ``t602x-die0.dtsi:1013``;
       ``t8103.dtsi:522``; ``t8112.dtsi:554``
     - Binding extension.
   * - ``/gpu/apple,sram-leak-coef``
     - ``t600x-die0.dtsi:788``; ``t602x-die0.dtsi:1014``;
       ``t8103.dtsi:523``; ``t8112.dtsi:555``
     - Binding extension.
   * - ``/gpu/apple,cs-leak-coef``
     - ``t602x-die0.dtsi:1015``
     - Binding extension.
   * - ``/gpu/apple,afr-leak-coef``
     - ``t602x-die0.dtsi:1016``
     - Binding extension.

The clean census found 10 unsupported compatible literals and 53 unsupported
property names. The accepted property set remains exactly the eight names in
``apple,agx.yaml``. A binding extension is not an approval: it requires the
typed property definition, units and bounds, source citation, signed owner
approval, regenerated schema report, zero-warning DTB report, and registry
mapping. Until every census row is resolved, the affected DTB and every
dependent board are ``QUARANTINED`` and a zero-warning success is impossible.
Any source path or property not present in this census is a new mismatch and
also blocks the gate. No current ``dtbs_check`` or hardware pass is claimed.

For each gate, the canonical
``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock`` stores
the schema-set ID and commit, DTS/DTB source and digest inventory, exact
``dtc`` and dt-schema versions, lock-derived argv array, and sorted
machine-readable warning report. A missing tool or report is
``TOOLING_BLOCK``; it is never converted to a schema PASS.

DTB digest and mutation boundary
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The DTB boundary consumes only ``Trusted<DtbMutationEnvelope>`` under the
frozen F-02 type. Its common envelope paths are authoritative; the payload
binds these exact fields:

.. list-table:: dtb-mutation-envelope/v1 bindings
   :header-rows: 1
   :widths: 38 42 20

   * - Field
     - Exact payload path
     - K-01 requirement
   * - Schema and source set
     - ``Trusted<DtbMutationEnvelope>.payload.schema_set.id``;
       ``Trusted<DtbMutationEnvelope>.payload.schema_set.version``;
       ``Trusted<DtbMutationEnvelope>.payload.source_identity``
     - Exact schema set and full DTS/DTB source identity.
   * - Manifest and board
     - ``Trusted<DtbMutationEnvelope>.payload.manifest_document_id``;
       ``Trusted<DtbMutationEnvelope>.payload.manifest_payload_digest``;
       ``Trusted<DtbMutationEnvelope>.payload.board_id``
     - Must equal the accepted platform manifest document ID, payload digest,
       and board ID.
   * - DTB bytes
     - ``Trusted<DtbMutationEnvelope>.payload.dtb.before_digest``;
       ``Trusted<DtbMutationEnvelope>.payload.dtb.after_digest``
     - SHA-256 is recomputed over the exact pre and post byte strings.
   * - Policy, tool, and artifact
     - ``Trusted<DtbMutationEnvelope>.payload.policy.id``;
       ``Trusted<DtbMutationEnvelope>.payload.policy.version``;
       ``Trusted<DtbMutationEnvelope>.payload.policy.digest``;
       ``Trusted<DtbMutationEnvelope>.payload.tool.id``;
       ``Trusted<DtbMutationEnvelope>.payload.tool.version``;
       ``Trusted<DtbMutationEnvelope>.payload.tool.digest``;
       ``Trusted<DtbMutationEnvelope>.payload.artifact.id``;
       ``Trusted<DtbMutationEnvelope>.payload.artifact.version``;
       ``Trusted<DtbMutationEnvelope>.payload.artifact.digest``
     - Exact approved versions and digests are required.
   * - Mutations
     - ``Trusted<DtbMutationEnvelope>.payload.authorized_mutations[]``
     - Ordered closed entries contain exact DTS path, operation,
       before-value digest, and after-value digest/value.
   * - Firmware and signer
     - ``Trusted<DtbMutationEnvelope>.payload.firmware.bundle_id``;
       ``Trusted<DtbMutationEnvelope>.payload.firmware.schema``;
       ``Trusted<DtbMutationEnvelope>.payload.expires_at``;
       ``Trusted<DtbMutationEnvelope>.payload.replay_identity``;
       ``Trusted<DtbMutationEnvelope>.signatures[]``
     - Firmware bundle/schema, expiry, replay identity, and the verified
       signer/signature entry are bound by the common envelope.

The producer may emit an envelope only after it has the exact manifest
document ID and payload digest, full source identity, pre-mutation DTB digest,
policy/tool/artifact locks, firmware schema, ordered mutation list, expiry,
and replay identity. The producer signs the common envelope and retains the
canonical bytes, signature evidence, before/after DTB bytes, and ordered diff.
For the K-01 baseline, the ordered allowlist contains only the exact AGX node
paths for ``apple,firmware-abi``; it is empty for every other property, node,
compatible, memory reservation, phandle, and boot argument. Any additional
entry requires a new binding, manifest schema, and policy revision before it
can be emitted.
The consumer independently verifies the signature and expiry, recomputes both
DTB digests and every value digest, checks the source/schema/policy/tool/
artifact/firmware tuple, checks ordered operations against the allowlist, and
records the resulting post digest in the platform manifest's DTB artifact.
Neither side inspects or relies on the opaque bootloader implementation.

The K-01 hostile fixture matrix is mandatory. Each condition is a hard
rejection and prevents boot-health success:

.. list-table:: dtb-mutation-envelope/v1 hostile fixtures
   :header-rows: 1
   :widths: 48 28 24

   * - Fixture
     - Rejection code
     - Required result
   * - Unauthorized node or property, including a new compatible, phandle,
       reservation, boot argument, or non-allowlisted property
     - ``DT_MUTATION_UNAUTHORIZED``
     - Reject; no DTB use.
   * - Wrong before-value digest or operation ordering
     - ``DT_MUTATION_BEFORE_MISMATCH``
     - Reject; no partial apply.
   * - Wrong policy, tool, or artifact ID/version/digest
     - ``DT_MUTATION_LOCK_MISMATCH``
     - Reject; quarantine the DTB.
   * - Stale, expired, or replayed envelope
     - ``DT_MUTATION_REPLAY``
     - Reject; retain the replay evidence.
   * - Source identity, board ID, manifest document ID, payload digest, or
       firmware bundle/schema transplanted from another tuple
     - ``DT_MUTATION_TUPLE_MISMATCH``
     - Reject; quarantine the board and release.
   * - Unknown mutation operation or mutation path
     - ``DT_MUTATION_UNKNOWN``
     - Reject; require a schema/policy revision.
   * - Claimed before or after digest differs from independently computed
       bytes, or a digest is mutable data inside the measured payload
     - ``DT_MUTATION_DIGEST_MISMATCH``
     - Reject; no claimed digest is trusted.
   * - Source identity or the independently computed post-mutation digest does
       not match the manifest's DTB source/artifact tuple
     - ``DT_MUTATION_DIGEST_MISMATCH``
     - Reject; quarantine the source, DTB, and dependent release tuple.
   * - Missing signature, signer, expiry, replay identity, source, or report
     - ``DT_MUTATION_INCOMPLETE``
     - Reject; ``TOOLING_BLOCK`` or ``UNKNOWN`` is not success.

The kernel accepts only the authenticated envelope and measured handoff. It
does not accept a DTB-carried digest, boot argument, or local alias as proof.
The lab records the raw handoff; the kernel records verification and measured
digests; the coordinator accepts the evidence. A future mutable field requires
a new F-02 schema and policy review.

Firmware interfaces
~~~~~~~~~~~~~~~~~~~

Apple peripherals are often firmware-mediated. The current kernel surface
includes Apple mailboxes and RTKit, SART address filtering, DART IOMMUs, PMGR
and PMP power control, SMC, AOP, SEP, AGX, DCP, NVMe, ISP, and audio clients.
The current Kconfig explicitly describes mailbox and RTKit as transports for
co-processors used by storage and display, and SART as an allow-list filter
used by clients such as the NVMe co-processor. These are ordering and memory
ownership constraints, not optional logging details.

The external firmware contract for each client contains:

* firmware bundle name, version, schema, provenance, signature, and digest;
* DT compatible and required properties, mailbox channel, SART/DART path,
  power domain, clocks, reset, and shared-memory regions;
* ownership and cacheability of every shared buffer, DMA address range,
  alignment, lifetime, and access direction;
* startup, timeout, crash, coredump, reset, and shutdown behavior;
* whether the client is required for boot, required for a capability, or
  optional for that exact board;
* the kernel driver ABI and Mesa/userspace consumer version, if applicable.

Firmware is rejected when its schema, ABI, board identity, or memory contract
does not match. A firmware crash, DART fault, SART rejection, stale shared
buffer, or failed reset is a failed capability row and is retained in the
evidence record. No generic firmware, DT, or power-domain fallback may make a
required row appear healthy.

The firmware ABI record is explicit and versioned at
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi``. Its
closed fields are
``generation``, ``compatibility``, ``tuning``, ``source``, ``digest``,
``signature_key_id``, ``min_kernel_abi``, ``max_kernel_abi``,
``required_dt_schema_set``, and ``required_mesa_abi`` where applicable.
``generation`` identifies the hardware and firmware generation;
``compatibility`` identifies the protocol tuple; and ``tuning`` identifies
the signed calibration, power, clock, and performance record. A DT property,
marketing name, or responding client is not provenance for any field.

The exact ABI leaf paths are
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.generation``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.compatibility``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.tuning``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.source``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.digest``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.signature_key_id``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.min_kernel_abi``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.max_kernel_abi``;
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.required_dt_schema_set``;
and
``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.required_mesa_abi``.
The bundle source and provenance are independently bound at
``Trusted<PlatformManifest>.payload.components.firmware_bundle.source`` and
``Trusted<PlatformManifest>.payload.components.firmware_bundle.provenance``;
the component bundle artifact is bound at
``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``
and the corresponding manifest artifact record.

The accepted constraint is exact for the manifest-selected generation and
ABI major, and is within the signed minor and patch bounds in the same record.
The kernel rejects absent, unknown, out-of-range, or signature-invalid
generation, compatibility, tuning, schema, memory, reset, or version data. A
client cannot silently negotiate an older protocol merely because it responds.
Update transitions are typed entries in
``Trusted<PlatformManifest>.payload.compatibility_relations[]`` binding
``Trusted<PlatformManifest>.payload.components.firmware_bundle.bundle.digest``
from the old artifact to the new artifact for the exact board ID and manifest
document ID. An unlisted upgrade or downgrade is refused. Updates are staged,
verified, and committed atomically, with the last-known-good bundle retained by
``Trusted<PlatformManifest>.payload.rollback.last_known_good``.

The exact firmware prerequisite and rejection are fixed by the following
relations in ``Trusted<PlatformManifest>.payload.compatibility_relations[]``.
Each row is a closed relation record with exact paths
``Trusted<PlatformManifest>.payload.compatibility_relations[].left_component_id``,
``Trusted<PlatformManifest>.payload.compatibility_relations[].relation``,
``Trusted<PlatformManifest>.payload.compatibility_relations[].right_component_id``,
``Trusted<PlatformManifest>.payload.compatibility_relations[].contract_id``,
and ``Trusted<PlatformManifest>.payload.compatibility_relations[].evidence_digest``:

.. list-table:: Firmware ABI relation closure
   :header-rows: 1
   :widths: 34 44 22

   * - Required fact
     - Typed relation endpoints
     - Unknown or unfrozen result
   * - Firmware generation and compatibility
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.generation``
       and
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.compatibility``
       to ``Trusted<PlatformManifest>.payload.components.linux_kernel.abi``
     - ``FIRMWARE_ABI_QUARANTINE``.
   * - Firmware tuning and DT schema
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.tuning``
       and
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.required_dt_schema_set``
       to ``Trusted<PlatformManifest>.payload.components.dtb_set.binding_schema``
     - ``DT_SCHEMA_QUARANTINE``.
   * - Firmware and Mesa
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi.required_mesa_abi``
       to ``Trusted<PlatformManifest>.payload.components.mesa_stack.abi``
     - ``MESA_ABI_QUARANTINE``.
   * - Firmware and boot stack
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.bundle``
       to ``Trusted<PlatformManifest>.payload.components.boot_stack.abi`` and
       ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``
     - ``BOOT_TUPLE_QUARANTINE``.
   * - Kernel, DTB, and firmware artifacts
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.artifacts[]``,
       ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``, and
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``
       to the corresponding typed entries in
       ``Trusted<PlatformManifest>.payload.artifacts[]``
     - ``ARTIFACT_TUPLE_QUARANTINE``.
   * - Manifest and rollback
     - ``Trusted<PlatformManifest>.payload.document_id`` to
       ``Trusted<PlatformManifest>.payload.rollback.set[]`` and
       ``Trusted<PlatformManifest>.payload.rollback.last_known_good``
     - ``ROLLBACK_SET_QUARANTINE``.

The relation is not to be frozen later: the prerequisite is an accepted F-02
``Trusted<PlatformManifest>.payload.compatibility_relations[]`` entry with
typed endpoints, a predicate, and evidence digest, authenticated in the same
platform manifest. Any unknown or unfrozen tuple is quarantined and the
release is rejected. Recovery selects the signed last-known-good manifest and
its complete rollback set; it never mixes a new kernel or DTB with an old
firmware member. Unknown reset, crash, shared-memory, mailbox, or coredump
behavior is a failed ABI admission, not an optional diagnostic.

The GPU binding currently carries ``apple,firmware-abi`` and describes the
calibration, globals, handoff, and page-table regions consumed by AGX. The
current Rust AGX driver also reads ``apple,firmware-compat``. Their ownership,
versioning, overwrite semantics, and relationship to the platform manifest
are represented only by the typed component and relation paths above. Until
the required binding extension and accepted F-02 relation entries exist, the
entire AGX census and every unknown ABI value remain quarantined and cannot
satisfy K-01.

Debug builds and diagnostics
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Bring-up and debug kernels are named variants with a separate config digest,
symbol package, toolchain record, and retention policy. They may enable
``CONFIG_DEBUG_KERNEL``, ``CONFIG_DEBUG_INFO_DWARF5``, ``CONFIG_KALLSYMS_ALL``,
``CONFIG_FRAME_POINTER``, ``CONFIG_DEBUG_FS``, dynamic debug, ftrace, pstore,
lock/memory debugging, and selected fault-injection options. ``DRM_APPLE_DEBUG``
and ``DRM_ASAHI_DEBUG_ALLOCATOR`` are lab options; the latter is intentionally
slow and can expose firmware bugs, so it is never part of a release image.

Debug instrumentation is enabled one dimension at a time when possible. Each
run records the variant and options so a change in timing, power, memory
layout, or firmware behavior is not mistaken for a release regression.
KASAN, KCSAN, KFENCE, UBSAN, and similar instrumentation are lab experiments
unless the physical gate explicitly proves that the resulting image has the
same relevant behavior. Their successful build is never support evidence.

Every bring-up boot captures, with privacy redaction:

* boot artifact and DTB digests, config digest, board/SoC identity, firmware
  IDs, and exact kernel source SHA;
* complete ``dmesg`` or persistent-console output, including probe deferrals,
  firmware resets, DART/SART faults, thermal trips, and power transitions;
* pstore, tracefs/ftrace, devcoredump, GPU coredump, and relevant hwmon,
  thermal-zone, power-supply, cpufreq, and cpuidle snapshots;
* test workload, duration, ambient conditions, AC/battery state, attached
  peripherals, and operator/lab evidence ID.

Subsystem ownership map
-----------------------

The map assigns one accountable owner for each seam. A subsystem owner may
review another subsystem's interface, but may not silently absorb its release
or physical qualification responsibility.

.. list-table:: Kernel bring-up ownership
   :header-rows: 1
   :widths: 22 30 28 20

   * - Area
     - Kernel/DT surface
     - Primary gate
     - Coupled owner
   * - Board identity and DT
     - ``arch/arm64/boot/dts/apple/``; Apple bindings; DT Makefile
     - Exact compatible, ``dtbs``, ``dtbs_check``, no new warnings
     - Platform registry; lab
   * - Early SoC
     - arm64 platform, AIC, timers, CPU topology, clocks, pinctrl, GPIO, UART
     - Deterministic boot, all CPUs, interrupts, timers, console, no fault
     - Human boot-artifact owner; lab
   * - IOMMU and DMA
     - DART, SART, ADMAC, PCIe, USB, NVMe DMA paths
     - Mapped I/O, fault injection, storage and peripheral stress
     - Firmware owner; storage/USB owners
   * - Firmware IPC
     - Mailbox, RTKit, ASC, DockChannel, AOP, SEP, PMP
     - ABI match, timeout/reset/coredump behavior, no stale-buffer use
     - Platform manifest; human boot-artifact boundary
   * - Power and thermal
     - PMGR/PMP, SMC, hwmon, thermal, cpufreq, cpuidle, battery, fans
     - Trip behavior, suspend/resume, idle drain, sustained-load soak
     - Board power profile; lab
   * - Storage and ports
     - ANS/NVMe, PCIe, USB, USB-PD, ATC PHY, Ethernet/docks/SD where present
     - Cold/warm I/O, hotplug, DMA faults, suspend interaction
     - Board peripheral inventory; lab
   * - Display and GPU
     - DCP/DRM Apple, AGX DRM, backlight, panels, DP/HDMI/Thunderbolt
     - Modeset, hotplug, suspend, reset, GPU stress, compositor soak
     - Mesa owner; display firmware owner
   * - Audio, camera, and media
     - MCA/AOP audio, codecs, DCP audio, ISP, V4L2, hardware media
     - Capture/playback, protection, suspend, encode/decode and camera tests
     - Userspace media owner; lab
   * - Connectivity and input
     - Broadcom Wi-Fi/Bluetooth, NVRAM/regulatory data, HID, Z2, sensors
     - Cold boot, roam/reconnect, coexistence, input and wake tests
     - Firmware/data owner; lab
   * - Security and recovery
     - SEP/Touch ID boundary, secure boot inputs, panic/pstore, reset paths
     - No secret leakage, correct refusal, recovery and rollback evidence
     - Installer/release authority; human security reviewer
   * - Release and evidence
     - Config/source/DTB/firmware/Mesa tuple and qualification record
     - Signed manifest, reproducibility, immutable raw evidence, promotion
     - Platform coordinator; physical lab

The boot-artifact owner is human-only at this boundary. The kernel lane can
consume an artifact ID, DTB handoff, boot status, console, and memory map
contract, but cannot review or alter the artifact's source. Mesa owns the AGX
userspace/firmware-facing graphics release and conformance result; the kernel
lane owns the DRM/firmware interface it exposes.

Dependency graph and bring-up order
-----------------------------------

The prerequisite table below is mechanically transcribed from the pinned
Kconfig files. An enclosing ``if ARCH_APPLE || COMPILE_TEST`` is included in
the effective condition for the Apple SoC and PMGR symbols. ``depends on``
expressions are build prerequisites; ``select`` entries are not operational
probe-order edges. The config report must preserve the source path and line
for every row and must not add an inferred edge.

.. list-table:: Exact Kconfig prerequisite closure
   :header-rows: 1
   :widths: 23 34 28 15

   * - Symbol and source
     - Declared ``depends on``
     - Enclosing condition and ``select``
     - Failure
   * - ``APPLE_MAILBOX``
       ``drivers/soc/apple/Kconfig:16-19``
     - ``PM``; ``ARCH_APPLE || (64BIT && COMPILE_TEST)``
     - ``ARCH_APPLE || COMPILE_TEST``; no select
     - ``CONFIG_CLOSURE_FAIL``
   * - ``APPLE_RTKIT``
       ``drivers/soc/apple/Kconfig:36-39``
     - ``APPLE_MAILBOX``; ``ARCH_APPLE || COMPILE_TEST``
     - ``ARCH_APPLE || COMPILE_TEST``; no select
     - ``CONFIG_CLOSURE_FAIL``
   * - ``APPLE_SART``
       ``drivers/soc/apple/Kconfig:61-63``
     - ``ARCH_APPLE || COMPILE_TEST``
     - ``ARCH_APPLE || COMPILE_TEST``; no select
     - ``CONFIG_CLOSURE_FAIL``
   * - ``MFD_MACSMC``
       ``drivers/mfd/Kconfig:328-333``
     - ``ARCH_APPLE || COMPILE_TEST``; ``OF``; ``APPLE_RTKIT``
     - No enclosing Apple condition; selects ``MFD_CORE``
     - ``CONFIG_CLOSURE_FAIL``
   * - ``NVME_APPLE``
       ``drivers/nvme/host/Kconfig:125-130``
     - ``OF && BLOCK``; ``APPLE_RTKIT && APPLE_SART``;
       ``ARCH_APPLE || COMPILE_TEST``
     - No enclosing Apple condition; selects ``NVME_CORE``
     - ``CONFIG_CLOSURE_FAIL``
   * - ``APPLE_PMGR_MISC``
       ``drivers/soc/apple/Kconfig:28-30``
     - ``PM``
     - ``ARCH_APPLE || COMPILE_TEST``; no select
     - ``CONFIG_CLOSURE_FAIL``
   * - ``APPLE_PMGR_PWRSTATE``
       ``drivers/pmdomain/apple/Kconfig:5-11``
     - ``PM``
     - ``ARCH_APPLE || COMPILE_TEST``; selects ``REGMAP``, ``MFD_SYSCON``,
       ``PM_GENERIC_DOMAINS``, and ``RESET_CONTROLLER``
     - ``CONFIG_CLOSURE_FAIL``
   * - ``ARCH_APPLE`` platform selection
       ``arch/arm64/Kconfig.platforms:36-40``
     - No declared ``depends on``
     - ``select APPLE_AIC``; ``select APPLE_PMGR_PWRSTATE if PM``;
       ``select HAVE_SHARED_GPIOS``
     - ``CONFIG_CLOSURE_FAIL``

The effective report also records the framework symbols used by the selected
source: ``CONFIG_PM``, ``CONFIG_OF``, ``CONFIG_BLOCK``,
``CONFIG_ARCH_APPLE``, ``CONFIG_COMPILE_TEST``, ``CONFIG_MAILBOX``, and
``CONFIG_NVME_CORE``. ``CONFIG_MAILBOX`` is a framework/config-closure fact,
not an invented ``depends on`` edge for ``APPLE_MAILBOX``. A module or built-in
arrangement that fails the exact expressions, selected symbols, or required
framework closure is ``CONFIG_CLOSURE_FAIL``; deferred-probe recovery cannot
turn it into health.

There is a separate operational initialization order. It is evidence about
runtime readiness and is not a Kconfig dependency graph: boot handoff, AIC,
timers, CPU, console, clocks, pinctrl/GPIO, and DART precede subsystem probe;
the Apple mailbox is ready before RTKit; RTKit is ready before SMC and Apple
NVMe; SART is ready before Apple NVMe; and PMGR power-domain initialization is
reported as its own branch. Each consumer records the prerequisite result and
failure class. The document makes no additional dependency claim.

The operational order is deliberately conservative:

#. **Admission:** resolve exact board ID, SoC ID, DT compatible, firmware
   schema, config profile, and manifest tuple. Refuse ambiguity.
#. **Boot and memory:** verify the opaque boot artifact handoff, map memory,
   start the console, enumerate CPUs, initialize AIC and timers, and capture
   the first dmesg.
#. **Foundational fabric:** bring up clocks, pinctrl/GPIO, DART, reset/watchdog,
   and the Apple mailbox. Bring up RTKit only after the mailbox is ready. Bring
   up SART before Apple ANS/NVMe. Bring up PMGR_MISC and PMGR_PWRSTATE as a
   separate power-domain branch. Bring up SMC only after RTKit and never treat
   PMGR success as SMC success. Exercise faults and deferred probes.
#. **Root I/O:** qualify Apple ANS/NVMe only after both RTKit and SART are
   ready, then PCIe, USB, USB-PD/PHY,
   internal input, and network prerequisites. No desktop validation occurs
   before stable storage and recovery are proven.
#. **Power:** qualify cpufreq/cpuidle, sensors, thermal trips, fans, charging,
   battery reporting, shutdown, warm reboot, and suspend/resume.
#. **Display/GPU:** initialize DCP and display paths, then the matching AGX
   kernel/Mesa tuple. Test simple framebuffer handoff before accelerated
   modeset and isolate GPU failures from compositor failures.
#. **Audio/media/connectivity:** add MCA/AOP/DCP audio, camera/ISP, media
   engines, Wi-Fi/Bluetooth, and board peripherals one capability at a time.
#. **Soak and recovery:** run reboot, suspend, hotplug, thermal, power, crash,
   rollback, and repeated boot tests with raw evidence.
#. **Promotion:** submit the complete board matrix to the coordinator. A
   missing feature or unexplained warning keeps the board below FULL.

Each step has a known-good checkpoint. A later subsystem may not make a failed
earlier step appear healthy by masking logs, disabling a required node, or
using a generic fallback.

Promotion consumes only the equality-checked set of
``Trusted<PlatformManifest>.payload.document_id``,
``Trusted<PlatformManifest>.payload_digest``,
``Trusted<PlatformManifest>.payload.board_id``,
``Trusted<PlatformManifest>.payload.components``;
``Trusted<QualificationRecord>.payload.document_id``,
``Trusted<QualificationRecord>.payload.board.board_id``,
``Trusted<QualificationRecord>.payload.manifest.manifest_id``, and
``Trusted<QualificationRecord>.payload.manifest.manifest_digest``. It also
requires the exact ``Trusted<TrustContext>`` AuthorityRoleBinding for the
promotion action. A local board ID, report digest, or promotion boolean cannot
substitute for those equalities.

M1/M2 gold baseline (K-02)
---------------------------

The gold baseline is a reference tuple for building and debugging the
qualification machinery. It is not a claim that the current source or any
compiled kernel is supported. The current DT comments and root compatibles map
the following families and boards:

.. list-table:: Reference DT inventory
   :header-rows: 1
   :widths: 14 24 62

   * - Generation
     - SoC IDs in the current tree
     - Reference board set to acquire and qualify
   * - M1
     - ``t8103`` base; ``t6000`` Pro; ``t6001`` Max; ``t6002`` Ultra
     - Base: ``j274`` Mac mini, ``j293`` 13-inch MacBook Pro, ``j313`` MacBook
       Air, ``j456``/``j457`` iMac. Pro/Max/Ultra: ``j314s``, ``j316s``,
       ``j314c``, ``j316c``, ``j375c``, ``j375d``.
   * - M2
     - ``t8112`` base; ``t6020`` Pro; ``t6021`` Max; ``t6022`` Ultra
     - Base: ``j413``, ``j415``, ``j473``, ``j493``. Pro/Max: ``j414s``,
       ``j416s``, ``j474s``, ``j414c``, ``j416c``, ``j475c``. Ultra:
       ``j180d`` and ``j475d``.

The first physical gold set should contain at least two independently
serialised units for every materially distinct board/profile, including base
laptop, base desktop, Pro/Max laptop, Max desktop, and Ultra desktop profiles
where applicable. The coordinator must name the units, RAM/storage class,
firmware baseline, panel, dock, and recovery host before K-02 can be promoted.
The remaining rows above are still required for M1/M2 full-board
qualification; the gold set only reduces bring-up concurrency.

K-02 exits design only when one frozen M1/M2 tuple has:

* every listed DTB built and schema-checked for its reference class;
* release and bring-up config digests, kernel source SHA, DTB digest, and
  firmware schema recorded in one manifest;
* boot, root storage, network, input, display/GPU, audio, camera/media,
  suspend, thermal, charging, idle-power, update/rollback, and recovery rows
  executed on physical reference boards;
* matching Mesa artifacts and graphics evidence for every GPU/display row;
* no unresolved critical dmesg, DART/SART, RTKit, GPU, thermal, power, or
  audio-safety fault.

Canonical physical qualification profile
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Every board-level evidence package references the coordinator-owned
``physical-qualification/v1`` profile by ID and digest. A materially distinct
profile is any change in board SKU, SoC/die, RAM class, storage path, panel,
port topology, power/thermal design, or firmware profile. Each such profile
requires at least two independently serialised physical units. Evidence from
one unit, a family label, or a shared SoC does not extrapolate to another
unit/profile.

For each unit and applicable capability, the profile requires three clean
installs, at least 50 cold boots and 50 warm boots, five successful cycles for
every applicable port/peripheral path, and ten update/rollback cycles. Every
attempt is retained, including retries and recovery actions. Human-observed
criteria are mandatory for physical seating and connectors, display artifacts,
keyboard/trackpad/input behavior, audio safety, fan/noise behavior, charging,
thermal comfort, suspend/resume, recovery prompts, and visible corruption;
automation cannot mark these observations PASS.

The profile contains numeric thresholds, not qualitative substitutions. At a
minimum it records boot and health timeouts, idle-drain budget in mW, thermal
and surface-temperature limits in mC, fan response time in ms, battery
telemetry tolerance in percent, I/O error count, suspend/resume success rate,
and allowed warning count. Required safety and integrity thresholds are zero
kernel panics/oops, zero DART/SART faults, zero storage-integrity errors, zero
thermal emergencies, zero unsafe audio events, zero unexplained DT warnings,
and 100 percent success for clean installs, applicable port cycles, and
update/rollback cycles. Board-specific analog limits must be signed numeric
values in the profile; no universal thermal, fan, or idle-drain number may be
invented.

The signed capability matrix marks each row ``PRESENT``, ``ABSENT``, or
``NOT_APPLICABLE`` for that exact unit/profile and records the human authority
for the decision. ``PRESENT`` requires the complete test row; ``ABSENT`` and
``NOT_APPLICABLE`` require signed hardware evidence and cannot hide an
unavailable test. Unknown applicability, an unqualified capability, missing
serial identity, or missing numeric threshold is ``UNKNOWN`` and blocks
promotion. Future silicon without two qualified units and a profile is
``UNKNOWN`` regardless of a compatible-looking SoC or product name.

The evidence package signs the profile ID/digest, unit serials, manifest and
all artifact digests, toolchain, raw logs, human observation forms, threshold
results, failed attempts, and operator/time records. K-02 and every later lane
must reference this exact profile; no generation or family extrapolation is
allowed.

M3, M4, A18, M5, and M6 lanes (K-03 through K-06)
--------------------------------------------------

Every newer lane repeats the K-02 gates and references its own
``physical-qualification/v1`` profile. It may reuse a narrowly identified
test row only when the manifest and coordinator ruling prove the exact board,
serialised unit class, firmware ABI, DT binding, driver behavior, power
profile, display/audio topology, Mesa tuple, and test environment are
identical. A passing family build or sibling board never supplies physical
evidence; there is no family extrapolation.

.. list-table:: Generation lane plan
   :header-rows: 1
   :widths: 15 28 35 22

   * - Lane
     - Current source evidence
     - Required work
     - Promotion condition
   * - K-03 M3
     - ``t8122`` base; ``t6030`` Pro; ``t6031`` Max; ``t6032`` Ultra;
       ``t6034`` 14-core Max variant
     - Validate shared M3 blocks separately from die count, memory channels,
       display, USB-PD, audio, camera, and board integration. Qualify every
       current M3 board DT and firmware profile.
     - All applicable M3 rows pass on physical boards; no SoC-recognition-only
       promotion.
   * - K-04 M4
     - ``t8132`` is present for M4 base boards in the current tree. The
       current snapshot does not establish M4 Pro/Max DT coverage.
     - Treat M4 Pro/Max as intake and unknown until exact IDs, bindings,
       firmware, DTs, and boards exist. Do not copy M3 assumptions into M4.
     - Base and each higher M4 variant are separately qualified, then promoted
       by exact board record.
   * - K-05 A18/M5
     - No A18 Pro MacBook Neo or M5 SoC/board family is identified by the
       current DT inventory.
     - Acquire hardware, capture Apple IDs and DT facts, define bindings and
       firmware contracts, then bring up from admission through physical soak.
       A new compatible is evidence-backed, not guessed from the product name.
     - Shipping hardware, exact registry records, complete tuple, and all
       applicable physical rows.
   * - K-06 M6
     - M6 is a program intake target, not a current kernel support claim.
     - Start only after shipping hardware and recovery capability are acquired;
       repeat discovery, DT, firmware, driver, Mesa, and lab gates unchanged.
     - Coordinator promotion after physical evidence. Announcement alone has
       no support effect.

The lane dependency is ``K-01 -> K-02 -> K-03 -> K-04 -> K-05 -> K-06`` for
release process maturity, while subsystem work can be parallelized inside a
lane after the foundational fabric is stable. A new lane does not reopen an
older board's evidence unless a shared driver, binding, firmware ABI, or Mesa
change invalidates it.

Board promotion and evidence states
-----------------------------------

The public registry uses narrow, monotonic evidence states. The coordinator,
not a kernel branch or CI job, changes the state.

.. list-table:: Board states
   :header-rows: 1
   :widths: 20 55 25

   * - State
     - Required meaning
     - Public implication
   * - DETECTED
     - Exact identity is known, but boot or capability evidence is absent.
     - Detection only.
   * - BRINGUP
     - A DT/config/firmware tuple is being debugged; failures and residuals are
       expected and recorded.
     - Not supported.
   * - EXPERIMENTAL
     - Repeatable boot and a declared subset work on identified hardware, with
       explicit missing rows and no FULL claim.
     - Limited test use only.
   * - DAILY_DRIVER
     - The coordinator has accepted a useful, repeatable subset and published
       residuals, but one or more applicable FULL rows remain.
     - Not fully compatible.
   * - FULL
     - Every applicable program capability and cross-repository gate passes on
       the exact board with immutable evidence.
     - The only state satisfying the program's full-compatibility mission.

Promotion requires board ID, SoC ID, DTB digest, config digest, kernel source
SHA, firmware and boot-artifact IDs, Mesa artifact, userspace image, test
profile, raw evidence IDs, operator, timestamps, failures, residuals, and
coordinator approval. A family-level green build never promotes all boards in
that family. A board remains below FULL while a physical feature is untested,
unsafe, silently disabled, or coupled to an unqualified artifact.

Kernel boot health: accepted boot-health/v1
-------------------------------------------

The kernel reports only the frozen ``boot-health/v1``
``Trusted<BootHealthCore>`` contract. It is
a signed core and contains no success marker. The exact core payload paths are
``Trusted<BootHealthCore>.payload.board_id``,
``Trusted<BootHealthCore>.payload.manifest_document_id``,
``Trusted<BootHealthCore>.payload.manifest_payload_digest``,
``Trusted<BootHealthCore>.payload.slot``,
``Trusted<BootHealthCore>.payload.generation``,
``Trusted<BootHealthCore>.payload.lineage``,
``Trusted<BootHealthCore>.payload.counter``,
``Trusted<BootHealthCore>.payload.source_generation``,
``Trusted<BootHealthCore>.payload.required_check_policy``,
``Trusted<BootHealthCore>.payload.rollback_set``,
``Trusted<BootHealthCore>.payload.checks``,
``Trusted<BootHealthCore>.payload.failure_class``,
``Trusted<BootHealthCore>.payload.retry_target``,
``Trusted<BootHealthCore>.payload.fallback_target``,
``Trusted<BootHealthCore>.payload.started_at``, and
``Trusted<BootHealthCore>.payload.completed_at``. The core's manifest document
ID and payload digest must equal the selected ``Trusted<PlatformManifest>``
envelope.

The required checks and failed-attempt limit are not K-01 constants. They are
resolved from the exact accepted manifest paths
``Trusted<PlatformManifest>.payload.components.boot_stack.boot_health.required_checks``
and
``Trusted<PlatformManifest>.payload.components.boot_stack.boot_health.max_failed_attempts``.
The rollback set is resolved from
``Trusted<PlatformManifest>.payload.rollback.set``. The core copies the
resulting policy IDs and digests into
``Trusted<BootHealthCore>.payload.required_check_policy`` and
``Trusted<BootHealthCore>.payload.rollback_set``; it does not edit the policy.
Every required check entry has its manifest-defined ID, predicate, timeout,
and policy digest. The observed ``Trusted<BootHealthCore>.payload.checks`` list
must contain exactly those entries, with one of the manifest-enumerated results
``PASS``, ``FAIL``, or ``TIMEOUT``. Missing, extra, duplicate, reordered, or
unknown entries fail the record.

The attempt counter is a persistent unsigned value in the manifest-defined
encoding and lineage. It strictly increases for the selected board, manifest
document ID, slot, generation, and source generation. A decrease, reset,
unknown epoch, invalid jump, wrap, or counter-policy mismatch is
``COUNTER_INVALID``. A recovery reset requires a separately authenticated
transition naming the old counter, new counter, reason, recovery target, and
source generation. The retry decision compares failed attempts with the exact
``max_failed_attempts`` value from the selected manifest; it never substitutes
a local numeric limit. The retry target must preserve board, manifest document
ID, manifest payload digest, slot, generation, lineage, source generation,
artifact set, required-check policy, and rollback set. When the manifest limit
is exhausted, only its complete signed fallback target or recovery path may be
selected.

The optional success object is a separate authenticated
``boot-success-mark/v1`` ``Trusted<BootSuccessMark>`` payload. Its exact
binding path to the core is ``Trusted<BootSuccessMark>.payload.core_digest``
(the verifier-computed ``D_core``), plus
``Trusted<BootSuccessMark>.payload.board_id``,
``Trusted<BootSuccessMark>.payload.manifest_document_id``,
``Trusted<BootSuccessMark>.payload.manifest_payload_digest``,
``Trusted<BootSuccessMark>.payload.slot``,
``Trusted<BootSuccessMark>.payload.generation``,
``Trusted<BootSuccessMark>.payload.lineage``,
``Trusted<BootSuccessMark>.payload.counter``,
``Trusted<BootSuccessMark>.payload.source_generation``,
``Trusted<BootSuccessMark>.payload.required_check_policy_digest``, and
``Trusted<BootSuccessMark>.payload.rollback_set_digest``. It contains no
authority to change those values. The verifier recomputes
``Trusted<BootSuccessMark>.payload.required_check_policy_digest`` from the
exact
``Trusted<PlatformManifest>.payload.components.boot_stack.boot_health.required_checks``
policy and recomputes
``Trusted<BootSuccessMark>.payload.rollback_set_digest`` from the exact
``Trusted<PlatformManifest>.payload.rollback.set``. The verifier also
recomputes ``D_core`` from the canonical ``Trusted<BootHealthCore>`` payload
before comparing it with
``Trusted<BootSuccessMark>.payload.core_digest``. A success mark is valid only
after every manifest-derived required check is ``PASS`` and the core and mark
signatures, expiry, and replay identity verify. Embedding a success marker in
the core, trusting a boolean, or accepting a mark with any mismatched binding
is ``BOOT_HEALTH_INTEGRITY_FAIL``.

The failure classes are ``IDENTITY``, ``DTB_INTEGRITY``, ``CONFIG``,
``FIRMWARE_ABI``, ``STORAGE``, ``HEALTH_TIMEOUT``, ``KERNEL_FATAL``,
``INTEGRITY``, and ``INFRASTRUCTURE``. Unknown classes, schema versions,
counter encodings, reset records, policy IDs, check IDs, mark presence, core
digest, or marker verification state are failures, not warnings. Every failed
attempt, retry, fallback, and recovery
action is retained. The kernel owns observed results and the core report; the
health authority signs the optional mark; the boot/recovery authority selects
the manifest-defined slot; and the coordinator owns acceptance. K-01 cannot
claim boot health when the accepted schemas, policy, persistence, key, exact
bindings, or recovery target are unavailable.

Bisectability and regression handling
-------------------------------------

The source queue is designed for ``git bisect`` across both common and
board-specific regressions:

* keep a linear, signed or otherwise auditable sequence from the recorded
  upstream base; avoid merge bubbles in the release queue;
* group a binding, DT, and driver change only when each intermediate commit
  would be invalid, and explain that dependency in the commit message;
* do not mix formatting, unrelated cleanup, generated files, or a new board
  with a firmware or GPU rewrite;
* make every commit buildable for the intended config, and every DT commit
  at least statically checkable. If a physical test requires a later boot
  artifact, record the first testable tuple rather than pretending the earlier
  commit was boot-qualified;
* preserve old configs and DT compatibles unless an explicit ABI migration and
  rollback plan exists;
* record per-commit test results, dmesg classification, and board coverage so
  ``git bisect run`` can distinguish infrastructure failure from a real kernel
  regression;
* on a regression, freeze the last-known-good manifest, bisect the kernel and
  DT queue independently where possible, then rerun the smallest affected
  physical matrix before rebuilding the full candidate.

Every queue refresh publishes ``git range-diff`` and the old/new artifact
matrix. A suspected regression is not closed by reverting a symptom if the
revert changes a firmware ABI, hides a dmesg fault, or leaves a capability
unqualified.

Testing and CI design
---------------------

The kernel repository snapshot supplies generic Linux test infrastructure but
no repo-local workflow. Omarchy CI should live in the platform/release
orchestration layer and invoke pinned kernel sources. It must distinguish
host-only, target-emulated, and physical results.

Named ownership and approvals
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Owner and CI authority resolves only through the common closed
``AuthorityRoleBinding`` records in ``Trusted<TrustContext>`` supplied by
F-03. K-01 neither defines nor consumes an owners registry. For every gate,
the resolver selects the exact binding whose role, scope, key, policy, expiry,
and signature authorize the exact manifest document ID, board ID, profile,
artifact set, report, or promotion action.

.. list-table:: AuthorityRoleBinding checks
   :header-rows: 1
   :widths: 24 40 36

   * - Check
     - Canonical binding member
     - Fail-closed rule
   * - Role
     - ``Trusted<TrustContext>.payload.authority_bindings[].role``
     - The exact enum is ``linux-queue``, ``kernel-config``, ``dt-binding``,
       ``board-identity``, ``firmware-abi``, ``boot-artifact``, ``boot-health``,
       ``ci-toolchain``, ``docs``, ``physical-lab``, ``recovery``,
       ``release``, or ``coordinator``; a team label or unassigned role is not
       an authority.
   * - Scope
     - ``Trusted<TrustContext>.payload.authority_bindings[].scope.manifest_document_id``;
       ``Trusted<TrustContext>.payload.authority_bindings[].scope.board_id``;
       ``Trusted<TrustContext>.payload.authority_bindings[].scope.profile_id``;
       ``Trusted<TrustContext>.payload.authority_bindings[].scope.artifact_id``;
       ``Trusted<TrustContext>.payload.authority_bindings[].scope.report_id``;
       ``Trusted<TrustContext>.payload.authority_bindings[].scope.gate_action``
     - Every scope member must equal the requested manifest document ID,
       board/profile, artifact, report, and gate action; an omitted or wildcard
       member is unknown and rejects the binding.
   * - Key
     - ``Trusted<TrustContext>.payload.authority_bindings[].key_id``;
       ``Trusted<TrustContext>.payload.authority_bindings[].signature``
     - Key must be active, pinned by the trust context, and verify the signed
       approval; an unknown or expired key rejects the gate.
   * - Policy
     - ``Trusted<TrustContext>.payload.authority_bindings[].policy_id``;
       ``Trusted<TrustContext>.payload.authority_bindings[].policy_digest``;
       ``Trusted<TrustContext>.payload.authority_bindings[].threshold``;
       ``Trusted<TrustContext>.payload.authority_bindings[].required_roles[]``;
       ``Trusted<TrustContext>.payload.authority_bindings[].separation_groups[]``
     - Policy ID and digest must equal the F-03 threshold policy for that gate;
       the threshold, required roles, and separation groups are read from the
       same signed TrustContext, never from the caller.
   * - Binding validity
     - ``Trusted<TrustContext>.payload.authority_bindings[].subject``;
       ``Trusted<TrustContext>.payload.authority_bindings[].expires_at``;
       ``Trusted<TrustContext>.payload.authority_bindings[].replay_identity``;
       ``Trusted<TrustContext>.payload.authority_bindings[].signature``
     - Subject, expiry, replay identity, and signature must verify under the
       closed trust context.
   * - Threshold and separation
     - ``Trusted<TrustContext>.payload.authority_bindings[].approval_id``;
       ``Trusted<OwnerApproval>.payload.approval_id``;
       ``Trusted<OwnerApproval>.payload.manifest_document_id``;
       ``Trusted<OwnerApproval>.payload.manifest_payload_digest``
     - Required distinct roles and approval threshold must be met; the same
       key cannot satisfy a separation-of-duties requirement. ``approval_id``
       must resolve to the separately authenticated ``owner-approval/v1``
       payload and its manifest document ID and payload digest must match the
       binding scope.

K-01 requires the F-03 bindings for Linux queue, config, DT binding, board
identity, firmware ABI, boot artifact, boot health, CI/toolchain, docs, lab,
recovery, and coordinator promotion. The queue/config actions require the
Linux and release roles; DT actions require DT and board-identity roles;
firmware and boot-health actions require firmware, health, and recovery roles;
documentation actions require the docs role; and physical promotion requires
lab and coordinator roles. Missing, ambiguous, out-of-scope, expired,
unverified, or insufficient bindings produce ``OWNER_BLOCK`` and keep K-01 in
``FAIL_CLOSED``.

Static and build gates
~~~~~~~~~~~~~~~~~~~~~~

The toolchain is pinned by
``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock``;
its recipe identity and digest are not inferred from a host. The signed lock
names the compiler and exact version/digest (GCC or Clang/LLVM), Rust compiler
and LLVM when Rust is used, ``dtc`` version/digest, dt-schema version/digest,
Python version, Sphinx version/digest, host/container image digest, locale,
working-directory policy, and relevant make variables. A version range,
unpinned package install, or host fallback creates a new tuple or produces
``TOOLING_BLOCK``.

For every queue tip and relevant commit, CI executes the complete argv arrays
at the following canonical paths, byte-for-byte and without shell expansion:

.. list-table:: Lock-derived K-01 command arrays
   :header-rows: 1
   :widths: 25 55 20

   * - Gate
     - Canonical argv path
     - Required target
   * - Config closure
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.olddefconfig``
     - ``olddefconfig``
   * - DTB build
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.dtbs``
     - ``dtbs``
   * - DTB schema check
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.dtbs_check``
     - ``dtbs_check``
   * - AGX binding check
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.dt_binding_check``
     - ``dt_binding_check``
   * - Warning build
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.warning_build``
     - ``W=1`` build
   * - Patch validation
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.checkpatch``
     - ``scripts/checkpatch.pl``
   * - Documentation
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.docs``
     - documentation and RST/toctree check
   * - Reproducibility
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock.command_arrays.reproducible_build``
     - clean rebuild comparison

Each referenced value is a non-empty closed JSON array of complete argument
strings. The array itself supplies the cross-compiler assignment, schema-file
assignment, Sphinx executable, output policy, and every other variable; no
caller may append arguments or provide a host value. The config array consumes
the locked fragment; the DT arrays cover every Apple DTB in the manifest's
explicit inventory, and the binding array names
``Documentation/devicetree/bindings/gpu/apple,agx.yaml``. The runner verifies
the normalized config digest, required built-in/module decisions, initramfs
closure, sorted JSON warning output, and the pinned zero-warning or reviewed
warning baseline.

The report lock at
``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock`` pins
the report contract. Its closed ``schemas[]`` list is exactly
``linux-k01-config-report/v1``, ``linux-k01-dtb-report/v1``,
``linux-k01-provenance-report/v1``, ``linux-k01-docs-report/v1``,
``linux-k01-reproducibility-report/v1``, and
``linux-k01-boot-health-report/v1``, ``linux-k01-mesa-report/v1``, and
``linux-k01-qualification-report/v1``. Its ``status_values`` list is exactly
``PASS``, ``FAIL``, ``TOOLING_BLOCK``, and ``UNKNOWN``; skipped, advisory,
partial, or locally invented statuses do not pass a gate. Its
``retention`` is append-only, access-audited retention for the life of the
support program plus seven years. Its ``expected_artifacts[]`` set is exactly
``kernel_image``, ``kernel_modules``, ``initramfs``, ``dtb_set``,
``binding_schemas``, ``platform_manifest``, ``signatures``, ``reports``,
``raw_logs``, ``reproducibility_diff``, ``firmware_bundle``, ``boot_artifacts``,
``mesa_artifacts``, ``userspace``, ``rollback_set``, and
``dtb_mutation_envelopes``.

Each required status check has one report schema and one canonical result
path: ``manifest-contract`` and ``provenance-and-signatures`` use
``linux-k01-provenance-report/v1``; ``queue-reproducibility`` uses
``linux-k01-reproducibility-report/v1``; ``config-closure`` uses
``linux-k01-config-report/v1``; ``dt-binding-baseline`` and
``dtbs-inventory`` use ``linux-k01-dtb-report/v1``;
``docs-rst-toctree`` and ``conflict-markers`` use
``linux-k01-docs-report/v1``; ``firmware-abi-contract`` uses
``linux-k01-provenance-report/v1``; ``boot-health-contract`` uses
``linux-k01-boot-health-report/v1``; and ``mesa-coupling`` uses
``linux-k01-mesa-report/v1``. ``physical-qualification`` uses the signed
``qualification-record/v1`` payload and its
``Trusted<QualificationRecord>.payload.test_results[]`` rather than a local
report schema. Every result path is closed and the status is one of the four
values above.

Every command publishes the selected report schema, status, source commit,
``Trusted<PlatformManifest>.payload.document_id``,
``Trusted<PlatformManifest>.payload_digest``, board/profile scope, the exact
argv array and environment lock, toolchain recipe digest, start/end time,
exit code, stdout/stderr digests, input/output artifact digests, warning
baseline ID, and AuthorityRoleBinding approval IDs. A non-zero command,
missing report or expected artifact, digest mismatch, unpinned tool, unknown
status, owner/policy gap, unexplained warning, retention failure, conflict
marker, or inability to reproduce the locked output blocks its status and all
dependent promotion. An unavailable tool is ``TOOLING_BLOCK``, never PASS.
Compile-only, QEMU, or desktop results never make a support claim; QEMU is
useful for generic arm64 regression tests but does not emulate Apple hardware.

The required status checks are ``manifest-contract``,
``queue-reproducibility``, ``config-closure``, ``dt-binding-baseline``,
``dtbs-inventory``, ``docs-rst-toctree``, ``firmware-abi-contract``,
``boot-health-contract``, ``provenance-and-signatures``, and
``conflict-markers``. Release candidates additionally require
``physical-qualification`` and ``mesa-coupling``. A non-zero command,
missing report/artifact, unpinned tool, unknown result, owner/approval gap,
unexplained warning, or retention failure blocks the corresponding status and
all dependent promotion. Advisory or skipped jobs never satisfy a required
check; unavailable host tooling is reported as ``TOOLING_BLOCK``.

KUnit and kselftest
~~~~~~~~~~~~~~~~~~~

KUnit tests are appropriate for deterministic kernel logic that can be isolated
from Apple hardware: compatible/manifest admission helpers, ABI/version
parsers, DT property validation helpers, firmware state machines, reset/error
transitions, buffer ownership checks, and dmesg classification helpers if such
code is later added. Tests must include unknown versions, truncated input,
duplicate fields, invalid alignment, timeout, reset, and partial-probe cases.
They do not prove that a real co-processor or board behaves correctly.

Relevant kselftest suites should run on a booted target as applicable,
including drivers, power, networking, filesystems, timers, RTC, input, and
userspace ABI tests. The target job records skipped tests with a reason tied to
the board capability profile. A skipped test is not a pass, and a test that
cannot observe physical hardware is not a substitute for the physical row.

LAVA-style physical jobs
~~~~~~~~~~~~~~~~~~~~~~~~

The physical lab runner should use a LAVA-style job contract even if its first
implementation is not LAVA. Each job names:

* immutable board inventory ID, exact model/SoC, lab fixture, recovery host,
  power controller, serial/log channel, peripherals, and firmware baseline;
* ``Trusted<PlatformManifest>.payload.document_id`` and
  ``Trusted<PlatformManifest>.payload_digest``, kernel/config/DTB/Mesa/
  boot-artifact digests, ``Trusted<PlatformManifest>.payload.components.boot_stack.slots``,
  ``Trusted<PlatformManifest>.payload.components.boot_stack.boot_health``, boot
  arguments, expected root device, timeout, retry policy, and
  ``Trusted<PlatformManifest>.payload.rollback.last_known_good``;
* ``Trusted<QualificationRecord>.payload.document_id``,
  ``Trusted<QualificationRecord>.payload.board.board_id``,
  ``Trusted<QualificationRecord>.payload.manifest.manifest_id``,
  ``Trusted<QualificationRecord>.payload.manifest.manifest_digest``,
  ``Trusted<QualificationRecord>.payload.qualification_profile_id``, and
  ``Trusted<QualificationRecord>.payload.evidence[]``;
* ordered actions: cold boot, warm reboot, shutdown, suspend/resume, workload,
  hotplug, network reconnect, display/audio/media checks, thermal soak, and
  evidence capture;
* pass/fail predicates, allowed skips, fail-closed teardown, raw log IDs, and
  operator-independent result signing.

The runner accepts the job only when the manifest board ID and payload digest,
qualification board ID and manifest digest, boot-health context, and DTB
mutation envelope board/manifest/firmware bindings are pairwise equal to the
same candidate tuple. A local report or handoff field that disagrees is a
cross-document failure, not a second source of truth.

Retries may diagnose a flaky fixture but cannot erase a failed attempt. A power
cycle or recovery action is itself evidence. A job that loses its board,
console, or manifest identity is INFRASTRUCTURE-FAIL and cannot produce a
qualification PASS.

Dmesg, thermal, and power acceptance
-------------------------------------

Dmesg acceptance is classified, not reduced to a grep for the word ``error``.
The classifier records the exact line, subsystem, first occurrence, repeat
count, and board/manifest context. The following are automatic blockers until
reviewed and reproduced: ``BUG``, ``Oops``, ``WARNING``, ``Call Trace``, kernel
panic, hung task, unexplained probe failure for a required node, DART/SART
fault, RTKit/co-processor crash, GPU reset/fault, storage corruption, thermal
emergency, unsafe audio protection state, or repeated suspend/resume failure.
Benign known lines require an exact, reviewed allowlist entry with an owner and
retirement condition; a broad ``ignore warnings`` rule is forbidden.

Thermal acceptance is board-profile based. For every thermal zone and fan:

* identify the sensor, unit, trip points, cooling device, and source DT node;
* prove plausible readings at idle, AC load, battery load, and ambient change;
* exercise sustained CPU/GPU/I/O load through the intended performance and
  cooling states, verify throttling before emergency behavior, and verify fans
  respond and stop safely;
* run a long soak with no thermal emergency, runaway oscillation, sensor loss,
  unexplained power-domain fault, or unsafe surface/speaker condition;
* compare temperature, frequency, fan, and power traces with the board's
  signed qualification profile. Universal temperatures or fan curves must not
  be invented for boards whose mechanics differ.

Power acceptance records cpufreq transitions, cpuidle residency, wakeups,
power-domain state, charger/AC transitions, battery current and capacity,
shutdown/reboot, and suspend/resume. It requires stable idle over a defined
soak, no unexplained wake source, no battery-report discontinuity, correct
charge limiting where present, and a board-specific idle-drain budget approved
in the qualification profile. A desktop that stays on is not evidence of safe
power management. A lower idle-drain number cannot waive a failed suspend,
thermal, charging, or recovery row.

Coupling to the platform manifest, boot artifacts, and Mesa
-----------------------------------------------------------

Once F-02 is accepted, the kernel lane will consume the canonical
``platform-manifest/v1`` and publish the fields needed by it. At this reviewed
tip no such accepted manifest is claimed. The minimum kernel-side tuple is read
from the canonical component records and typed relations below; no abbreviated
member name is an alternate authority:

.. list-table:: Cross-repository tuple
   :header-rows: 1
   :widths: 25 45 30

   * - Member
     - Required coupling
     - Failure behavior
   * - Kernel
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.source_patch_queue``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.artifacts[]``
     - Refuse promotion if source/config/artifact provenance is incomplete.
   * - Device tree
     - ``Trusted<PlatformManifest>.payload.components.dtb_set.source``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.binding_schema``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.abi``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.identity_match``
     - Refuse boot-health success when identity or ABI is inconsistent.
   * - Firmware
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.bundle``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.tuning``;
       ``Trusted<PlatformManifest>.payload.compatibility_relations[]``
     - Required client failure blocks capability and promotion.
   * - Human boot artifact
     - ``Trusted<PlatformManifest>.payload.components.boot_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.abi``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.slots``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.boot_health``
     - Kernel reports the observed handoff; it does not inspect the source
       repository or substitute an unqualified artifact.
   * - Mesa
     - ``Trusted<PlatformManifest>.payload.components.mesa_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.abi``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.compatibility_relations[]``
     - No GPU/display PASS with an arbitrary Mesa build.
   * - Userspace/initramfs
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.abi``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_lock.initramfs_module_closure[]``;
       ``Trusted<PlatformManifest>.payload.package_set``;
       ``Trusted<PlatformManifest>.payload.artifacts[]``
     - Missing required userspace/firmware is an honest failure, not a warning.

The boot-artifact handoff must expose enough authenticated data for the kernel
to verify the exact
``Trusted<PlatformManifest>.payload.identity_match`` and
``Trusted<PlatformManifest>.payload.components.dtb_set.source`` tuple,
``Trusted<DtbMutationEnvelope>.payload.dtb.before_digest``,
``Trusted<DtbMutationEnvelope>.payload.dtb.after_digest``,
``Trusted<DtbMutationEnvelope>.payload.manifest_document_id``,
``Trusted<DtbMutationEnvelope>.payload.manifest_payload_digest``,
``Trusted<DtbMutationEnvelope>.payload.firmware.schema``, and the manifest's
``Trusted<PlatformManifest>.payload.components.boot_stack.boot_health`` slot
context. Memory reservations, boot arguments, console/debug transport, and
DTB location are evidence fields under the same authenticated handoff; they
cannot override the manifest or mutation envelope. The details and
implementation remain with the qualified human owner.

For graphics, the kernel AGX driver and Mesa must agree on GPU generation,
firmware compatibility, shared memory structures, reset behavior, page size,
DRM uAPI, and calibration data. The current AGX Kconfig requires Rust, IOMMU
support, and 16 KiB pages, while its binding distinguishes G13/G14 variants;
this is a concrete reason not to infer M4, A18, M5, or M6 graphics support from
an existing ``DRM_ASAHI`` build. The display DCP path, accelerated AGX path,
and Mesa conformance result are separate rows even when one desktop session
uses all three.

Unknowns and coordinator questions
-----------------------------------

The following are intentionally unresolved. They are blockers to implementation
or promotion, not invitations to guess.

Explicit unknowns
~~~~~~~~~~~~~~~~~

* The exact A18 Pro, M5, and M6 Apple SoC IDs, target types, board IDs, DT
  compatible strings, memory topologies, and firmware schemas are not in the
  current kernel DT inventory.
* The current DT inventory has M4 ``t8132`` base boards, but this snapshot does
  not establish M4 Pro/Max DT coverage. It also does not establish A18/M5/M6
  coverage.
* The current AGX binding names G13/G14 variants. The ABI and driver path for
  later GPU generations, including whether a different graphics driver is
  required, is not decided here.
* The accepted F-02 document, signed context, concrete component records, and
  evidence are not present in this design lane, but their required paths and
  relation vocabulary are frozen above. Any absent queue, config, DTB,
  firmware, boot, Mesa, artifact, ABI, or rollback record remains a
  fail-closed candidate gap.
* The relationship between the DT's ``apple,firmware-abi`` property, the
  driver's ``apple,firmware-compat`` property, bootloader overwrites, and the
  release firmware record is an implementation evidence gap. It must be
  represented by the accepted typed entries in
  ``Trusted<PlatformManifest>.payload.compatibility_relations[]``; an unknown
  relation is quarantined rather than frozen later or inferred.
* Board-specific thermal limits, fan curves, idle-drain budgets, speaker
  protection measurements, and acceptable dmesg allowlists require physical
  lab baselines. They cannot be safely generalized from SoC family names.
* The current linux repository has no local CI workflow. Runner ownership,
  cross-compilers, Rust/LLVM toolchain versions, schema packages, artifact
  retention, and physical lab scheduling are not yet assigned.
* The human-only boot-artifact owner must publish the opaque artifact schema,
  provenance and health-status fields that the kernel/release boundary can
  validate without repository inspection.

Coordinator questions
~~~~~~~~~~~~~~~~~~~~~

The coordinator should answer these before K-01 implementation begins:

* Which accepted F-02 signed context artifact and report publication location
  will supply the already-frozen platform-manifest paths to K-01 consumers?
* Which human maintainers own boot artifacts, firmware bundles, security/SEP,
  power/thermal, display/GPU, audio/media, connectivity, and the physical lab?
* Which serialised M1/M2 boards are the gold fixtures, what RAM/storage/panel
  classes are mandatory, and which recovery/power/serial equipment is ready?
* What are the approved board-profile budgets for idle drain, thermal soak,
  fan response, suspend cycles, boot loops, audio safety, and dmesg blockers?
* What firmware sources, signatures, licenses, schemas, and offline recovery
  rules are allowed for each client? May any firmware be redistributed, or
  must the installer acquire it from an Apple-authoritative source?
* What is the human-owned boot-artifact handoff contract, including DTB
  selection, memory reservations, console, slot health, and rollback status?
* Which Mesa branch/build is the first M1/M2 reference, and what exact kernel
  DRM/AGX ABI and firmware compatibility report is required for graphics PASS?
* Which toolchain and builders are reproducibility authorities for C, Rust,
  DTB, documentation, and signed release artifacts? Which results are merely
  advisory on macOS hosts?
* What is the upstream-sync cadence, stable base policy, queue retirement
  policy, and escalation path when an upstream change invalidates physical
  evidence?
* What evidence retention, redaction, signing, and public-ledger policy applies
  to serial logs, board identifiers, firmware versions, crash dumps, thermal
  traces, and battery data?
* When shipping hardware exists for A18, M5, or M6, who opens intake and which
  coordinator ruling authorizes a new DT family or binding?

K-01 through K-06 exit ledger
-----------------------------

This design maps the requested work into reviewable exits. These are proposed
acceptance gates, not completed work.

.. list-table:: K-slice design exits
   :header-rows: 1
   :widths: 12 38 50

   * - Slice
     - Design exit
     - Evidence required before coordinator review
   * - K-01
     - Upstream-sync policy, minimal queue, config/ABI contract, DT layout,
       firmware boundary, debug profiles, ownership, dependencies, boot-health
       contract, and CI/docs contract approved.
     - Accepted F-02 manifest and F-03 ``Trusted<TrustContext>``
       ``AuthorityRoleBinding`` context, exact source/config/DTB/firmware/
       boot/Mesa tuple report, AGX warning census resolution, named approvals,
       reproducible reconstruction, and no untracked interface authority.
   * - K-02
     - M1/M2 gold tuple is repeatable and bisectable.
     - Physical board records, complete applicable test rows, exact tuple
       digests, dmesg/thermal/power evidence, Mesa result, recovery evidence,
       and residual list.
   * - K-03
     - M3 boards use the same gates with generation-specific evidence.
     - Exact M3 board matrix, DT/binding/firmware review, physical soak and
       graphics/media/peripheral qualification.
   * - K-04
     - M4 base, Pro, and Max are separately admitted and qualified.
     - Shipping hardware and exact identities for every variant; no inherited
       M3 assumptions; complete physical records.
   * - K-05
     - A18 Pro and M5 intake, bring-up, and qualification are operational.
     - Hardware acquisition, exact DT/firmware/graphics contracts, and full
       board-level evidence for each applicable feature.
   * - K-06
     - M6 follows the unchanged intake and qualification process.
     - Shipping hardware, recovery capability, manifest tuple, and immutable
       physical evidence; marketing announcements do not substitute.

Until those evidence packages exist, every K-slice remains TODO or in design.
This file records the plan only; it does not promote a board, kernel, DTB,
firmware, Mesa build, or release.
