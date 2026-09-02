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

The design is grounded in ``linux-omarchy`` branch ``origin/asahi`` at
``77cb8f24c2381a8abb7272d7bbdec548d6426a8a`` (2026-05-30), the current
Apple bindings, ``arch/arm64/configs/asahi.config``, the Apple DT inventory,
the kernel KUnit/kselftest and documentation guidance, and the Apple/Asahi
entries in ``MAINTAINERS``. The repository has no ``.github`` workflow tree at
that snapshot. CI described here is therefore a proposed Omarchy/platform
boundary, not an observed property of the source repository.

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
this reviewed tip. The following are the required canonical field names and
semantics that F-02 must publish; they are not permission to create a second
local manifest. Until an accepted, signed F-02 schema revision and registry
artifact exist, K-01 is ``FAIL_CLOSED`` and cannot be reported as PASS, DONE,
or implementation-ready.

The manifest is canonical serialized data: UTF-8, one field per line in the
schema-defined order, no duplicate keys, explicit units, and SHA-256 over the
exact bytes including the final newline. Any field not present in the accepted
schema, any unknown enum, and any digest or signature mismatch invalidates the
tuple.

.. list-table:: Required platform-manifest/v1 fields
   :header-rows: 1
   :widths: 30 48 22

   * - Field
     - Required contents
     - K-01 rule
   * - ``manifest.schema_id`` and ``manifest.id``
     - Accepted schema revision and SHA-256 of canonical manifest bytes.
     - Both required; F-02 acceptance is required.
   * - ``board.registry_id``, ``board.product_id``, ``board.soc_id``
     - Canonical registry keys, human-authoritative product identifier, and
       canonical SoC identifier.
     - Must resolve to one signed registry record.
   * - ``board.linux_compatible``
     - Exact ordered Linux compatible strings for the admitted DTB.
     - Must be mapped by the registry, never inferred from a product ID.
   * - ``linux.repo_url``, ``linux.fetched_at_utc``, ``linux.base_ref``,
       ``linux.base_sha``, ``linux.base_signature_key_id``, and
       ``linux.base_signature_evidence``
     - Authoritative source URL, UTC fetch evidence, immutable signed base
       ref/tag, exact base SHA, pinned signing key, and verification record.
     - All values and successful verification are required.
   * - ``linux.queue_id``, ``linux.queue_sha``, ``linux.queue_digest``,
       ``linux.patch_order``, ``linux.range_diff_digest``, and
       ``linux.provenance``
     - Queue identity, exact queue-tip SHA, SHA-256 of the canonical ordered
       queue, ordered patch ledger, range-diff digest, and author/source
       provenance records.
     - All values are required and must reconstruct the exact tip.
   * - ``linux.config.profile``, ``linux.config.preimage_sha256``,
       ``linux.config.normalized_sha256``
     - Profile name, digest of the exact generated preimage, and digest of its
       normalized ``CONFIG_`` serialization.
     - The preimage and normalization recipe are retained and reproducible.
   * - ``linux.config.built_in``, ``linux.config.modules``, and
       ``linux.initramfs.module_closure``
     - Required symbols forced built-in, required module symbols, and the
       complete ordered module/dependency/signature closure in the initramfs.
     - A missing, unexpectedly modular, unsigned, or extra required-path
       dependency fails admission.
   * - ``dtb.source_identity``, ``dtb.blob_sha256``,
       ``dtb.pre_mutation_sha256``, ``dtb.post_mutation_sha256``,
       ``dtb.mutation_envelope_digest``, ``dtb.binding_schema_set_id``, and
       ``dtb.binding_schema_set_version``
     - Exact DTS/DTB source identity, signed pre-mutation blob digest, measured
       post-mutation digest, envelope digest, and pinned binding/schema set ID
       and version used for validation.
     - The source identity, blob, and registry mapping must agree.
   * - ``toolchain.id`` and ``toolchain.recipe_digest``
     - Pinned compiler, Rust/LLVM, ``dtc``, dt-schema, Sphinx, Python, and
       build-container or host recipe identity.
     - A toolchain drift is a new tuple, not a transparent rebuild.
   * - ``firmware.abi`` and ``firmware.bundle``
     - Per-client ABI generation/compatibility/tuning record and exact signed
       bundle digests.
     - Unknown or unverified firmware is a hard failure.
   * - ``artifacts.coupling``
     - Ordered entries for kernel image, modules, initramfs, signed DTB,
       mutation envelope, firmware, boot artifact, Mesa, userspace, and this
       manifest, each with producer, digest, and compatibility relation.
     - Consumers may use only this exact closed set; no member is substituted.

