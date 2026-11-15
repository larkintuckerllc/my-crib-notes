# Kubernetes API Versioning

A Kubernetes object can have several API representations. The version used to inspect it affects the fields and structure you see, but does not create a separate object. This overview starts with a seemingly missing HPA setting, then explores how built-in and custom resource APIs expose and store configuration.

The opening example comes from Kubernetes 1.20. The later investigation uses a separate Kubernetes 1.35 lab with External Secrets Operator 0.16.2. These are different environments, not stages of one object's upgrade.

## Where did the setting go?

A HorizontalPodAutoscaler (HPA) configures how Kubernetes scales a workload. Consider a Kubernetes 1.20 cluster where an HPA was created through `autoscaling/v2beta2`, including a scaling-behavior setting.

Retrieving that HPA without specifying an API version returned an `autoscaling/v1` representation. Its `spec` did not show the behavior setting. It looked as though the configuration had disappeared.

## Same object, different view

Requesting the HPA through `autoscaling/v2beta2` made the behavior setting visible again. The request did not restore, upgrade, or replace the object. It selected another representation of the same object.

A **resource type** is a category available through the API, such as HPAs. An **object** is one particular instance of that type. Accessing an object through different supported API versions does not give it a new identity.

**Key distinction:** Changing the version used to read an object changes the representation, not the configuration.

## Preferred does not mean most detailed

API discovery advertises the versions a cluster serves and a preferred version for each API group. Clients such as `kubectl` use discovery to resolve resource names when a version is not explicitly requested.

In this example, the cluster preferred `autoscaling/v1`. That explained the default output, even though another served version exposed scaling behavior as an ordinary field.

**Key distinction:** Preferred is a client-selection preference, not a promise of the most expressive schema.

## Versions can reorganize information

Newer API versions do not necessarily contain all the old fields plus additional ones. They can rename or restructure settings.

For example, the v1 HPA representation has a dedicated CPU-utilization target field. The v2-family representation expresses that target within a more general metrics structure. The same 80% CPU target can therefore appear in different shapes.

A field disappearing from a schema does not necessarily mean the underlying capability disappeared. Version-specific documentation explains the mapping.

## What does the controller actually use?

The HPA controller still used the configured scaling behavior in this example. Its interpretation was not limited to the fields visible in the user's chosen `spec` representation.

For this historical HPA API, Kubernetes preserved newer configuration in compatibility annotations when converting to v1. The setting could therefore survive without appearing as `spec.behavior` in that view.

**Key distinction:** Not seeing a setting as an ordinary field in one representation does not prove Kubernetes has forgotten it.

## A schema describes a representation

An API schema describes structure: field names, types, and validation rules. It describes a type of representation, not one object's configured values. For example, a schema can describe a replica count as a number; a particular HPA supplies a value such as 2.

The API reference is useful for understanding that structure. It is not, by itself, a complete explanation of controller behavior or every compatibility mechanism.

## The storage version is another choice

Kubernetes also chooses a version for storing objects of a resource type. That storage choice is separate from both the version a client requests and the preferred version advertised through discovery.

New writes use the configured storage version, even when the client submits another served version. Older stored records may remain in an earlier encoding until rewritten; ordinary API users generally do not need to manage those encodings themselves.

Knowing the storage version gives us a concrete API schema to study.

## Make the choices visible in a lab

In the separate Kubernetes 1.35 lab, discovery and storage-version reporting exposed these choices for HPAs:

| Information | Observed value |
| --- | --- |
| Served autoscaling versions | `autoscaling/v1`, `autoscaling/v2` |
| Preferred autoscaling version | `autoscaling/v2` |
| Version used for new HPA writes | `autoscaling/v2` |

Here, preferred and storage versions agreed. That made the v2 HPA schema a practical reference, but agreement is not a universal rule.

The lab enabled an optional storage-reporting API. Other clusters may not expose that inspection mechanism, even though they still store objects.

## Use the schema alongside the behavior guide

A storage-version schema can be a useful baseline, rather than the final answer to everything Kubernetes understands. A representation can preserve information indirectly, as the historical HPA annotations illustrate.

Feature documentation complements the schema by explaining meaning, defaults, controller behavior, and special cases. Read the documentation for the relevant Kubernetes or operator release, rather than assuming the latest documentation describes an older installation.

**Key distinction:** The schema explains the shape of a representation; behavior documentation explains how the configuration is interpreted.

## Extend the API with a new resource type

A **CustomResourceDefinition (CRD)** adds a resource type to the Kubernetes API. Individual instances of that type are custom resource objects.

For example, External Secrets introduces the `ExternalSecret` type. Defining that type is different from creating an individual ExternalSecret object. The CRD supplies the API definition and schemas; the operator's controllers interpret the custom objects and perform the associated work.

A CRD alone does not fetch external secrets or populate Kubernetes Secrets.

## Software releases and API versions are separate

The lab installed External Secrets Operator 0.16.2 from Helm chart 0.16.2. For the ExternalSecret resource type, that release provides both `external-secrets.io/v1` and `external-secrets.io/v1beta1`.

The software release number and the custom API version describe different things. A software release numbered 0.16.2 can expose a stable v1 API alongside a beta API. It does not follow that the custom API must be named after the software release.

## The CRD declares served and storage versions

Unlike our built-in HPA investigation, the custom resource's configured storage choice is directly available in its CRD. The installed ExternalSecret CRD declares:

| Version | Served | Storage |
| --- | --- | --- |
| `v1` | Yes | Yes |
| `v1beta1` | Yes | No |

**Served** means clients can use that version of the API. **Storage** identifies the one version used for new writes. Each version also has its schema in the CRD.

Clients can read and write through either served version. `storage: false` does not make v1beta1 read-only; writes through it are converted to the storage representation.

The CRD's own `apiextensions.k8s.io/v1` version describes the definition object, not the custom API it introduces.

## Discovery still supplies the preferred version

The CRD has served and storage declarations, but no preferred-version flag. API-group discovery reports the preference.

For the installed `external-secrets.io` group, discovery reports v1 as preferred. The ExternalSecret storage version is also v1. These choices happen to agree, but storage selection does not determine the discovery preference.

## Group versions can expose different resource types

An API version such as `external-secrets.io/v1` selects a version of the group. However, each group version can expose a different set of resource types. They do not all have to advance together.

For the installed External Secrets 0.16.2 definitions:

| Resource type | Served versions within `external-secrets.io` |
| --- | --- |
| ExternalSecret | `v1`, `v1beta1` |
| PushSecret | `v1alpha1` |

The group prefers v1, but PushSecret is not available through that version in this installation. Its CRD declares the versions that actually expose it.

**Key distinction:** A group-level preferred version is not a promise that every resource in the group supports it.

## A missing report is not missing storage

The lab's StorageVersion inspection API reported the HPA storage version, but had no entry for ExternalSecret. That did not mean the operator installation had failed or that custom objects could not be stored.

For custom resources, the CRD directly declares the storage choice and versioned schemas. We could inspect those declarations even without a corresponding StorageVersion report.

## A repeatable investigation

Before concluding that a setting is missing, work through four questions:

1. **Resource type and release:** Which API group and resource are involved, and which Kubernetes or operator release is running?
2. **Served and preferred versions:** Which versions expose that resource, and which representation does the client select by default?
3. **Storage choice, where available:** What version is used for new writes, and where is that declared or reported?
4. **Schemas and behavior guides:** How do the relevant representations express the setting, and how does the controller interpret it?

**In short:** One object can have several API representations. Check the view and its documentation before assuming an unseen setting is gone.
