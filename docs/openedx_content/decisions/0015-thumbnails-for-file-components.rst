.. _openedx-content-adr-0015:

15. Thumbnails for File Components
===================================

Status
------

Draft. Depends on :ref:`openedx-content-adr-0013` and :ref:`openedx-content-adr-0014`.

Context
-------

When a course author uploads an image to a course's "Files", represented as a File component (a :class:`Component` of type ``file``) per :ref:`openedx-content-adr-0013`, Studio automatically generates a smaller thumbnail image. This lets the Files page show a preview of each image without loading the full file.

Three properties of the existing design shape this decision:

- :class:`PublishableEntityVersion` objects, which back every :class:`Component` version (including a File component's), are created once and never updated.
- A thumbnail is not authored content: it is a derived, disposable artifact that can always be regenerated from the original file. The legacy contentstore already treats it this way — it is generated whenever an asset is uploaded or a course is imported, but never written into a course's export package.
- :class:`Media` is a self-contained model for storing and deduplicating binary data by content hash within a :class:`LearningPackage`. Its own documentation explicitly invites third-party apps to extend it with a ``OneToOneField``, independent of any :class:`Component`.

Decisions
---------

1. Thumbnails are not versioned
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Creating or regenerating a thumbnail never creates a new version of the File component. The source image continues to be versioned normally, like any other File component content, but the thumbnail itself is not authored content and does not participate in that versioning.

2. Thumbnail metadata lives platform-side, not in ``openedx_content``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A new Django model, owned by openedx-platform, is keyed to the File component — via a ``ForeignKey`` referencing :class:`Component`, and records which :class:`Media` object is the current thumbnail for that File component, together with any other presentation-adjacent fields the platform needs for that asset. Consistent with Decision 1, this model is a plain, mutable Django row: creating a new thumbnail or changing these fields is a simple update, not a new version.

3. Thumbnail bytes are stored as ``Media``, referenced by a direct foreign key
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The thumbnail image itself is written to ``openedx_content`` as an ordinary :class:`Media` object. The platform-side model from Decision 2 also holds a plain foreign key to that :class:`Media` row — separate from its key relationship to the File component — and regenerating a thumbnail simply repoints that foreign key at a new, or existing and content-identical, :class:`Media` row.

Critically, this :class:`Media` is never associated with the source File component's :class:`ComponentVersion` through ``ComponentVersionMedia``. It is a sibling piece of data that happens to live in the same :class:`LearningPackage`, addressed and deduplicated the same way as any other :class:`Media`, but outside of the versioned graph that the File component participates in.

This gives the thumbnail the storage and deduplication properties of :class:`Media`, including sharing bytes across reruns of a course via ``media_file_namespace``.

4. The platform-side model supports multiple thumbnail variants per File component
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Rather than a single thumbnail per File component, the model from Decision 2 is keyed on ``(file_component, variant)``, where ``variant`` is a short string identifying the size or context a thumbnail was generated for (e.g. ``"default"``), with a ``UniqueConstraint`` on that pair. Each ``(file_component, variant)`` row has its own foreign key to a :class:`Media` row, per this decision.

Today, only one variant (``"default"``) is ever generated. Nothing here commits to building multiple sizes now; it only keeps the schema from needing a breaking migration if and when a second size is needed.

5. Thumbnail generation remains synchronous, at upload and at import time
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Thumbnails continue to be generated eagerly, at the same points where the legacy contentstore already generates them: when an author uploads an image, and when a course is imported. No thumbnail is generated speculatively ahead of need, and none is generated lazily at serving time.

The same idea applies to a course rerun, to copy/paste, and to syncing a downstream File component with a newer published version of its upstream: in every case, a source with a known, already-generated thumbnail is being carried over to a destination, so instead of regenerating from scratch, the platform-side row from Decision 2 is copied (or updated, for a sync) to point at the corresponding copy of the thumbnail :class:`Media` in the destination. Generating a new thumbnail is only needed when there is no existing one to copy.

This only applies when a File component is actually copied or its content is updated by a sync. When an existing File component is reused instead, because it is already a downstream copy of the same upstream, or because it happens to match on ``component_code`` and content hash (:ref:`openedx-content-adr-0014`, Decision 4, first and second bullets), no File component is copied at all, so its existing platform-side row (if any) is already correct and needs no action.

6. Thumbnails are served by a dedicated platform-side endpoint, not the general asset-serving API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The URL scheme :ref:`openedx-content-adr-0005` defines is resolved by looking up ``(component_version, path)`` in ``ComponentVersionMedia``, and per Decision 3, a thumbnail's :class:`Media` is never registered there. So a thumbnail cannot be served through that same endpoint.

Instead, the platform-side app that owns the model from Decision 2 exposes its own endpoint, e.g. ``GET /api/contentstore/v2/file_components/{component_key}/thumbnail``, which looks up the current thumbnail's :class:`Media` directly from that model and serves it, reusing the same underlying serving mechanism :ref:`openedx-content-adr-0005` already defines for any ``Media`` (signed URL / ``X-Accel-Redirect``, ``Content-Security-Policy: sandbox``, ``nosniff``), just reached through a different lookup than ``ComponentVersionMedia``.

Consequences
------------

- No schema changes are required in ``openedx_content``. The File component remains, as :ref:`openedx-content-adr-0013` describes it, a dumb mapping of paths to :class:`Media`.
- Thumbnails do not participate in draft/publish, backup and restore, or any other machinery that assumes a :class:`PublishableEntityVersion` is immutable once created, because a thumbnail never becomes one.
- Thumbnails are not included in a course's export package, and are regenerated from the source asset on import. This preserves the behavior the legacy contentstore already has today. No new mechanism is introduced to make thumbnails portable, because they were never treated as portable content: the source image is the asset of record, and the thumbnail is a derived, disposable artifact of it.
- Regenerating a thumbnail repoints the platform-side model's foreign key at a new :class:`Media` row; it does not delete or overwrite the previous one, which is left unreferenced. ``openedx_content`` has no deletion or garbage-collection mechanism for individual :class:`Media` rows today.
- Because the platform-side model is not versioned, there is only ever one current thumbnail per File component (per variant), not one per :class:`ComponentVersion`, so there is no way to retrieve the thumbnail as it existed at a past version. This is not a practical limitation today: the only consumer of thumbnails is Studio's Files page, which always shows a File component's current draft, so "the current thumbnail" and "the thumbnail of what Studio is showing" are always the same thing.

Rejected Alternatives
---------------------

Storing the thumbnail in an ``openedx_content`` metadata model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:ref:`openedx-content-adr-0013` introduces ``FileComponentMetadata``, a separate, unversioned model keyed by a ``OneToOneField(primary_key=True)`` to :class:`Component`, for per-asset metadata that does not fit within the existing :class:`Component`/:class:`Media` models (``locked``, ``private`` and ``legacy_path``). Those fields are generic: they apply the same way to a File component regardless of what kind of file it holds. Decision 2 of this ADR follows that same trivial-unversioned-model *shape*, but for a purpose specific to image content, not a generic one.

:ref:`openedx-content-adr-0013` also introduces ``ImageMedia``, a model that extends :class:`Media` with a ``OneToOneField`` to capture metadata "purely derived from the image byte data itself" (e.g. dimensions). Storing the current thumbnail as a field on ``ImageMedia`` was considered instead, since it would use the same extension pattern and the same :class:`Media`-backed deduplication as Decision 3.

We reject this because a thumbnail does not meet the "purely derived from the image byte data itself" bar that ``ImageMedia`` is scoped to. Unlike dimensions, which are a fixed function of the bytes alone, the correct thumbnail for a given image also depends on an external, configurable parameter — the target size and resizing algorithm. If that parameter changes, the correct thumbnail for the same, unchanged image bytes changes with it — something that can never happen to an image's dimensions.

More fundamentally, a thumbnail only makes sense for image content, is driven by that platform-configurable parameter rather than being a fixed function of the bytes, and, per Decision 4 of this ADR, needs to support more than one value per image — since ``ImageMedia`` is 1:1 with a single :class:`Media`, a single ``thumbnail`` field on it could not represent more than one size, short of adding one field per size or a separate table keyed by size. None of that fits a model that is meant to stay a simple, purely-derived reflection of a single image's byte data, nor the generic, file-type-agnostic pattern ``FileComponentMetadata`` follows above.

Creating a new ``openedx-core`` app for this metadata
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Rather than putting the model from Decision 2 in openedx-platform, we could add a new sibling app to ``openedx-core`` itself, following the same pattern ``openedx_learning`` and ``openedx_catalog`` use, keyed with a ``ForeignKey`` to :class:`Component` and :class:`Media` the same way. This is technically straightforward: a new app can sit above ``openedx_content`` without violating the layering enforced by ``.importlinter``.

We reject this because ``openedx-core`` apps are meant to be generic, reusable content-management primitives, independent of any particular product's decisions. The specific size and resizing algorithm used to generate a thumbnail is a product/UX decision for this platform's Studio, not a property of the content model itself, and is expected to change independently of ``openedx-core``'s own release cycle. Coupling that kind of platform-specific configuration to a new ``openedx-core`` package would tie its releases to product decisions that have nothing to do with content modeling.

Grouping the thumbnail as a second file within the source File component
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:ref:`openedx-content-adr-0013` allows a File component to group multiple related files together (e.g. different resolutions of the same image), which could in principle hold the thumbnail alongside the original. This was rejected because ``ComponentVersionMedia`` is a full snapshot of a version's media, not a delta: regenerating the thumbnail would force a new :class:`ComponentVersion` that also rewrites the unrelated, unchanged association to the original file. Grouped files also carry no notion of which one is a derived thumbnail versus the source, so this approach would still require a naming convention to tell them apart, without avoiding the versioning cost.

Making ``ComponentVersionMedia`` mutable for derived files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To avoid the cost above, we considered allowing certain paths within a ``ComponentVersionMedia`` mapping to be updated in place, without creating a new :class:`ComponentVersion`. This was rejected because :class:`PublishableEntityVersion` is immutable by design across all of ``openedx_content`` — other systems (backup and restore, course export/import, version history and diff tooling) already depend on a version being a fixed, reproducible snapshot in time. Carving out a mutability exception for one kind of derived file would undermine that guarantee platform-wide, for a benefit that only this feature needs.

Keeping thumbnails in the legacy MongoDB-backed contentstore
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Leaving thumbnails behind in the legacy contentstore would keep a MongoDB dependency alive for one narrow piece of functionality, working against the eventual retirement of ``contentstore``.

An independent storage model with no relationship to ``Media``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A platform-side model with its own image field and storage backend, entirely disconnected from ``openedx_content`` — the pattern ``edxval.VideoImage`` already uses for video thumbnails — was considered. It is a simpler design, but it forfeits the content-addressed deduplication that :class:`Media` already provides, and it does not follow the extension pattern that :class:`Media` explicitly documents for third-party apps.

Generating thumbnails on the fly at serving time
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Rather than generating a thumbnail eagerly at upload and import time, we considered generating it lazily, the first time it is requested, and caching the result. This would avoid spending storage and CPU on thumbnails that are never viewed, and would let the same design serve multiple sizes on demand instead of a single fixed one. We reject it for now on the grounds of added complexity: it requires a caching strategy and changes to the asset-serving view, beyond what this ADR needs to resolve. Nothing about this design precludes adding on-the-fly generation as a future enhancement layered on top of the same storage model.

Open Questions
--------------

**Exact location of the platform-side model.** This ADR specifies that the thumbnail metadata model lives in openedx-platform, but does not pin down which Django app owns it. This is left to implementation.