The config preimage is the exact generated ``olddefconfig`` bytes, including
comments, blank lines, original ordering, and final LF. Its digest is
``linux.config.preimage_sha256``. Normalization removes comments and blank
lines, validates each remaining ``CONFIG_SYMBOL=value`` line, sorts by symbol,
and appends one final LF. The SHA-256 of that canonical sequence is
``linux.config.normalized_sha256``. The report must include the raw preimage,
both digests, the defconfig and fragment identities, and a symbol-by-symbol
built-in/module classification. The initramfs closure is the transitive
dependency closure of every required module, with module filename, vermagic,
signature key ID, and digest; it is not just the list of modules explicitly
requested by a build recipe.

The required-symbol ledger names the exact value and reason for every symbol
needed by the admitted board/profile. At minimum, its dependency rows cover
``CONFIG_ARCH_APPLE``, ``CONFIG_OF``, ``CONFIG_PM``, ``CONFIG_BLOCK``,
``CONFIG_MAILBOX``, ``CONFIG_APPLE_MAILBOX``, ``CONFIG_APPLE_RTKIT``,
``CONFIG_APPLE_SART``, ``CONFIG_NVME_CORE``, ``CONFIG_NVME_APPLE``,
``CONFIG_MFD_MACSMC``, ``CONFIG_APPLE_PMGR_MISC``, and
``CONFIG_APPLE_PMGR_PWRSTATE``; capability-specific rows add DART, display,
audio, network, input, and Mesa-related symbols. A symbol marked required but
missing from the built-in list, module list, or explicit not-applicable list
fails the config gate.

``artifacts.coupling`` is checked in both directions. The manifest names the
exact kernel/config/DTB/firmware/Mesa/userspace members, and each member embeds
or is accompanied by the manifest ID, slot, board registry ID, and relevant
compatibility digests. A build or boot report with only a source SHA or only a
DTB digest is incomplete. This tuple is a design contract; no current artifact
is claimed to satisfy it.

K-01: source, queue, configuration, and interface contract
-----------------------------------------------------------

Upstream sync and minimal patch queue
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The queue starts at an immutable ``origin/asahi`` base and is rebased or
recreated from a recorded upstream reference. The downstream branch must not
become a second unreviewable kernel history.

.. list-table:: Queue layers
   :header-rows: 1
   :widths: 18 24 58

   * - Layer
     - Owner and review
     - Rule
   * - Upstream base
     - Kernel coordinator; Asahi maintainers
     - Record the exact upstream commit, repository URL, fetch time, and
       ``git range-diff`` against the previous base. Do not use a moving tag or
       an unrecorded local merge.
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

Base and queue provenance is self-contained. The F-02 record must carry the
authoritative Linux repository URL, the UTC ``fetched_at`` timestamp, the
immutable signed ref or tag, the signing-key fingerprint, and the successful
signature-verification evidence. It must record the exact base SHA, queue ID,
queue-tip SHA, queue digest, downstream tip SHA, ordered patch SHAs, and a
SHA-256 digest for each serialized patch. ``queue_sha`` is the exact commit
reached by applying the ordered queue; ``tip_sha`` must equal it unless the
report explicitly records a reproducible, manifest-approved build transform.
Each patch entry also records its author, committer, source URL or review
record, upstream status, ``Fixes`` or dependency link, and retirement
condition. A local remote name, a moving branch, or a date alone is not
provenance.

