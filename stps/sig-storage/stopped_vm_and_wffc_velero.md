# OpenShift-virtualization-tests Test plan

## **Velero Backup/Restore: Stopped VM and WFFC Support - Quality Engineering Plan**

### **Metadata & Tracking**

- **Enhancement(s):** N/A (Technical Debt / Test Automation)
- **Feature Tracking:** [CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)
- **Epic Tracking:** [CNV-12960](https://redhat.atlassian.net/browse/CNV-12960) — CNV Storage Technical Debt Backlog
- **Feature Maturity:**
  - DP: N/A
  - TP: N/A
  - GA: 5.0
  <!-- Note: originally backlogged for 4.21; retargeted to 5.0 -->
- **QE Owner(s):** Adam Cinko (@acinko-rh)
- **Owning SIG:** sig-storage
- **Participating SIGs:** None

**Document Conventions (if applicable):** None

### **Feature Overview**

OpenShift Virtualization customers rely on Velero/OADP backup and restore for disaster recovery and workload migration, and expect that protection to work whether a VM is running or stopped, and regardless of whether its StorageClass uses Immediate or WaitForFirstConsumer (WFFC) volume binding. Today, stopped (powered-off) VMs and VMs using WFFC StorageClasses are not covered by existing Velero test coverage, so a regression in either scenario could go undetected and put customer data at risk during an actual disaster recovery event. This STP covers General Availability (GA) test coverage, targeting CNV v5.0.0, for backup and restore of stopped VMs (block and filesystem volume modes) and WFFC StorageClass DataVolumes.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

This section documents the mandatory QE review process. The goal is to understand the feature's value,
technology, and testability before formal test planning.

#### **1. Requirement & User Story Review Checklist**

- [x] **Review Requirements**
  - _List the key D/S requirements reviewed:_
    - Velero backup must successfully capture a stopped VM (VM in powered-off state) including its DataVolume and VM specification
    - Velero restore must successfully recreate a stopped VM that can be subsequently started and retain its data
    - Velero backup must handle DataVolumes provisioned with WaitForFirstConsumer (WFFC) StorageClasses where the PV binding is deferred until pod scheduling
    - Velero restore must correctly recreate WFFC-bound DataVolumes and ensure the VM can start with storage provisioned in the correct topology zone

- [x] **Understand Value and Customer Use Cases**
  - _Describe the feature's value to customers:_ Customers using OADP/Velero for disaster recovery and migration need confidence that all VM states are protected. Stopped VMs represent valid production workloads (e.g., template VMs, scheduled-off VMs, maintenance windows), and WFFC is the recommended StorageClass binding mode for multi-zone clusters. Without test coverage, regressions in these scenarios could cause data loss during DR operations.
  - _List the customer use cases identified:_
    - As a cluster admin, I want to back up and restore template VMs that remain in a stopped state, so that my VM templates survive a disaster recovery event
    - As a cluster admin, I need to recover VMs that were powered off during a scheduled maintenance window, so that maintenance activity doesn't put those workloads at risk
    - As a cluster admin, I want to migrate workloads using WFFC StorageClasses between clusters or namespaces, so that storage topology is preserved after migration
    - As a cluster admin, I expect backups of VMs in multi-zone clusters to preserve storage locality, so that restored VMs bind to the correct topology zone

- [x] **Testability**
  - _Note any requirements that are unclear or untestable:_ All requirements are testable through the existing OADP/Velero test framework. The existing `data_protection/oadp/` infrastructure supports parameterized VM configurations and DataVolume modes.

- [x] **Acceptance Criteria**
  - _List the acceptance criteria:_
    - Stopped VM can be backed up via Velero with DataMover without errors
    - Stopped VM can be restored from Velero backup and started successfully
    - Data written to a stopped VM before backup is present after restore and VM start
    - Restored stopped VM's specification (e.g., CPU/memory resources, network interfaces) and DataVolume metadata (size, StorageClass) match the original VM prior to backup
    - VM with WFFC StorageClass DataVolume can be backed up via Velero
    - VM with WFFC StorageClass DataVolume can be restored and started with correct storage binding
  - _Note any gaps or missing criteria:_ The Jira description is minimal ("Add stopped VM and WFFC to velero tests plan and automate the tests"). Specific WFFC StorageClass names and zone topology requirements should be confirmed with the Storage Ecosystem team.

- [x] **Non-Functional Requirements (NFRs)**
  - _List applicable NFRs and their targets:_
    - Backup/restore operations must complete within the existing test timeout thresholds (8-10 minutes per operation)
    - Tests must be idempotent and not leave orphaned resources in the cluster
  - _Note any NFRs not covered and why:_
    - Performance: Not covered — benchmarking of backup/restore duration is out of scope for this test debt task (see Section II.1, Out of Scope)
    - Scalability: This feature introduces no new scale requirements; it relies on the existing OADP/Velero backup mechanism, which already has its own concurrency and throughput limits for bulk/multi-VM backups. Testing against those existing platform-level constraints is out of scope for this test debt task (see Section II.1, Out of Scope)
    - Security: No new security surface introduced; covered by existing RBAC context (see Section II.2, Security Testing)
    - Monitoring/Observability: No new metrics or alerts introduced by this test debt task (see Section II.2, Monitoring)
    - UI: N/A — feature has no UI surface; no customer-facing UI testing value identified
    - Documentation: No new user-facing documentation required; this is QE test coverage for existing product behavior

#### **2. Known Limitations**

- **Velero backup of stopped VMs with WFFC StorageClass requires the DataMover feature**
  - _Sign-off:_ [PM name/date]

#### **3. Technology and Design Review**

- [x] **Developer Handoff/QE Kickoff**
  - _Key takeaways and concerns:_ This is a QE-driven test automation task. No developer handoff is required as the product functionality (Velero backup/restore of stopped VMs and WFFC volumes) already exists. The focus is on closing test coverage gaps.

- [x] **Technology Challenges**
  - _List identified challenges:_
    - OADP operator compatibility with the test environment version must be verified; previous sprint comments indicate OADP testing on main was blocked
    - WFFC StorageClass testing requires a cluster with zone-aware storage provisioner or at minimum a StorageClass configured with `volumeBindingMode: WaitForFirstConsumer`
  - _Impact on testing approach:_ Tests validate StorageClass binding mode before executing WFFC scenarios as a sanity check. A WFFC-capable StorageClass is guaranteed present on test clusters (see Section II.5, Risks -- Test Environment), so this is not a skip condition.

- [x] **API Extensions**
  - _List new or modified APIs:_ None. This task uses existing Velero/OADP APIs and KubeVirt VM APIs.
  - _Testing impact:_ No API changes; existing test automation is reused.

- [x] **Test Environment Needs**
  - _See environment requirements in Section II.3 and testing tools in Section II.3.1_

- [x] **Topology Considerations**
  - _Describe topology requirements:_ Multi-node, multi-zone cluster required for WFFC testing to validate storage topology-aware provisioning.
  - _Impact on test design:_ Test clusters are guaranteed multi-zone (see Section II.3, Test Environment), so no topology-based skip condition is needed for WFFC scenarios.

### **II. Software Test Plan (STP)**

This STP serves as the **overall roadmap for testing**, detailing the scope, approach, resources, and schedule.

#### **1. Scope of Testing**

This test plan covers the addition of two new VM configuration categories to the existing Velero/OADP backup and restore test suite:

1. **Stopped VM backup/restore** -- Validating that VMs in powered-off state can be backed up and restored with data integrity preserved, and that restored VMs can be started successfully.
2. **WFFC StorageClass backup/restore** -- Validating that VMs using DataVolumes provisioned with WaitForFirstConsumer volume binding mode can be backed up and restored with correct storage binding behavior.

Both categories are tested using the DataMover backup path (Velero with CSI DataMover) which is the primary backup mechanism for OpenShift Virtualization.

**Testing Goals**

- [P0] Verify that a stopped VM with block volume mode DataVolume can be backed up and restored via Velero DataMover, and the restored VM can be started with data intact
- [P0] Verify that a stopped VM with filesystem volume mode DataVolume can be backed up and restored via Velero DataMover, and the restored VM can be started with data intact
- [P0] Verify that a Velero backup of a stopped VM with either block or filesystem volume mode DataVolume fails with a clear, actionable error when the OADP/DataMover dependency is unavailable, without leaving orphaned backup resources in the cluster
- [P0] Verify that restoring a stopped VM from a backup with a missing or corrupted DataVolume snapshot fails clearly rather than producing a VM with unbootable or missing storage, for both block and filesystem volume modes
- [P1] Verify that a running VM with WFFC StorageClass DataVolume can be backed up and restored via Velero DataMover with correct storage binding
- [P1] Verify that a stopped VM with WFFC StorageClass DataVolume can be backed up and restored via Velero DataMover
- [P1] Verify data integrity (file content written before backup is readable after restore) for all new test configurations
- [P2] Verify that restored stopped VMs with WFFC DataVolumes can be started and the storage is provisioned in the expected topology zone

_Priority note:_ Stopped VM backup/restore is prioritized P0 because it is a previously completely untested VM state for Velero, representing a higher risk of undetected regressions. WFFC StorageClass coverage is prioritized P1 because it extends an existing, well-established backup/restore path (running VMs) with an additional storage-binding dimension, representing incremental rather than foundational risk.

_Implementation note:_ The two failure-path P0 goals (OADP/DataMover unavailable, missing or corrupted DataVolume snapshot) state intent at the STP level. The concrete failure-injection mechanism (e.g., how OADP unavailability or snapshot corruption is simulated) is a test-implementation detail to be designed in the STD before automation.

**Out of Scope (Testing Scope Exclusions)**

The following items are explicitly Out of Scope for this test cycle and represent intentional exclusions.
No verification activities will be performed for these items during this test cycle.

- **Windows guest OS backup/restore scenarios**
  - _Rationale:_ Existing Windows VM restore coverage already exists in `tests/data_protection/oadp/test_velero.py`; this test debt task targets RHEL guest coverage only and does not extend Windows coverage further
  - _PM/Lead Agreement:_ [Name/Date]

- **CSI-only backup (without DataMover)**
  - _Rationale:_ This task targets the DataMover backup path only, which is the primary backup mechanism for OpenShift Virtualization
  - _PM/Lead Agreement:_ [Name/Date]

- **Velero schedule-based automated backups**
  - _Rationale:_ Only on-demand backup/restore is tested; scheduled backups do not exercise different stopped-VM or WFFC code paths
  - _PM/Lead Agreement:_ [Name/Date]

- **Multi-namespace backup/restore with stopped VMs or WFFC StorageClass DataVolumes**
  - _Rationale:_ The existing multi-namespace backup/restore test (`test_restore_multiple_namespaces`) only exercises a running VM with no WFFC StorageClass; it does not cover stopped VMs or WFFC. This combination is genuinely untested, not covered elsewhere, and is deferred from this test debt task's scope to keep the initial pass focused on single-namespace stopped-VM and WFFC coverage.
  - _PM/Lead Agreement:_ [Name/Date]

- **Backup/restore of VMs with hotplugged volumes**
  - _Rationale:_ Separate feature scope, unrelated to stopped VM or WFFC coverage
  - _PM/Lead Agreement:_ [Name/Date]

- **Performance benchmarking of backup/restore duration for new scenarios**
  - _Rationale:_ Performance testing is not in scope for this test debt task (see Section I.1, NFRs)
  - _PM/Lead Agreement:_ [Name/Date]

**Test Limitations**

- **OADP operator availability on the test cluster is required; if OADP operator installation or compatibility issues arise (as experienced in previous sprints), tests will be blocked**
  - _Sign-off:_ [PM name/date]

- **Zone-aware storage provisioning is cluster-dependent; WFFC topology binding behavior may not be fully exercised on single-zone clusters**
  - _Sign-off:_ [PM name/date]

#### **2. Test Strategy**

**Functional**

- [x] **Functional Testing** -- Validates that the feature works according to specified requirements and user stories
  - _Details:_ All test scenarios validate end-to-end backup and restore workflows. Each scenario creates a VM, writes test data, performs Velero backup, deletes the original resources, restores from backup, and verifies data integrity and VM operability.

- [x] **Automation Testing** -- Confirms test automation plan is in place for CI and regression coverage (all tests are expected to be automated)
  - _Details:_ All tests are automated in Python/pytest within the `tests/data_protection/oadp/` directory of the openshift-virtualization-tests repository. Tests use the existing parameterized framework and Polarion markers for traceability. Target: all scenarios automated and integrated into nightly CI before CNV v5.0.0 code freeze.

- [x] **Regression Testing** -- Verifies that new changes do not break existing functionality
  - _Details:_ Existing Velero backup/restore tests must continue to pass. The new `stopped_vm` fixture and parameterization must not alter the behavior of existing test cases that use `stopped_vm=False`.

- [ ] **Self-Validation Testing** -- Should any of the new tests be included in the self-validation test package?
  - _Details:_ N/A. These scenarios extend edge-case backup/restore state coverage (stopped VMs, WFFC binding) rather than core operational health checks; not proposed for the self-validation package at this time.

**Non-Functional**

- [ ] **Performance Testing** -- Validates feature performance meets requirements (latency, throughput, resource usage)
  - _Details:_ N/A. Performance testing is not in scope for this test debt task.

- [ ] **Scale Testing** -- Validates feature behavior under increased load and at production-like scale
  - _Details:_ N/A. Scale testing is not in scope for this test debt task.

- [ ] **Security Testing** -- Verifies security requirements, RBAC, authentication, authorization, and vulnerability scanning
  - _Details:_ N/A. No new security surface introduced; tests use existing RBAC context.

- [ ] **Usability Testing** -- Validates user experience and accessibility requirements
  - _Details:_ N/A. No UI or CLI changes involved.

- [ ] **Monitoring** -- Does the feature require metrics and/or alerts?
  - _Details:_ N/A. No new metrics or alerts for backup/restore.

**Integration & Compatibility**

- [x] **Compatibility Testing** -- Ensures feature works across supported platforms, versions, and configurations
  - _Details:_ Tests validate both block and filesystem volume modes. WFFC tests validate compatibility with WaitForFirstConsumer StorageClasses. Backward compatibility is maintained by preserving existing test parameterization unchanged.

- [ ] **Upgrade Testing** -- Validates upgrade paths from previous versions, data migration, and configuration preservation
  - _Details:_ N/A. Upgrade testing is not in scope for this test automation task.

- [x] **Dependencies** -- Blocked by deliverables from other components/products
  - _Details:_ Depends on OADP operator being installable and functional on the test cluster. Previous sprint was blocked due to OADP testing on main being non-functional. OADP operator version compatibility with the target CNV version must be verified.

- [x] **Cross Integrations** -- Does the feature affect other features or require testing by other teams?
  - _Details:_ Validation requires coordination with the OADP operator and the underlying storage provider (Storage Ecosystem) to confirm DataMover and WFFC StorageClass compatibility across supported CNV configurations. The OADP/Velero team (Red Hat) and Storage Ecosystem team are the relevant external owners for this confirmation (see Section II.5, Dependencies).

**Infrastructure**

- [ ] **Cloud Testing** -- Does the feature require multi-cloud platform testing?
  - _Details:_ N/A. Tests run on standard OCP cluster infrastructure.

#### **3. Test Environment**

- **Cluster Topology:** Multi-node, multi-zone (minimum 2 worker nodes across at least 2 zones)
- **OCP & OpenShift Virtualization Version(s):** OCP 4.23+ / CNV v5.0.0
- **CPU Virtualization:** Standard (Intel VT-x / AMD-V)
- **Compute Resources:** Default (2 worker nodes with sufficient memory for RHEL VMs)
- **Special Hardware:** None
- **Storage:** StorageClass with snapshot support (`ocs-storagecluster-ceph-rbd` on ODF-based clusters; an equivalent snapshot-capable StorageClass on other platforms); additional StorageClass with `volumeBindingMode: WaitForFirstConsumer` for WFFC tests. Exact names, provisioners, and worker/zone topology to be pinned by the QE owner in coordination with the Storage Ecosystem team (see Section I.1, Acceptance Criteria gaps).
- **Network:** Standard cluster networking (OVN-Kubernetes)
- **Required Operators:** OpenShift Virtualization Operator, OADP Operator (with Velero and DataMover components)
- **Platform:** Any supported OCP platform (bare-metal, AWS, Azure, GCP)
- **Special Configurations:** None

#### **3.1. Testing Tools & Frameworks**

- **Test Framework:** N/A -- existing standard pytest framework
- **CI/CD:** N/A -- existing standard CI lane
- **Other Tools:** None

#### **4. Entry Criteria**

The following conditions must be met before testing can begin:

- [ ] Requirements and design documents are **approved and merged**
- [ ] Test environment can be **set up and configured** (see Section II.3 - Test Environment)
- [x] OADP operator is installable and functional on the target cluster version
- [x] At least one StorageClass with snapshot support is available
- [x] For WFFC tests: a StorageClass with `volumeBindingMode: WaitForFirstConsumer` is available

#### **5. Risks**

**Timeline/Schedule**

- **Risk:** OADP testing on main may remain blocked as experienced in previous sprints, preventing test development and validation
  - **Mitigation:** Monitor OADP operator releases and validate compatibility before sprint commitment. Use a staging branch for test development while awaiting OADP fix.
  - _Estimated impact on schedule:_ 1-2 sprints delay if OADP remains blocked
  - _Sign-off:_ [Name/Date]

**Test Coverage**

- **Risk:** WFFC StorageClass behavior may vary between storage providers, reducing coverage confidence on non-default storage backends
  - **Mitigation:** Document tested StorageClass configurations. Consider adding parameterization for multiple WFFC-capable StorageClasses in a follow-up task to widen provider coverage.
  - _Areas with reduced coverage:_ WFFC with non-default storage providers
  - _Sign-off:_ [Name/Date]

**Test Environment**

- **Risk:** None -- test clusters always have a StorageClass configured with WaitForFirstConsumer binding mode available, so WFFC test coverage does not depend on optional infrastructure.
  - **Mitigation:** N/A
  - _Missing resources or infrastructure:_ N/A
  - _Sign-off:_ N/A

**Untestable Aspects**

- **Risk:** None -- all identified scenarios (stopped VM backup/restore, WFFC binding, topology zone verification) are reproducible with existing cluster infrastructure and require no production-only conditions.
  - **Mitigation:** N/A -- no untestable aspects identified
  - _Alternative validation approach:_ N/A
  - _Sign-off:_ N/A

**Resource Constraints**

- **Risk:** A prior implementation attempt for this same test coverage (RedHatQE/openshift-virtualization-tests#162) was closed without merge when OADP testing on main became blocked (see Section I.2, Known Limitations); rebasing and updating that work requires additional rework effort
  - **Mitigation:** Reuse PR #162's code as a starting reference for the new implementation rather than starting from scratch. Its diff was small (~35 additions) and was already reviewed at the time, reducing the risk and effort of the rebase.
  - _Current capacity gaps:_ None identified
  - _Sign-off:_ [Name/Date]

**Dependencies**

- **Risk:** OADP operator version compatibility with the target CNV/OCP version may introduce API changes or behavioral differences
  - **Mitigation:** Pin OADP operator version in test prerequisites. Validate operator compatibility during environment setup phase.
  - _Dependent teams or components:_ OADP/Velero team (Red Hat)
  - _Sign-off:_ [Name/Date]

**Other**

- **Risk:** None -- no additional risks identified outside the categories above.
  - **Mitigation:** N/A
  - _Sign-off:_ N/A

---

### **III. Test Scenarios & Traceability**

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want to back up and restore a stopped VM with a block volume mode DataVolume via Velero DataMover, so that its specification, DataVolume metadata, and data survive a disaster recovery event
  - _Test Scenario:_ [Tier 2] Verify backup and restore of a stopped VM with block volume mode DataVolume using Velero DataMover; confirm the restored VM's specification and DataVolume metadata match the original, the VM can be started, and data is intact
  - _Priority:_ P0

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want to back up and restore a stopped VM with a filesystem volume mode DataVolume via Velero DataMover, so that its specification, DataVolume metadata, and data survive a disaster recovery event
  - _Test Scenario:_ [Tier 2] Verify backup and restore of a stopped VM with filesystem volume mode DataVolume using Velero DataMover; confirm the restored VM's specification and DataVolume metadata match the original, the VM can be started, and data is intact
  - _Priority:_ P0

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want a Velero backup of a stopped VM to fail with a clear, actionable error when OADP/DataMover is unavailable, so that I'm not left with a silent failure or orphaned backup resources
  - _Test Scenario:_ [Tier 2] Verify that a Velero backup of a stopped VM, with either block or filesystem volume mode DataVolume, fails with a clear, actionable error when OADP/DataMover is unavailable, and that no orphaned backup resources remain in the cluster
  - _Priority:_ P0

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want restoring a stopped VM from a backup with a missing or corrupted DataVolume snapshot to fail clearly, so that I don't end up with a VM that has unbootable or missing storage
  - _Test Scenario:_ [Tier 2] Verify that restoring a stopped VM from a backup with a missing or corrupted DataVolume snapshot fails clearly, for both block and filesystem volume modes, rather than producing a VM with unbootable or missing storage
  - _Priority:_ P0

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want to back up and restore a running VM with a WFFC StorageClass DataVolume via Velero DataMover, so that data integrity is preserved for workloads using WaitForFirstConsumer storage binding
  - _Test Scenario:_ [Tier 2] Verify backup and restore of a running VM with WFFC StorageClass DataVolume using Velero DataMover; confirm the restored PVC/PV binds through the configured WaitForFirstConsumer StorageClass and data integrity is preserved after restore
  - _Priority:_ P1

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want to back up and restore a stopped VM with a WFFC StorageClass DataVolume via Velero DataMover, so that the restored VM starts successfully with correct storage binding
  - _Test Scenario:_ [Tier 2] Verify backup and restore of a stopped VM with WFFC StorageClass DataVolume using Velero DataMover; confirm the restored PVC/PV binds through the configured WaitForFirstConsumer StorageClass and the restored VM can be started
  - _Priority:_ P1

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want data written to a VM before backup to be readable after restore, so that I can trust Velero backups for both stopped VM and WFFC configurations
  - _Test Scenario:_ [Tier 2] Verify data written to a VM before backup is readable after restore for both stopped VM and WFFC configurations
  - _Priority:_ P1

- **[CNV-44308](https://redhat.atlassian.net/browse/CNV-44308)** -- As a cluster admin, I want a restored stopped VM with a WFFC DataVolume to start with storage provisioned in the expected topology zone, so that zone-local data locality is preserved after restore
  - _Test Scenario:_ [Tier 2] Verify that a restored stopped VM with a WFFC DataVolume starts with storage provisioned in the expected topology zone (bound PV's zone label matches the source zone)
  - _Priority:_ P2

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

- **Reviewers:**
  - QE Architect (OCP-V): Ruth Netser (`@rnetser`)
  - QE Members (OCP-V): Jenia Peimer (`@jpeimer`), Emanuele Prella (`@ema-aka-young`), Jose Manuel Castano (`@joscasta`)

* **Approvers:**
  - QE Architect (OCP-V): Ruth Netser (`@rnetser`)
  - PM: Peter Lauterbach (`@pelauter`)
