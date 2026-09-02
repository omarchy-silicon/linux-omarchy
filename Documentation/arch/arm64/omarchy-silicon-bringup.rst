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

The m1n1 repository is an opaque human-produced artifact boundary. This note
defines only the external artifact and handoff contract needed by the kernel
and release manifest; it does not inspect, reproduce, or make source-level
claims about m1n1-omarchy.

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

The target type is the exact lower-case board identifier and the SoC ID is the
lower-case Apple device-tree SoC identifier. A family or marketing-compatible
fallback cannot replace either value. New DT files must follow
``Documentation/devicetree/bindings/arm/apple.yaml`` and the DT coding style.

Common includes are additive and auditable. A common file may not silently
override a board's safety-critical power, thermal, audio, or display behavior.
New family files first land with the binding, minimal SoC description, and a
single observed board; additional boards follow as separate, reviewable
patches. Merging two similar SoCs into one DTSI is allowed only when the
register layout, firmware ABI, power topology, and applicable DT properties
are proven compatible.

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

The GPU binding currently carries ``apple,firmware-abi`` and describes the
calibration, globals, handoff, and page-table regions consumed by AGX. The
current Rust AGX driver also reads ``apple,firmware-compat``. Their ownership,
versioning, bootloader overwrite semantics, and relationship to the platform
manifest require a coordinator ruling before a downstream ABI is frozen; see
the unknowns below.

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
     - Platform manifest; human m1n1 boundary
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

The m1n1/boot-firmware owner is human-only under the program fence. The kernel
lane can consume an artifact ID, DTB handoff, boot status, console, and memory
map contract, but cannot review or alter that repository's source. Mesa owns
the AGX userspace/firmware-facing graphics release and conformance result; the
kernel lane owns the DRM/firmware interface it exposes.

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
                   clocks/pinctrl/GPIO + DART/SART + PMGR/SMC
                           |             |              |
                           v             v              v
                     mailbox/RTKit   NVMe/PCIe/USB   cpufreq/thermal/fans
                           |             |              |
                           +-------> power/suspend <----+
                                      |
                   +------------------+------------------+
                   v                  v                  v
             DCP/display/AGX      audio/ISP/media    Wi-Fi/Bluetooth/HID
                   |                  |                  |
                   +-------------> Mesa/userspace <-----+
                                      |
                                      v
                    physical qualification -> board promotion

The operational order is deliberately conservative:

#. **Admission:** resolve exact board ID, SoC ID, DT compatible, firmware
   schema, config profile, and manifest tuple. Refuse ambiguity.
#. **Boot and memory:** verify the opaque boot artifact handoff, map memory,
   start the console, enumerate CPUs, initialize AIC and timers, and capture
   the first dmesg.
#. **Foundational fabric:** bring up clocks, pinctrl/GPIO, PMGR/SMC, DART/SART,
   reset/watchdog, and mailbox/RTKit. Exercise faults and deferred probes.
#. **Root I/O:** qualify NVMe and the boot path, then PCIe, USB, USB-PD/PHY,
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

The first physical gold set should contain at least one base laptop, one base
desktop, one Pro/Max laptop, one Max desktop, and one Ultra desktop. The
coordinator must name the serialised devices, RAM/storage class, firmware
baseline, panel, dock, and recovery host before K-02 can be promoted. The
remaining rows above are still required for M1/M2 full-board qualification;
the gold set only reduces bring-up concurrency.

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

M3, M4, A18, M5, and M6 lanes (K-03 through K-06)
--------------------------------------------------

Every newer lane repeats the K-02 gates. It may reuse evidence only when the
manifest proves that the board, firmware ABI, DT binding, driver behavior,
power profile, display/audio topology, and Mesa tuple are equivalent.

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

Static and build gates
~~~~~~~~~~~~~~~~~~~~~~

For every queue tip and relevant commit, run the smallest applicable set first
and the full set before release:

* ``make ARCH=arm64 olddefconfig`` with the recorded platform fragment, then
  verify the normalized config digest and required built-in/module decisions;
* ``make ARCH=arm64 CROSS_COMPILE=... dtbs`` and
  ``make ARCH=arm64 CROSS_COMPILE=... dtbs_check`` for all Apple DTBs in scope;
* ``make ARCH=arm64 ... W=1`` for changed kernel/DT/config work where the
  toolchain supports it, plus ``scripts/checkpatch.pl`` and the relevant
  compiler/static-analysis gates;
* kernel documentation build or a standalone RST parser for this document;
  an unavailable Sphinx/parser is a reported infrastructure failure, not a
  documentation PASS;
* reproducible build comparison, module/signature inventory, source/license
  provenance, secret scan, conflict-marker scan, and ``git diff --check``;
* no support claim from a compile-only or QEMU result. QEMU is useful for
  generic arm64 regression tests but does not emulate Apple hardware support.

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

The kernel lane consumes the canonical ``platform-manifest/v1`` and publishes
the fields needed by it. The minimum kernel-side tuple is:

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

The m1n1 handoff must expose enough immutable data for the kernel to verify
board identity and the DTB/manifest relationship: exact target/SoC identity,
DTB location and digest, memory reservations, boot arguments, firmware schema,
console/debug transport, and boot slot/health context. The details and
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
       firmware boundary, debug profiles, ownership, dependencies, and CI
       contract approved.
     - Coordinator ruling, owner assignments, canonical schema references,
       source/config/DT/firmware report, and no untracked interface authority.
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