The reconstruction record is executable and reproducible. Starting from a
fresh checkout of the recorded URL, the builder must fetch only the recorded
signed ref, verify its signature against the pinned key fingerprint, detach
at ``base_sha``, verify every ordered patch digest, and apply the patches in
the recorded order. It then verifies ``tip_sha``, the queue digest, and the
generated ``range-diff`` against the previous ``base_sha..tip_sha`` pair. The
report retains the exact fetch command, ref advertisement, verification
output, patch order, range-diff, environment/toolchain recipe, and all output
digests. If the signed ref, key, patch order, source provenance, or range-diff
is unavailable, reconstruction is ``FAIL_CLOSED`` rather than best effort.

The queue digest preimage is also canonical: one UTF-8, LF-terminated record
per patch in application order containing ordinal, commit SHA, patch-blob
SHA-256, subject, author, source/review reference, upstream status, and
dependency/retirement links. The manifest stores that preimage digest and its
ordered patch ledger, so two queues with the same tip but different provenance
or ordering cannot silently share a queue ID.

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

Board identity is admitted through an explicit registry mapping. Each signed
record contains ``board.registry_id``, ``board.product_id``, the exact ordered
``board.linux_compatible`` list, ``board.soc_id``, hardware revision, and
profile ID. The mapping is many-field but one-way at admission: a Linux
compatible string is accepted only when the registry maps it to exactly one
canonical product/board record, and a product ID is never converted into a
compatible string by string manipulation. Serial number and product-family
labels are evidence attributes, not substitutes for the registry key.

Contradictory metadata is quarantined. In particular, any ``j713`` value that
does not occur in the signed human-authoritative reconciliation artifact is
``UNKNOWN`` and cannot select a DTB, firmware, config, or physical profile.
The quarantine is lifted only by a new signed registry record naming the
conflicting sources, the resolved canonical ID, the exact compatible mapping,
owner, and approval. No family-level inference can resolve the contradiction.

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

The reviewed source is not already a passing baseline. For example,
``arch/arm64/boot/dts/apple/t8103.dtsi`` and ``t8112.dtsi`` contain
``apple,firmware-version`` and ``apple,firmware-compat`` plus performance and
power-tuning properties, while the current AGX schema does not describe those
properties and rejects additional properties. ``t602x-die0.dtsi`` also carries
firmware-version/compat values outside that schema. This is an explicit
schema/DTS mismatch to resolve or quarantine; this document makes no
``dtbs_check`` pass claim.

The gate has two sets. The accepted baseline set is the exact compatible and
property set above, with a pinned zero-warning report for every in-scope Apple
DTB. Deliberate downstream work may propose
``apple,firmware-version``, ``apple,firmware-compat``,
``operating-points-v2``, ``apple,perf-base-pstate``,
``apple,min-sram-microvolt``, ``apple,avg-power-filter-tc-ms``,
``apple,avg-power-ki-only``, ``apple,avg-power-kp``,
``apple,avg-power-min-duty-cycle``, ``apple,avg-power-target-filter-tc``,
``apple,fast-die0-integral-gain``, ``apple,fast-die0-proportional-gain``,
``apple,perf-filter-drop-threshold``, ``apple,perf-filter-time-constant``,
``apple,cs-opp``, ``apple,afr-opp``, and
``apple,csafr-min-sram-microvolt``, but only as separately owned binding
changes with schema definitions, source references, owner approval, and a
retirement/upstreaming record. Those properties are not part of the accepted
baseline until that review is complete. A warning may be present in a
quarantined downstream report only when its exact path, schema message, owner,
rationale, and retirement condition are pinned. Any new or unexplained
warning, any warning baseline drift, or any DTB outside the explicit inventory
is ``FAIL_CLOSED``.

For each gate, CI stores the schema-set ID and commit, DTS/DTB source and
digest inventory, exact ``dtc`` and dt-schema versions, command line, and a
sorted machine-readable warning report. The required commands are
``make ARCH=arm64 dt_binding_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/gpu/apple,agx.yaml``
and ``make ARCH=arm64 dtbs_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/gpu/apple,agx.yaml``
for the named inventory. A missing tool or report is ``TOOLING_BLOCK``; it is
never converted to a schema PASS.

