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

F-02 is the platform-manifest acceptance slice. The supplied F-02 snapshot is
rejected and frozen, not accepted authority. K-01 references its provisional
typed paths as an external dependency and does not define a parallel manifest,
registry, owner registry, or digest authority. Until an accepted, signed F-02
schema revision and its signed context exist, K-01 is ``FAIL_CLOSED`` and
cannot be reported as PASS, DONE, or implementation-ready.

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

.. list-table:: Provisional F-02 platform-manifest/v1 paths referenced by K-01
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
   * - Manifest target and board identity
     - ``Trusted<PlatformManifest>.payload.board_registry_digest``;
       ``Trusted<PlatformManifest>.payload.board_targets[]``;
       ``Trusted<BoardRegistry>.payload.boards[].board_id``;
       ``Trusted<BoardRegistry>.payload.boards[].identity_match``;
       ``Trusted<BoardRegistry>.payload.boards[].soc.soc_id``;
       ``Trusted<BoardRegistry>.payload.boards[].firmware.bundle_id``;
       ``Trusted<BoardRegistry>.payload.boards[].firmware.firmware_schema_id``
     - The registry digest resolves each explicit target to one complete
       registry record; the manifest has no local ``board_id`` or
       ``identity_match`` authority.
   * - Linux identity predicates
     - ``Trusted<BoardRegistry>.payload.boards[].identity_match.macos``;
       ``Trusted<BoardRegistry>.payload.boards[].identity_match.linux.compatible[]``;
       ``Trusted<BoardRegistry>.payload.boards[].identity_match.linux.model``
     - The complete closed registry predicate is compared; no ordered tuple,
       SoC token, product name, firmware token, or provenance inference fills a
       missing member.
   * - Linux source and provenance
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.source.source_kind``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source.repository_id``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source.source_commit``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source.upstream_commit``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source.source_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.source.provenance_report_digest``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.provenance``
     - Only the closed F-02 component source and provenance records are
       authoritative; queue-specific leaves remain an unresolved dependency.
   * - Linux kernel ABI
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.abi_contract_id``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.packages[]``
     - Kernel userspace and DRM ABI identity is the closed component contract
       and its typed artifact/package records.
   * - Linux configuration
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_inputs[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.patch_lock``
     - These are the exact F-02 component leaves. K-01's profile, preimage,
       required-symbol, and initramfs closure has no F-02 wire path and is an
       explicit rejected dependency until F-02 supplies one.
   * - Linux toolchain and reports
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock``
     - The closed F-02 lock entries and reports are required. K-01 command
       arrays are not component leaves in the rejected F-02 snapshot.
   * - DTB source and schema
     - ``Trusted<PlatformManifest>.payload.components.dtb_set.source``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.config_inputs[]``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.dt_schema``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``
     - DTS inventory, binding-set identity, DT ABI, and source digest agree.
   * - DTB artifacts and mutation
     - ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``;
       ``Trusted<DtbMutationEnvelope>``
     - Component artifacts are manifest leaves. Pre/post DTB digests and the
       authenticated mutation envelope are the separate F-02 auxiliary type.
   * - Firmware bundle and ABI
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.source``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.provenance``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.firmware_schema``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi_contract_id``
     - F-02 exposes the closed firmware schema and component artifact/source
       records. K-01 ABI generation, compatibility, and tuning fields are an
       unresolved rejected dependency, not local aliases.
   * - Mesa stack
     - ``Trusted<PlatformManifest>.payload.components.mesa_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.abi_contract_id``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.artifacts[]``
     - GPU generation, kernel DRM ABI, firmware ABI, and Mesa artifact agree.
   * - Boot stack
     - ``Trusted<PlatformManifest>.payload.components.boot_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.abi_contract_id``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.boot_check_profile``
     - The opaque artifact boundary and manifest-declared boot profile are
       identified without source inspection. Slots and health are boot-health
       payload fields, not component aliases.
   * - Kernel, DTB, firmware, Mesa, boot, and userspace artifacts
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.artifacts[]``
     - Every artifact has the exact F-02 ID, component ID, kind, media type,
       size, content digest, version, and signature-policy fields; only this
       closed set may be selected.
   * - Compatibility relations
     - ``Trusted<PlatformManifest>.payload.components.<owner>.compatibility_relations[]``;
       ``Trusted<PlatformManifest>.payload.compatibility``
     - Component relation lists are the canonical source and the top-level
       value is the exact F-02 projection; no parallel top-level relation list
       exists. The relation member set includes ``relation_schema``.
   * - Rollback closure
     - ``Trusted<PlatformManifest>.payload.components.<component>.rollback``;
       ``Trusted<PlatformManifest>.payload.rollback.last_known_good_required``;
       ``Trusted<PlatformManifest>.payload.rollback.manifest_ids[]``;
       ``Trusted<PlatformManifest>.payload.rollback.artifact_ids[]``;
       ``Trusted<PlatformManifest>.payload.rollback.minimum_retention``;
       ``Trusted<PlatformManifest>.payload.rollback.failure_attempt_limit``;
       ``Trusted<PlatformManifest>.payload.rollback.projection_digest``
     - Component rollback coordinates are canonical; the top-level value is
       the exact F-02 projection and contains no local set or last-known-good
       alias.
   * - Package, lock, and evidence closure
     - ``Trusted<PlatformManifest>.payload.package_set``;
       ``Trusted<PlatformManifest>.payload.consumer_schema_set``;
       component ``toolchain_lock`` and ``report_lock``;
       ``Trusted<QualificationRecord>.payload.evidence``
     - F-02 owns the exact package and consumer projections, component locks,
       and qualification evidence. There is no manifest ``locks`` or
       ``evidence`` shadow object.

The canonical F-02 source record is
``Trusted<PlatformManifest>.payload.components.linux_kernel.source`` with the
closed fields ``source_kind``, ``repository_id``, ``source_commit``,
``upstream_commit``, ``source_digest``, and
``provenance_report_digest``. Its sibling component
``provenance`` and ``recipe_digest`` are also required. There is no F-02
queue-specific path, queue ledger, command-array, tag-object, signer, or
range-diff field in the rejected snapshot. K-01's queue evidence is
therefore an unresolved F-02 dependency and cannot be placed under a local
shadow path.

The proposed queue procedure still starts from the immutable signed ref and
peeled commit selected by the coordinator, preserves the ordered patch ledger,
and records the fetch/ref, verification, apply, and range-diff evidence. The
current candidate values are immutable tag ``asahi-7.1.9-1`` and peeled commit
``77cb8f24c2381a8abb7272d7bbdec548d6426a8a``. The tag object ID, signer
fingerprint, verification evidence digest, and verification result are split
between known and unknown evidence: the locally resolved tag object is
``f3bed724fe7160d4a1f9dfb35a6e68f55153d41a``, while GPG verification is
unavailable. The tag object and peeled commit do not imply a passed signature
check. Until an accepted F-02 revision provides typed queue leaves or a
canonical evidence binding, the queue gate remains ``FAIL_CLOSED``.

F-02 defines the Linux configuration leaves only as the closed
``Trusted<PlatformManifest>.payload.components.linux_kernel.config_inputs[]``
and
``Trusted<PlatformManifest>.payload.components.linux_kernel.patch_lock``
records. The exact generated ``olddefconfig`` preimage bytes, normalized
``CONFIG_SYMBOL=value`` serialization, required-symbol linkage, built-in/module
classification, and ordered initramfs dependency/signature closure have no
corresponding F-02 component fields in the rejected snapshot. They remain an
unresolved F-02 dependency and cannot be represented by a local configuration
lock alias. The report lock retains only its F-02-defined closed
entries until an accepted schema revision supplies the missing K-01
configuration record.

The aggregate rows above are closed typed objects, not shorthand aliases. The
following leaf paths make the closure mechanically addressable:

.. list-table:: Canonical K-01 leaf paths
   :header-rows: 1
   :widths: 32 53 15

   * - Record
     - Exact canonical leaf paths
     - Missing result
   * - Source and component grammar
     - Each of the exact component paths
       ``Trusted<PlatformManifest>.payload.components.linux_kernel``,
       ``Trusted<PlatformManifest>.payload.components.dtb_set``,
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle``,
       ``Trusted<PlatformManifest>.payload.components.mesa_stack``, and
       ``Trusted<PlatformManifest>.payload.components.boot_stack`` has only
       the F-02 ``Component`` fields: ``component_schema``, ``component_id``,
       ``source``, ``provenance``, ``recipe_digest``, ``abi_contract_id``,
       ``config_inputs[]``, ``policy_inputs[]``, ``patch_lock``,
       ``toolchain_lock``, ``report_lock``, ``artifacts[]``, ``packages``,
       ``firmware_schema``, ``dt_schema``, ``boot_check_profile``,
       ``rollback``, and ``compatibility_relations[]``
     - Missing, extra, or mismatched component leaves fail through the F-02
       parser or cross-document verifier. K-01-specific queue/config/ABI
       details remain unresolved dependencies.
   * - Artifact records
     - ``Trusted<PlatformManifest>.payload.artifacts[].artifact_id``;
       ``Trusted<PlatformManifest>.payload.artifacts[].component_id``;
       ``Trusted<PlatformManifest>.payload.artifacts[].kind``;
       ``Trusted<PlatformManifest>.payload.artifacts[].media_type``;
       ``Trusted<PlatformManifest>.payload.artifacts[].size_bytes``;
       ``Trusted<PlatformManifest>.payload.artifacts[].content_digest``;
       ``Trusted<PlatformManifest>.payload.artifacts[].artifact_version``;
       ``Trusted<PlatformManifest>.payload.artifacts[].signature_policy_id``;
       and the identical ``artifacts[]`` record on each component
     - Missing or conflicting component/projection records fail with the
       F-02 manifest projection or cross-document signal.
   * - Inputs and locks
     - On each exact component path, ``config_inputs[]`` has
       ``input_id``, ``input_kind``, ``source_digest``,
       ``normalized_content_digest``, and ``policy_digest``;
       ``policy_inputs[]`` has ``policy_id``, ``policy_version``,
       ``policy_digest``, and ``source_digest``; ``patch_lock``,
       ``toolchain_lock``, and ``report_lock`` use their F-02 closed entries
     - Required component modes, missing entries, unknown fields, and digest
       conflicts fail through F-02 parsing or binding integrity.
   * - Firmware, DT schema, and boot profile
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.firmware_schema``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.dt_schema``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.boot_check_profile``
     - These nullable fields are required only on their named component by
       the F-02 fixed profile; K-01 must not invent ``abi``, ``bundle``,
       ``tuning``, ``slots``, or ``boot_health`` children.
   * - Component rollback and relations
     - Each component's ``rollback`` and ``compatibility_relations[]``;
       top-level ``Trusted<PlatformManifest>.payload.compatibility`` and
       ``Trusted<PlatformManifest>.payload.rollback`` are exact F-02
       projections
     - Projection mismatch, missing component records, and duplicate
       relation ownership fail through the F-02 authority-conflict signal.

Every listed array is closed, ordered, and duplicate-rejected by the F-02
grammar. A consumer resolves leaves from the single
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

The queue starts at the immutable signed ref and peeled commit associated with
the F-02 source record
``Trusted<PlatformManifest>.payload.components.linux_kernel.source``. The
recorded ``source_commit`` and ``upstream_commit`` are compared with the
external signed-ref evidence before a queue is rebased or recreated. The
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

The ordered patch ledger, queue tip, range diff, and signed-ref verification
evidence are external queue evidence associated with the F-02 component
``source`` and ``provenance`` records; they are not F-02 component fields.
The ledger contains each ordinal, commit ID, patch-blob digest, subject,
author, committer, source or review reference, upstream status, dependency
link, and retirement condition. A local remote name, a moving branch, a date,
or a source SHA without the signed-ref evidence is not provenance. Until F-02
adds an accepted typed binding for this evidence, its absence keeps the queue
gate fail-closed.

Reconstruction starts from ``authoritative_url``, fetches only
``immutable_signed_ref``, verifies the tag object and its peeled commit,
checks the advertised-ref digest, applies ``ordered_patch_ledger`` in order,
checks ``queue_tip``, and recomputes ``range_diff_digest`` against
``previous_base``. The report retains the exact fetch/ref command array,
advertisement, verification output, patch order, range-diff, environment,
toolchain recipe, and output digests. An unavailable signed ref, tag object,
fingerprint, GPG result, patch entry, source record, or range-diff is
``FAIL_CLOSED`` rather than best effort.

The source operations require complete lock-derived argv arrays, but the
rejected F-02 snapshot defines no component ``command_arrays`` path. The
arrays therefore remain implementation evidence associated with the component
report lock, not a K-01 or local manifest schema. Each future accepted binding
must contain every executable, argument, repository URL/ref, output path, and
environment assignment needed for the operation. The runner must pass the
arrays directly with no shell expansion, caller-supplied remote, moving branch,
or appended option.

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

Board identity is admitted from the explicit
``Trusted<PlatformManifest>.payload.board_targets[]`` after its
``Trusted<PlatformManifest>.payload.board_registry_digest`` resolves the
single trusted
``Trusted<BoardRegistry>.payload.boards[].board_id`` record. The exact F-02
identity leaves are that record's
``identity_match.macos`` and ``identity_match.linux.compatible[]`` plus
``identity_match.linux.model``; ``soc.soc_id`` and the closed ``firmware``
record remain diagnostic and compatibility inputs, not substitutes for board
identity. The manifest has no local ``board_id`` or ``identity_match`` tuple.

Any missing, duplicate, conflicting, expired, or unverifiable registry record
quarantines the board and prevents DTB, firmware, config, boot, or physical
profile selection. The j713 case is an explicit hostile fixture: the same token
from two observations cannot reconcile two registry records or create a board
target. Token equality, a product name, a SoC token, or a Linux compatible
string cannot fill a missing registry member by inference. Only one complete
trusted registry record may satisfy an explicit manifest target.

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
provisional rejected F-02 type. Its common envelope paths are the candidate
external contract; the payload
binds these exact fields:

.. list-table:: dtb-mutation-envelope/v1 bindings
   :header-rows: 1
   :widths: 38 42 20

   * - Field
     - Exact payload path
     - K-01 requirement
   * - Schema and source set
     - ``Trusted<DtbMutationEnvelope>.payload.schema``;
       ``Trusted<DtbMutationEnvelope>.payload.schema_set_digest``;
       ``Trusted<DtbMutationEnvelope>.payload.source_identity``;
       ``Trusted<DtbMutationEnvelope>.payload.dt_schema_identity``
     - Exact F-02 schema-set digest, source identity, and DT schema identity.
   * - Manifest and board
     - ``Trusted<DtbMutationEnvelope>.payload.board_identity.board_id``;
       ``Trusted<DtbMutationEnvelope>.payload.platform_manifest_document_id``;
       ``Trusted<DtbMutationEnvelope>.payload.platform_manifest_payload_digest``
     - Must equal the selected registry board target and the verified platform
       manifest document ID and payload digest.
   * - DTB bytes
     - ``Trusted<DtbMutationEnvelope>.payload.pre_mutation_dtb_digest``;
       ``Trusted<DtbMutationEnvelope>.payload.post_mutation_dtb_digest``
     - SHA-256 is recomputed over the exact pre and post byte strings.
   * - Policy, tool, and artifact
     - ``Trusted<DtbMutationEnvelope>.payload.policy_identity``;
       ``Trusted<DtbMutationEnvelope>.payload.tool_identity``;
       ``Trusted<DtbMutationEnvelope>.payload.artifact_identity``;
       ``Trusted<DtbMutationEnvelope>.payload.firmware_bundle_identity``;
       ``Trusted<DtbMutationEnvelope>.payload.dt_schema_identity``
     - Each exact F-02 identity record is compared by all of its closed
       members, including version and digest.
   * - Mutations
     - ``Trusted<DtbMutationEnvelope>.payload.authorized_mutations[]``
     - Ordered closed entries contain ``sequence``, ``mutation_id``,
       ``property_path``, ``operation``, before/after value digests,
       authorization rule ID, and before/after preimage digests.
   * - Nonce, replay, expiry, and signer
     - ``Trusted<DtbMutationEnvelope>.payload.nonce``;
       ``Trusted<DtbMutationEnvelope>.payload.replay_identity.replay_id``;
       ``Trusted<DtbMutationEnvelope>.payload.replay_identity.replay_domain``;
       ``Trusted<DtbMutationEnvelope>.payload.replay_identity.issued_nonce_digest``;
       ``Trusted<DtbMutationEnvelope>.payload.signer_authority``;
       ``Trusted<DtbMutationEnvelope>.payload.expires_at``;
       ``Trusted<DtbMutationEnvelope>.signatures[]``
     - The nonce digest uses the F-02 ``omarchy-dtb-nonce/v1`` domain and the
       replay domain is ``omarchy-dtb-mutation/v1``. Expiry, durable replay,
       authority binding, and the verified signature are all required.

The producer may emit an envelope only after it has the exact platform-manifest
document ID and payload digest, board identity, full source identity,
pre/post-mutation DTB digests, policy/tool/artifact/firmware/DT-schema
identities, ordered mutation list, nonce, expiry, and replay identity. The
producer signs the common envelope and retains the canonical bytes, signature
evidence, before/after DTB bytes, and ordered diff. The replay identity uses
``replay_domain = "omarchy-dtb-mutation/v1"`` and
``issued_nonce_digest = sha256(ASCII("omarchy-dtb-nonce/v1") || 0x00 || nonce)``.
For the K-01 baseline, the ordered allowlist contains only the exact AGX node
paths for ``apple,firmware-abi``; it is empty for every other property, node,
compatible, memory reservation, phandle, and boot argument. Any additional
entry requires a new binding, manifest schema, and policy revision before it
can be emitted.
The consumer independently verifies the signature and expiry, reserves the
nonce/replay identity durably, recomputes both DTB digests and every value
digest, checks the source/schema/policy/tool/artifact/firmware/DT-schema tuple,
checks ordered operations against the allowlist, and records the resulting
post digest in the owning DTB component artifact. It uses the F-02 closed
authority seam for signer role and scope; K-01 does not resolve a signer
locally.
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
     - ``UNKNOWN_MUTATION``
     - Reject; no DTB use.
   * - Wrong signer, domain, context, role, or authority scope
     - ``SIGNATURE_CONTEXT_MISMATCH``
     - Reject before mutation; no local authority fallback.
   * - Wrong before-value digest or operation ordering
     - ``DTB_INPUT_VERIFICATION_FAILURE``
     - Reject; no partial apply.
   * - Wrong policy, tool, or artifact ID/version/digest
     - ``CROSS_DOCUMENT_MISMATCH``
     - Reject; quarantine the DTB.
   * - Stale, expired, or replayed envelope, nonce, or replay ID
     - ``EXPIRY_OR_REPLAY_FAILURE``
     - Reject; retain the replay evidence.
   * - Source identity, board ID, manifest document ID, payload digest, or
       firmware bundle/schema transplanted from another tuple
     - ``CROSS_DOCUMENT_MISMATCH``
     - Reject; quarantine the board and release.
   * - Unknown mutation operation or mutation path
     - ``UNKNOWN_MUTATION``
     - Reject; require a schema/policy revision.
   * - A valid digest is substituted into a different typed field, or a
       claimed digest differs from independently computed bytes
     - ``DTB_INPUT_VERIFICATION_FAILURE``
     - Reject; no claimed digest is trusted.
   * - Source identity or the independently computed post-mutation digest does
       not match the manifest's DTB source/artifact tuple
     - ``CROSS_DOCUMENT_MISMATCH``
     - Reject; quarantine the source, DTB, and dependent release tuple.
   * - Missing signature, signer, expiry, replay identity, source, or report
     - ``DTB_INPUT_BOUNDARY_FAILURE``
     - Reject; ``TOOLING_BLOCK`` or an unknown result is not success.
   * - Wrong nonce domain or nonce digest
     - ``DTB_INPUT_VERIFICATION_FAILURE``
     - Reject before mutation; reserve nothing for the invalid tuple.

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

The F-02 firmware identity is the exact
``Trusted<PlatformManifest>.payload.components.firmware_bundle.firmware_schema``
record with ``schema_id``, ``schema_version``, and ``schema_digest``. The
bundle's source, provenance, ``abi_contract_id``, and ``artifacts[]`` are the
other canonical component leaves. The top-level
``Trusted<PlatformManifest>.payload.firmware_schema`` is only the exact
projection of that component field. F-02 has no ``abi`` object with
generation, compatibility, tuning, key, or kernel/Mesa bound children; those
firmware protocol and calibration facts remain an unresolved dependency and
must not be given local manifest paths. A DT property, marketing name, or
responding client is not provenance for any field.

The kernel rejects absent, unknown, out-of-range, or signature-invalid
firmware schema, memory, reset, or version evidence. A client cannot silently
negotiate an older protocol merely because it responds. Update transitions use
the owning component's typed
``compatibility_relations[]`` and its ``artifacts[].content_digest`` records;
the top-level ``Trusted<PlatformManifest>.payload.compatibility`` and
``Trusted<PlatformManifest>.payload.rollback`` values are exact F-02
projections. An unlisted upgrade or downgrade is refused. Updates are staged,
verified, and committed atomically against the component rollback coordinates;
F-02 does not provide a bundle digest or last-known-good alias.

The exact firmware prerequisite and rejection are fixed by the following
component relation records. Each owning component relation has the closed
F-02 fields ``relation_schema``, ``left_component_id``, ``relation``,
``right_component_id``, ``contract_id``, and ``evidence_digest``. The relation
list is owned by the lexicographically smaller component ID; the top-level
projection is ``Trusted<PlatformManifest>.payload.compatibility``:

.. list-table:: Firmware ABI relation closure
   :header-rows: 1
   :widths: 34 44 22

   * - Required fact
     - Typed relation endpoints
     - Unknown or unfrozen result
   * - Firmware generation and compatibility
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi_contract_id``
       to ``Trusted<PlatformManifest>.payload.components.linux_kernel.abi_contract_id``
     - F-02 relation absent or mismatched: reject with its closed
       cross-document/projection signal.
   * - Firmware tuning and DT schema
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.firmware_schema``
       to ``Trusted<PlatformManifest>.payload.components.dtb_set.dt_schema``
     - F-02 relation absent or mismatched: reject with its closed
       cross-document/projection signal.
   * - Firmware and Mesa
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.abi_contract_id``
       to ``Trusted<PlatformManifest>.payload.components.mesa_stack.abi_contract_id``
     - F-02 relation absent or mismatched: reject with its closed
       cross-document/projection signal.
   * - Firmware and boot stack
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``
       to ``Trusted<PlatformManifest>.payload.components.boot_stack.abi_contract_id``
       and ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``
     - F-02 relation absent or mismatched: reject with its closed
       cross-document/projection signal.
   * - Kernel, DTB, and firmware artifacts
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.artifacts[]``,
       ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``, and
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``
       to the corresponding typed entries in
       ``Trusted<PlatformManifest>.payload.artifacts[]``
     - F-02 artifact projection or cross-document mismatch: reject.
   * - Manifest and rollback
     - ``Trusted<PlatformManifest>.payload.document_id`` to
       ``Trusted<PlatformManifest>.payload.rollback.manifest_ids[]`` and
       each component's ``rollback.previous_manifest_ids[]`` and
       ``artifact_ids[]``
     - F-02 rollback projection or cross-document mismatch: reject.

The prerequisite is an accepted F-02 component relation entry with typed
endpoints, one closed relation kind, a contract ID, and evidence digest,
authenticated in the same platform manifest. Any unknown or unfrozen tuple is
rejected. Recovery selects a manifest ID and artifact set from the exact
component rollback coordinates and top-level rollback projection; it never
mixes a new kernel or DTB with an old firmware member. Unknown reset, crash,
shared-memory, mailbox, or coredump behavior is a failed ABI admission, not an
optional diagnostic.

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
     - F-02 configuration dependency remains unresolved.
   * - ``APPLE_RTKIT``
       ``drivers/soc/apple/Kconfig:36-39``
     - ``APPLE_MAILBOX``; ``ARCH_APPLE || COMPILE_TEST``
     - ``ARCH_APPLE || COMPILE_TEST``; no select
     - F-02 configuration dependency remains unresolved.
   * - ``APPLE_SART``
       ``drivers/soc/apple/Kconfig:61-63``
     - ``ARCH_APPLE || COMPILE_TEST``
     - ``ARCH_APPLE || COMPILE_TEST``; no select
     - F-02 configuration dependency remains unresolved.
   * - ``MFD_MACSMC``
       ``drivers/mfd/Kconfig:328-333``
     - ``ARCH_APPLE || COMPILE_TEST``; ``OF``; ``APPLE_RTKIT``
     - No enclosing Apple condition; selects ``MFD_CORE``
     - F-02 configuration dependency remains unresolved.
   * - ``NVME_APPLE``
       ``drivers/nvme/host/Kconfig:125-130``
     - ``OF && BLOCK``; ``APPLE_RTKIT && APPLE_SART``;
       ``ARCH_APPLE || COMPILE_TEST``
     - No enclosing Apple condition; selects ``NVME_CORE``
     - F-02 configuration dependency remains unresolved.
   * - ``APPLE_PMGR_MISC``
       ``drivers/soc/apple/Kconfig:28-30``
     - ``PM``
     - ``ARCH_APPLE || COMPILE_TEST``; no select
     - F-02 configuration dependency remains unresolved.
   * - ``APPLE_PMGR_PWRSTATE``
       ``drivers/pmdomain/apple/Kconfig:5-11``
     - ``PM``
     - ``ARCH_APPLE || COMPILE_TEST``; selects ``REGMAP``, ``MFD_SYSCON``,
       ``PM_GENERIC_DOMAINS``, and ``RESET_CONTROLLER``
     - F-02 configuration dependency remains unresolved.
   * - ``ARCH_APPLE`` platform selection
       ``arch/arm64/Kconfig.platforms:36-40``
     - No declared ``depends on``
     - ``select APPLE_AIC``; ``select APPLE_PMGR_PWRSTATE if PM``;
       ``select HAVE_SHARED_GPIOS``
     - F-02 configuration dependency remains unresolved.

The effective report also records the framework symbols used by the selected
source: ``CONFIG_PM``, ``CONFIG_OF``, ``CONFIG_BLOCK``,
``CONFIG_ARCH_APPLE``, ``CONFIG_COMPILE_TEST``, ``CONFIG_MAILBOX``, and
``CONFIG_NVME_CORE``. ``CONFIG_MAILBOX`` is a framework/config-closure fact,
not an invented ``depends on`` edge for ``APPLE_MAILBOX``. A module or built-in
arrangement that fails the exact expressions, selected symbols, or required
framework closure remains a blocked K-01 configuration result; deferred-probe
recovery cannot turn it into health.

There is a separate operational initialization order. It is evidence about
runtime readiness and is not a Kconfig dependency graph: boot handoff, AIC,
timers, CPU, console, clocks, pinctrl/GPIO, and DART precede subsystem probe;
the Apple mailbox is ready before RTKit; RTKit is ready before SMC; RTKit is
ready before Apple NVMe; SART is ready before Apple NVMe; and PMGR power-domain
initialization is reported as its own branch. Each consumer records the
prerequisite result and failure class. The document makes no additional
dependency claim.

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
``Trusted<PlatformManifest>.payload.board_registry_digest``,
``Trusted<PlatformManifest>.payload.board_targets[]``,
``Trusted<BoardRegistry>.payload.boards[].board_id``,
``Trusted<PlatformManifest>.payload.components``;
``Trusted<QualificationRecord>.payload.document_id``,
``Trusted<QualificationRecord>.payload.board.board_id``,
``Trusted<QualificationRecord>.payload.manifest.manifest_id``, and
``Trusted<QualificationRecord>.payload.manifest.manifest_digest``. It also
requires the exact typed F-02 verifier result for the promotion action,
provided through ``Trusted<TrustContext>`` and ``ExpectedContext``. A local
board ID, report digest, or promotion boolean cannot substitute for those
equalities.

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

Kernel boot health: F-02 boot-health/v1 dependency
--------------------------------------------------

The kernel reports only the F-02 ``boot-health/v1``
``Trusted<BootHealthCore>`` contract. It is a signed core and contains no
success marker. Its exact additional payload fields are
``board_id``, ``manifest_id``, ``manifest_digest``, ``profile_id``,
``profile_digest``, ``lineage_id``, ``source_generation``, ``slot``,
``attempt``, ``checks``, ``checks_digest``, ``success``, and ``fallback``.
The exact leaves are
``Trusted<BootHealthCore>.payload.board_id``,
``Trusted<BootHealthCore>.payload.manifest_id``,
``Trusted<BootHealthCore>.payload.manifest_digest``,
``Trusted<BootHealthCore>.payload.profile_id``,
``Trusted<BootHealthCore>.payload.profile_digest``,
``Trusted<BootHealthCore>.payload.lineage_id``,
``Trusted<BootHealthCore>.payload.source_generation``,
``Trusted<BootHealthCore>.payload.slot.slot_id``,
``Trusted<BootHealthCore>.payload.slot.slot_generation``,
``Trusted<BootHealthCore>.payload.slot.boot_artifact_digest``,
``Trusted<BootHealthCore>.payload.attempt.counter``,
``Trusted<BootHealthCore>.payload.attempt.started_at``,
``Trusted<BootHealthCore>.payload.attempt.finished_at``,
``Trusted<BootHealthCore>.payload.attempt.previous_slot``,
``Trusted<BootHealthCore>.payload.attempt.boot_context_generation``,
``Trusted<BootHealthCore>.payload.checks[]``,
``Trusted<BootHealthCore>.payload.checks_digest``,
``Trusted<BootHealthCore>.payload.success``,
``Trusted<BootHealthCore>.payload.fallback.decision``,
``Trusted<BootHealthCore>.payload.fallback.target_slot``,
``Trusted<BootHealthCore>.payload.fallback.rollback_set_digest``, and
``Trusted<BootHealthCore>.payload.fallback.failure_code``. The core's
``manifest_id`` equals the selected platform manifest ``document_id`` and its
``manifest_digest`` equals the verifier-computed platform-manifest
``payload_digest``.

The required checks and limits are not K-01 constants. They are resolved from
the exact F-02 manifest path
``Trusted<PlatformManifest>.payload.components.boot_stack.boot_check_profile``
and its ``profile_id``, ``profile_digest``, ``required_check_ids[]``,
``allowed_classes[]``, ``measurement_rules[]``, ``retry_limit``,
``failure_limit``, and ``rollback_manifest_ids[]`` leaves. The rollback
projection is resolved from
``Trusted<PlatformManifest>.payload.rollback.last_known_good_required``,
``manifest_ids[]``, ``artifact_ids[]``, ``minimum_retention``,
``failure_attempt_limit``, and ``projection_digest``, together with every
component's closed ``rollback`` coordinates. The core does not edit either
manifest projection. Every required ``checks[]`` entry has the manifest
profile's ID, class, measurement, and evidence digest, and its status is the
F-02 value ``pass``, ``fail``, or ``not-run``. Missing, extra, duplicate,
reordered, unknown, or laundered entries fail the record.

The attempt counter is persistent and strictly increases for the selected
board, manifest, slot, slot generation, lineage, and source generation. A
decrease, reset, invalid jump, wrap, or atomic-record failure uses the F-02
``BOOT_COUNTER_FAILURE`` signal. A recovery transition is separately
authenticated and names the old/new counter, reason, target, and source
generation. Retry uses the exact manifest ``retry_limit`` or ``failure_limit``
where applicable; it never substitutes a local numeric value. A retry or
fallback preserves every F-02 core binding and may select only a slot and
artifact set derived from the exact component rollback records.

The optional success object is a separate authenticated
``boot-success-mark/v1`` ``Trusted<BootSuccessMark>`` payload. Its exact
binding path to the core is ``Trusted<BootSuccessMark>.payload.core_digest``
(the verifier-computed ``D_core``), plus
``payload.board_id``, ``payload.manifest_id``, ``payload.manifest_digest``,
``payload.profile_id``, ``payload.profile_digest``, ``payload.lineage_id``,
``payload.source_generation``, ``payload.slot_id``,
``payload.slot_generation``, ``payload.attempt_counter``,
``payload.marker_generation``, ``payload.marked_at``,
``payload.checks_digest``, ``payload.rollback_set_digest``, and
``payload.marker_replay_id``. It contains no authority to change those
values. The verifier recomputes the profile digest from the exact
``boot_check_profile`` and the rollback-set digest from the exact component
rollback records and manifest rollback projection. It also recomputes
``D_core`` before comparing it with ``core_digest``. A success mark is valid
only after every manifest-derived required check is ``pass`` and the core and
mark signatures, expiry, and replay identity verify. Embedding a success
marker in the core, trusting a boolean, or accepting a mark with any
mismatched binding uses the F-02 boot hold/reject signals.

Boot failures use only the F-02 ``FailureCode`` vocabulary. The six boot HOLD
codes are ``BOOT_MARKER_AUTH_FAILURE``, ``BOOT_CONTEXT_MISMATCH``,
``BOOT_COUNTER_FAILURE``, ``BOOT_REQUIRED_CHECK_FAILURE``,
``BOOT_FALLBACK_FAILURE``, and ``TRUST_BOUNDARY_FAILURE``; all other F-02
codes are reject decisions. Unknown classes, schema versions, counter
encodings, reset records, policy IDs, check IDs, mark presence, core digest,
or marker verification state are failures, not warnings. Every failed
attempt, retry, fallback, and recovery action is retained. K-01 cannot claim
boot health when the F-02 schema, policy, persistence, trusted source, exact
bindings, or recovery target is unavailable.

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

Ownership, signatures, and approvals are not a K-01 schema. The only authority
input is the exact typed ``Trusted<TrustContext>`` supplied by F-03 and the
typed ``ExpectedContext`` supplied to the common F-02 verifier. K-01 consumes
the one external seam:

.. code-block:: text

   verify(Canonical<T>, Trusted<TrustContext>, VerifiedClock,
          ExpectedContext) -> Trusted<T> | TrustError
   admit(Trusted<T>, Admitted<Policy>) -> Admitted<T> | AdmissionError

The F-02 exhaustive signing table selects the payload type, domain, context,
signer role, and complete expected scope. K-01 neither repeats that table nor
defines role, scope, key, threshold, approval, or owner-record fields. CI and
promotion ownership remain external orchestration inputs. F-03/F-02 are
currently rejected and no accepted trust context is present, so every K-01
authority-dependent gate remains ``FAIL_CLOSED``. A wrong role or scope uses
``SIGNATURE_CONTEXT_MISMATCH``; an absent, invalid, or untrusted authority uses
``TRUST_FAILURE`` or ``TRUST_BOUNDARY_FAILURE``; stale or replayed material
uses ``EXPIRY_OR_REPLAY_FAILURE``. No local authority or rejection namespace
may override those F-02 results.

Static and build gates
~~~~~~~~~~~~~~~~~~~~~~

The toolchain is pinned by
``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock``;
its exact F-02 fields are ``mode``, ``entries[].toolchain_id``,
``entries[].toolchain_version``, ``entries[].toolchain_digest``,
``entries[].flags_digest``, and ``lock_digest``. Recipe details, host/container
identity, locale, working-directory policy, make variables, and command arrays
are not F-02 component leaves. A version range, unpinned package install, or
host fallback creates an unresolved K-01 tuple and cannot authorize a gate.

For every queue tip and relevant commit, proposed CI must execute complete argv
arrays without shell expansion. The rejected F-02 snapshot defines no
component ``command_arrays`` path, so the following is a K-01 CI interface
proposal, not a canonical manifest path:

.. list-table:: Proposed K-01 CI command arrays
   :header-rows: 1
   :widths: 25 55 20

   * - Gate
     - External K-01 CI argv key
     - Required target
   * - Config closure
     - ``k01-ci/argv/olddefconfig``
     - ``olddefconfig``
   * - DTB build
     - ``k01-ci/argv/dtbs``
     - ``dtbs``
   * - DTB schema check
     - ``k01-ci/argv/dtbs_check``
     - ``dtbs_check``
   * - AGX binding check
     - ``k01-ci/argv/dt_binding_check``
     - ``dt_binding_check``
   * - Warning build
     - ``k01-ci/argv/warning_build``
     - ``W=1`` build
   * - Patch validation
     - ``k01-ci/argv/checkpatch``
     - ``scripts/checkpatch.pl``
   * - Documentation
     - ``k01-ci/argv/docs``
     - documentation and RST/toctree check
   * - Reproducibility
     - ``k01-ci/argv/reproducible_build``
     - clean rebuild comparison

Each future referenced value must be a non-empty closed array of complete
argument strings. The array itself supplies the cross-compiler assignment,
schema-file assignment, Sphinx executable, output policy, and every other
variable; no caller may append arguments or provide a host value. The config
array consumes the locked fragment; the DT arrays cover every Apple DTB in the
manifest's explicit inventory, and the binding array names
``Documentation/devicetree/bindings/gpu/apple,agx.yaml``. The runner verifies
the normalized config digest, required built-in/module decisions, initramfs
closure, sorted JSON warning output, and the pinned zero-warning or reviewed
warning baseline only after an accepted schema and executable CI consumer
exist.

The report lock at
``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock`` pins
only the F-02 fields ``mode``, ``entries[].report_id``,
``entries[].report_kind``, ``entries[].report_digest``,
``entries[].producer_toolchain_digest``, and ``lock_digest``. The following
K-01 report schemas, statuses, and expected artifacts are proposed CI report
content, not additional F-02 component fields:

* report schemas are
  ``linux-k01-config-report/v1``, ``linux-k01-dtb-report/v1``,
  ``linux-k01-provenance-report/v1``, ``linux-k01-docs-report/v1``,
  ``linux-k01-reproducibility-report/v1``, and
  ``linux-k01-boot-health-report/v1``, ``linux-k01-mesa-report/v1``, and
  ``linux-k01-qualification-report/v1``;
* proposed report status labels are exactly
  ``PASS``, ``FAIL``, ``TOOLING_BLOCK``, and ``UNKNOWN``; skipped, advisory,
  partial, or locally invented statuses do not pass a gate. Its retention is
  append-only, access-audited retention for the life of the support program
  plus seven years. Its expected artifact set is exactly
  ``kernel_image``, ``kernel_modules``, ``initramfs``, ``dtb_set``,
  ``binding_schemas``, ``platform_manifest``, ``signatures``, ``reports``,
  ``raw_logs``, ``reproducibility_diff``, ``firmware_bundle``, ``boot_artifacts``,
  ``mesa_artifacts``, ``userspace``, ``rollback_set``, and
  ``dtb_mutation_envelopes``.

Those report labels are not F-02 ``FailureCode`` values and cannot be used as
authority outcomes. An unavailable tool or absent report remains a blocked
K-01 CI result until the external schema and consumer are accepted.

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
  baseline ID, and the exact typed verifier-context result. A non-zero command,
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

Correction-round hostile specification probes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following variants are scratch-only design probes. They record the exact
F-02 specification signal that a future parser, verifier, or consumer must
produce; they are not evidence that an executable validator exists. Every
variant is rejected before a partial trusted value or successful gate can be
returned.

.. list-table:: Hostile correction probes
   :header-rows: 1
   :widths: 25 38 37

   * - Planted variant
     - Scratch mutation
     - Exact F-02 rejection signal
   * - Path drift
     - Rename an exact component path to an unlisted alias or add an alternate
       manifest identity path.
     - ``UNKNOWN_FIELD`` at the first unknown path.
   * - Omitted component leaf
     - Remove a required component source, lock, artifact, rollback, relation,
       firmware schema, DT schema, or boot profile member.
     - ``PARSE_SCHEMA_FAILURE`` at the missing member.
   * - Typed digest substitution
     - Put a valid digest from another typed artifact, source, policy, or
       schema into the selected field.
     - ``CROSS_DOCUMENT_MISMATCH`` at the first unequal bound field.
   * - Wrong signer or scope
     - Use a valid signer with the wrong payload row, domain, context, role, or
       expected scope.
     - ``SIGNATURE_CONTEXT_MISMATCH`` at the first signer/context/scope path.
   * - Stale or replayed evidence
     - Reuse an expired envelope, nonce, replay ID, source generation, or
       already-consumed marker.
     - ``EXPIRY_OR_REPLAY_FAILURE`` at the expiry or replay path.
   * - Cross-board or cross-manifest evidence
     - Transplant a board predicate, manifest document ID/digest, or
       qualification binding from another tuple.
     - ``CROSS_DOCUMENT_MISMATCH`` at the first identity binding.
   * - Unknown failure code
     - Add an unlisted failure code or duplicate a code with a second meaning.
     - ``PARSE_SCHEMA_FAILURE`` at the code-table entry.
   * - Malformed boot health
     - Omit/duplicate/reorder a required check, use an unknown class/status,
       change the profile or counter binding, or accept a marker without the
       matching core digest.
     - ``BOOT_REQUIRED_CHECK_FAILURE``, ``BOOT_CONTEXT_MISMATCH``,
       ``BOOT_COUNTER_FAILURE``, or ``BOOT_MARKER_AUTH_FAILURE`` at the first
       applicable path.
   * - DTB/firmware transplant
     - Replace ``firmware_bundle_identity`` or ``dt_schema_identity`` with a
       valid identity from another board, manifest, or DTB tuple.
     - ``CROSS_DOCUMENT_MISMATCH`` at the first identity digest.
   * - DTB mutation replay or mutation drift
     - Change nonce domain/digest, before/after digest, operation order, path,
       or an allowlisted property after signing.
     - ``EXPIRY_OR_REPLAY_FAILURE``, ``UNKNOWN_MUTATION``, or
       ``DTB_INPUT_VERIFICATION_FAILURE`` at the first differing path.
   * - Partial CI
     - Remove a generated binding/output entry, alter its digest, omit a
       required report/artifact, or present an unavailable tool as success.
     - ``BINDING_INTEGRITY_FAILURE`` or a blocked K-01 CI result; no promotion.

These probes were planted and checked in temporary copies only. Because the
F-02 snapshot has no executable parser, verifier, generated consumer binding,
or CI runner, the observed result is a specification mapping and not runtime
guard evidence. The future implementation must reproduce the same code and
first-path signals.

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
  boot-artifact digests,
  ``Trusted<PlatformManifest>.payload.components.boot_stack.boot_check_profile``,
  ``Trusted<BootHealthCore>.payload.slot``,
  ``Trusted<BootHealthCore>.payload.attempt``, boot arguments, expected root
  device, timeout, retry policy, and
  ``Trusted<PlatformManifest>.payload.rollback``;
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
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.source``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.toolchain_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_inputs[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.patch_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.report_lock``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.artifacts[]``
     - Refuse promotion if source/config/artifact provenance is incomplete;
       detailed queue and config records remain unresolved F-02 inputs.
   * - Device tree
     - ``Trusted<PlatformManifest>.payload.components.dtb_set.source``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.config_inputs[]``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.dt_schema``;
       ``Trusted<PlatformManifest>.payload.components.dtb_set.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.board_registry_digest``;
       ``Trusted<PlatformManifest>.payload.board_targets[]``;
       ``Trusted<BoardRegistry>.payload.boards[].identity_match``
     - Refuse boot-health success when identity or ABI is inconsistent.
   * - Firmware
     - ``Trusted<PlatformManifest>.payload.components.firmware_bundle.source``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.provenance``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.firmware_schema``;
       ``Trusted<PlatformManifest>.payload.components.firmware_bundle.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.compatibility``
     - Required client failure blocks capability and promotion.
   * - Human boot artifact
     - ``Trusted<PlatformManifest>.payload.components.boot_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.abi_contract_id``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.components.boot_stack.boot_check_profile``;
       ``Trusted<BootHealthCore>.payload.slot``;
       ``Trusted<BootHealthCore>.payload.fallback``
     - Kernel reports the observed handoff; it does not inspect the source
       repository or substitute an unqualified artifact.
   * - Mesa
     - ``Trusted<PlatformManifest>.payload.components.mesa_stack.source``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.abi_contract_id``;
       ``Trusted<PlatformManifest>.payload.components.mesa_stack.artifacts[]``;
       ``Trusted<PlatformManifest>.payload.compatibility``
     - No GPU/display PASS with an arbitrary Mesa build.
   * - Userspace/initramfs
     - ``Trusted<PlatformManifest>.payload.components.linux_kernel.abi_contract_id``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.config_inputs[]``;
       ``Trusted<PlatformManifest>.payload.components.linux_kernel.packages[]``;
       ``Trusted<PlatformManifest>.payload.package_set``;
       ``Trusted<PlatformManifest>.payload.artifacts[]``
     - Missing required userspace/firmware is an honest failure, not a warning;
       detailed initramfs closure remains the unresolved F-02 config dependency.

The boot-artifact handoff must expose enough authenticated data for the kernel
to verify the exact selected
``Trusted<BoardRegistry>.payload.boards[].identity_match`` record,
``Trusted<PlatformManifest>.payload.components.dtb_set.source`` and
``dt_schema`` tuple,
``Trusted<DtbMutationEnvelope>.payload.pre_mutation_dtb_digest``,
``Trusted<DtbMutationEnvelope>.payload.post_mutation_dtb_digest``,
``Trusted<DtbMutationEnvelope>.payload.platform_manifest_document_id``,
``Trusted<DtbMutationEnvelope>.payload.platform_manifest_payload_digest``,
``Trusted<DtbMutationEnvelope>.payload.firmware_bundle_identity``,
``Trusted<DtbMutationEnvelope>.payload.dt_schema_identity``, and the
``Trusted<BootHealthCore>.payload.slot`` context. Memory reservations, boot
arguments, console/debug transport, and DTB location are evidence fields under
the same authenticated handoff; they cannot override the manifest, boot core,
or mutation envelope. The details and implementation remain with the
qualified human owner.

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

* The supplied F-02 snapshot at ``c315c7e79928d0041deb582bed79a61074361b21``
  is independently rejected and frozen. Its paths are a provisional external
  dependency, not accepted canonical authority; F-03 trust context and
  authority policy are likewise unsettled. K-01 remains ``FAIL_CLOSED``.
* No executable F-02 schema package, code generator, generated Python/Swift/
  Rust binding, validator, accepted/hostile fixture runner, consumer guard, or
  CI workflow is present in this design lane. The command arrays, report
  statuses, and detailed queue/config/firmware evidence above are design
  requirements only.
* This checkout is intentionally not clean: exactly 13 modified paths outside
  this owned RST belong to another lane. They are not read, staged, tested
  through, or altered by this correction. No clean-checkout closure is claimed.
  A clean isolated checkout and the executable gates remain required.
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
  evidence are not present in this design lane. Any absent queue, config, DTB,
  firmware, boot, Mesa, artifact, ABI, or rollback record remains a
  fail-closed candidate gap; the provisional path mapping above does not
  close that dependency.
* The relationship between the DT's ``apple,firmware-abi`` property, the
  driver's ``apple,firmware-compat`` property, bootloader overwrites, and the
  release firmware record is an implementation evidence gap. It must be
  represented by the accepted typed entries in the owning component's
  ``compatibility_relations[]`` and the exact top-level
  ``Trusted<PlatformManifest>.payload.compatibility`` projection; an unknown
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
     - Accepted F-02 manifest and F-03 ``Trusted<TrustContext>`` verifier
       context, exact source/config/DTB/firmware/boot/Mesa tuple report, AGX
       warning census resolution, named approvals, reproducible
       reconstruction, and no untracked interface authority.
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

Until those evidence packages exist, every K-slice remains incomplete or in
design.
This file records the plan only; it does not promote a board, kernel, DTB,
firmware, Mesa build, or release.