DTB digest and mutation boundary
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The signed build artifact has ``dtb.pre_mutation_sha256``: SHA-256 of the
exact DTB bytes produced by the pinned source/toolchain and signed before any
bootloader operation. For this tuple, ``dtb.blob_sha256`` is the same
pre-mutation digest. This digest is immutable for that artifact. A human
boot-artifact owner may produce a separately signed
``dtb.mutation_envelope``. For the baseline, its authorized mutator is one
named boot-artifact implementation and its allowlist contains only the exact
AGX node paths for ``apple,firmware-abi``. The allowlist is empty for every
other property, node, compatible, memory reservation, phandle, and boot
argument. A future field requires a new manifest/schema review; it cannot be
smuggled into a digest exception.

The envelope binds ``manifest.id``, board registry ID, slot, boot-artifact
digest, pre-mutation digest, post-mutation digest, mutator ID/version,
allowlisted paths, firmware ABI record, mutation count, and signing-key ID.
The boot-artifact owner signs it and retains the pre-mutation bytes, the
post-mutation bytes, the field-level diff, and verification log. The lab owns
the raw handoff capture; the kernel owns the runtime result that it computed
the post-mutation digest and checked the envelope. The coordinator owns
acceptance of that evidence.

``dtb.post_mutation_sha256`` is the measured digest of the mutable bytes
actually handed to the kernel. It is not called immutable and it is not
trusted merely because it is carried in the DTB or boot arguments. Before
using any DT node, the kernel verifies the envelope signature, recomputes the
post-mutation digest over the handed-off bytes, checks the pre-mutation digest
against the signed manifest member, verifies that the field-level diff is
exactly the authorized allowlist, and checks board, slot, manifest, and ABI
identity. Missing envelope evidence, an unauthorized change, a digest
mismatch, or an unknown mutator causes ``DTB_INTEGRITY_FAIL`` and prevents
boot-health success. The kernel does not inspect the boot-artifact source.

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

The firmware ABI record is explicit and versioned. For every client, the
canonical ``firmware.abi`` entry contains ``generation``, ``compat``,
``tuning_profile``, ``source``, ``digest``, ``signature_key_id``,
``min_kernel_abi``, ``max_kernel_abi``, ``required_dt_schema_set``, and
``required_mesa_abi`` where applicable. ``generation`` identifies the
hardware/firmware generation; ``compat`` identifies the protocol and firmware
compatibility tuple; and ``tuning_profile`` identifies the signed provenance
of calibration, power, clock, and performance values. A DT property or a
marketing/product name is not provenance for any of those fields.

The accepted constraint is exact for the ABI major and manifest-selected
generation, and is within the signed minor/patch range recorded by the
firmware registry. The kernel must reject an absent, unknown, out-of-range,
or signature-invalid generation, compat, tuning profile, or version. A client
may not silently negotiate an older protocol merely because it responds.
Update transitions are a signed allowlist of ``old_bundle_digest`` to
``new_bundle_digest`` for a specific board registry ID and manifest schema;
an unlisted upgrade or downgrade is refused. Updates are staged, verified, and
committed atomically, with the last-known-good bundle retained for recovery.

The ABI record couples the firmware bundle to the exact kernel source/queue,
normalized config, DTB source and blob digest, DTB mutation envelope, boot
artifact, and Mesa/userspace ABI digests. The coupling is bidirectional: a
firmware bundle cannot be selected by a manifest that does not name its exact
kernel/DTB/Mesa relations, and the kernel cannot report the client healthy
without verifying those relations. On failure, recovery selects the signed
last-known-good manifest and its complete rollback set; it never mixes a new
kernel or DTB with an old firmware member. Unknown reset, crash, shared-memory,
mailbox, or coredump behavior is a failed ABI admission, not an optional
diagnostic.

The GPU binding currently carries ``apple,firmware-abi`` and describes the
calibration, globals, handoff, and page-table regions consumed by AGX. The
current Rust AGX driver also reads ``apple,firmware-compat``. Their ownership,
versioning, bootloader overwrite semantics, and relationship to the platform
manifest are therefore represented by the records above. The current
schema/DTS mismatch remains an unresolved implementation input: until a
reviewed binding extension and F-02 registry entry exist, those extra
properties and any unknown ABI values are quarantined and cannot satisfy
K-01.

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

The minimum dependency graph is:

.. code-block:: text

   board registry + exact DT compatible
                  |
                  v
   human boot artifact -> DTB handoff -> AIC/timers/CPU/console
                                      |
                                      v
                   clocks/pinctrl/GPIO + DART
                           |             |
                           v             v
                      mailbox       SART
                           |
                           v
                         RTKit ---------------> SMC (MFD_MACSMC)
                           |  \                    |
                           |   \----------------> NVMe (with SART)
                           v                        |
                     DCP/display/AGX               v
                                                    PCIe/USB/root I/O

                   PMGR_MISC -> PMGR_PWRSTATE -> power domains
                       (independent branch; never a substitute for SMC)
                           |
                   +------------------+------------------+
                   v                  v                  v
             cpufreq/thermal      audio/ISP/media    Wi-Fi/Bluetooth/HID
                   |                  |                  |
                   +-------------> Mesa/userspace <-----+
                                      |
                                      v
                    physical qualification -> board promotion

The arrows above are admission prerequisites, not merely preferred probe
order. In the current Kconfig, ``APPLE_RTKIT`` depends on
``APPLE_MAILBOX``; ``NVME_APPLE`` depends on both ``APPLE_RTKIT`` and
``APPLE_SART``; and ``MFD_MACSMC`` depends on ``APPLE_RTKIT``. The PMGR
symbols are separate: ``APPLE_PMGR_MISC`` and ``APPLE_PMGR_PWRSTATE`` have
their own PM/OF and power-domain prerequisites and do not establish SMC
availability. The manifest's config report must show these exact symbol
values and the probe report must show each prerequisite ready before its
consumer. Any module or built-in arrangement that violates this graph is
``CONFIG_CLOSURE_FAIL``; deferred-probe recovery may not turn it into health.

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

The kernel reports only the accepted ``boot-health/v1`` contract. It must not
invent a local health schema or treat a desktop login as success. A canonical
record contains ``schema_id``, ``manifest.id``, ``board.registry_id``,
``slot``, ``attempt.counter``, ``attempt.max``, ``checks``,
``failure_class``, ``retry_target``, ``fallback_target``,
``rollback_set``, ``started_at``, ``completed_at``, and
``success_marker``. The marker authenticates the complete canonical record,
not just a boolean.

The attempt counter is a persistent unsigned 64-bit value with valid range
``1..2^63-1``. It strictly increases for every attempt of a manifest/slot,
and ``attempt.max`` is exactly ``3`` retries before fallback. A decrease,
zero/reset, unknown counter epoch, jump outside the accepted bound, or wrap
is ``COUNTER_INVALID``. A reset is accepted only when the boot-health/v1
contract contains a signed recovery transition with an explicitly named old
counter, new counter, reason, and recovery target; an unknown or unauthenticated
reset still fails closed and cannot produce success.

The required check set and timeout values are fixed for K-01; a manifest may
only make a timeout stricter and must record the resulting value:

.. list-table:: boot-health/v1 required checks
   :header-rows: 1
   :widths: 32 20 48

   * - Check ID
     - Timeout
     - Success predicate
   * - ``manifest_identity``
     - 5 seconds
     - Manifest ID, board registry ID, slot, kernel source/queue, and config
       digest equal the accepted tuple.
   * - ``dtb_integrity``
     - 2 seconds
     - Signed DTB mutation envelope verifies and the recomputed post-mutation
       digest and allowed diff match the manifest.
   * - ``config_module_closure``
     - 10 seconds
     - Required built-ins, modules, signatures, and initramfs closure match.
   * - ``root_storage``
     - 30 seconds
     - The manifest-named root device mounts and the required read/write
       health operation succeeds without I/O or integrity errors.
   * - ``required_firmware_ipc``
     - 30 seconds
     - Every firmware client marked required responds with the accepted ABI,
       memory contract, reset state, and board mapping.
   * - ``userspace_ready``
     - 20 seconds
     - The signed health agent has loaded the exact userspace/initramfs member
       and can persist an authenticated result.

Each check has one of the enumerated results ``PASS``, ``FAIL``, or
``TIMEOUT``; missing, extra, or unknown check IDs fail the record. The kernel
may emit intermediate failure evidence, but it may emit a success marker only
after all six checks pass. The marker is an authenticated, atomically replaced
record: canonical bytes are signed by the health-agent key, written to a
temporary file, flushed, atomically renamed, and the directory flushed. A
marker with the wrong manifest, slot, counter, check-set digest, key ID, or
signature is absent for health purposes.

The only failure classes are ``IDENTITY``, ``DTB_INTEGRITY``, ``CONFIG``,
``FIRMWARE_ABI``, ``STORAGE``, ``HEALTH_TIMEOUT``, ``KERNEL_FATAL``,
``INTEGRITY``, and ``INFRASTRUCTURE``. Unknown classes, schema versions,
counter encodings, reset records, or marker states are failures, not
warnings. A failed attempt retries the same exact slot/manifest at most three
times, preserving every attempt; ``retry_target`` is therefore the same
manifest ID, slot, and artifact tuple. The fallback target is the manifest's signed
``fallback_target``; if it is absent or does not point to a complete accepted
tuple, boot enters recovery. The exact atomic rollback set is
``kernel_image``, ``kernel_modules``, ``initramfs``, ``signed_dtb``,
``dtb.mutation_envelope``, ``firmware.bundle``, ``boot_artifact``, ``mesa``,
``userspace``, and ``manifest``. No member may be rolled back independently.

The kernel owns the observed check results and emits the health report; the
health-agent owner signs the marker; the boot-artifact/recovery owner owns
slot selection; and the coordinator owns acceptance. This ownership split is
part of the manifest evidence. K-01 cannot claim boot health when the accepted
schema, counter persistence, authentication key, exact check set, or recovery
target is not available.

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

F-02 must publish a signed ``owners.v1`` registry resolving these stable owner
IDs to named humans or accountable teams: ``linux-queue-owner``,
``kernel-config-owner``, ``initramfs-owner``, ``dt-binding-owner``,
``board-registry-owner``, ``firmware-abi-owner``, ``boot-artifact-owner``,
``boot-health-owner``, ``ci-toolchain-owner``, ``docs-owner``,
``physical-lab-owner``, and ``qualification-coordinator``. Generic labels such
as "the team" or an unassigned role do not approve a gate. The queue/config
gates require the Linux owner and release owner; DT gates require the DT and
registry owners; firmware and boot-health gates require their named owners
plus the recovery owner; documentation gates require the docs owner; and
physical promotion requires the lab owner plus the coordinator. Approval
records contain owner ID, identity, timestamp, manifest/schema digest, and
signature. Missing owner resolution or a required second approval is
``OWNER_BLOCK`` and keeps K-01 in ``FAIL_CLOSED``.

Static and build gates
~~~~~~~~~~~~~~~~~~~~~~

The toolchain is pinned by ``toolchain.id`` and
``toolchain.recipe_digest``. The signed recipe names the compiler and exact
version/digest (GCC or Clang/LLVM), Rust compiler and LLVM when Rust is used,
``dtc`` version/digest, dt-schema version/digest, Python version, Sphinx
version/digest, host/container image digest, locale, and relevant make
variables. A version range, unpinned package install, or host fallback creates
a new tuple or produces ``TOOLING_BLOCK``.

For every queue tip and relevant commit, run the smallest applicable set first
and the full set before release:

* ``make ARCH=arm64 olddefconfig`` with the recorded platform fragment, then
  verify the normalized config digest, required built-in/module decisions,
  and initramfs module closure;
* ``make ARCH=arm64 CROSS_COMPILE=... dtbs`` and
  ``make ARCH=arm64 CROSS_COMPILE=... dtbs_check`` for all Apple DTBs in scope;
* ``make ARCH=arm64 dt_binding_check`` with
  ``DT_SCHEMA_FILES=Documentation/devicetree/bindings/gpu/apple,agx.yaml`` and
  the corresponding ``dtbs_check`` invocation, retaining sorted JSON warning
  output and the pinned zero-warning or reviewed-warning baseline;
* ``make ARCH=arm64 ... W=1`` for changed kernel/DT/config work where the
  toolchain supports it, plus ``scripts/checkpatch.pl`` and the relevant
  compiler/static-analysis gates;
* ``make SPHINXBUILD=... htmldocs`` or the pinned standalone RST/toctree
  structural checker for this document; an unavailable Sphinx/parser is a
  reported infrastructure failure, not a documentation PASS;
* reproducible build comparison, module/signature inventory, source/license
  provenance, secret scan, conflict-marker scan, and ``git diff --check``;
* no support claim from a compile-only or QEMU result. QEMU is useful for
  generic arm64 regression tests but does not emulate Apple hardware support.

Every command publishes a JSON report with ``report_schema``, status
(``PASS``, ``FAIL``, ``TOOLING_BLOCK``, or ``UNKNOWN``), commit and manifest
IDs, board/profile scope, exact command and environment, toolchain recipe
digest, start/end time, exit code, stdout/stderr digests, input/output
artifact digests, warning baseline ID, and owner/approval IDs. Reports retain
raw logs and machine-readable diagnostics. CI retains the inputs, generated
kernel/modules/initramfs/DTBs, schemas, manifests, signatures, reports, raw
logs, and reproducibility diff for the life of support plus seven years in an
append-only, access-audited store.

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
* manifest SHA, kernel/config/DTB/Mesa/boot-artifact digests, boot arguments,
  expected root device, timeout, retry policy, and last-known-good fallback;
* ordered actions: cold boot, warm reboot, shutdown, suspend/resume, workload,
  hotplug, network reconnect, display/audio/media checks, thermal soak, and
  evidence capture;
* pass/fail predicates, allowed skips, fail-closed teardown, raw log IDs, and
  operator-independent result signing.

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
tip no such accepted manifest is claimed. The minimum kernel-side tuple is:

.. list-table:: Cross-repository tuple
   :header-rows: 1
   :widths: 25 45 30

   * - Member
     - Required coupling
     - Failure behavior
   * - Kernel
     - Source SHA, downstream queue ID, toolchain/build recipe digest, config
       digest, module/signature inventory
     - Refuse promotion if source/config/artifact provenance is incomplete.
   * - Device tree
     - Exact DTB source and blob digest, board/SoC compatible, binding version,
       disabled nodes, memory regions, firmware ABI properties
     - Refuse boot-health success when identity or ABI is inconsistent.
   * - Firmware
     - Per-client bundle digest, schema, version, ownership, memory/DMA and
       reset contract
     - Required client failure blocks capability and promotion.
   * - Human boot artifact
     - Opaque artifact IDs, board/DTB handoff, memory map, bootargs, console,
       selected slot, and boot-health transport
     - Kernel reports the observed handoff; it does not inspect the source
       repository or substitute an unqualified artifact.
   * - Mesa
     - Source/build digest, AGX generation, kernel DRM uAPI, firmware ABI and
       calibration compatibility, conformance result
     - No GPU/display PASS with an arbitrary Mesa build.
   * - Userspace/initramfs
     - Kernel ABI expectations, root device, firmware paths, diagnostics, and
       required modules
     - Missing required userspace/firmware is an honest failure, not a warning.

The boot-artifact handoff must expose enough authenticated data for the kernel
to verify board identity and the DTB/manifest relationship: exact target/SoC
identity, DTB location and pre/post digest envelope, memory reservations, boot
arguments, firmware schema, console/debug transport, and boot slot/health
context. The details and implementation remain with the qualified human
owner.

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
* The manifest fields and signatures for kernel queue ID, config digest, DTB
  digest, firmware schema, boot-artifact ID, and Mesa compatibility need the
  canonical schema owner to confirm exact names and versioning.
* The relationship between the DT's ``apple,firmware-abi`` property, the
  driver's ``apple,firmware-compat`` property, bootloader overwrites, and the
  release firmware registry is not yet a frozen cross-repository contract.
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

* Which exact platform-manifest schema revision and field names are canonical,
  and where should the kernel's config/DT/firmware compatibility report live?
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
     - Accepted F-02 manifest and owners schemas, exact source/config/DTB/
       firmware/boot/Mesa tuple report, AGX warning baseline, named approvals,
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
