====================
djangocms-references
====================

``djangocms-references`` is a `django CMS <https://www.django-cms.org>`_ addon
that answers the question *"where is this object used?"*.

For any content object (a page, an alias, a snippet, a poll, …) it finds every
other object and plugin that references it and lists them in a single **Show
References** view. This is useful before you delete, unpublish or edit shared
content, so you can see what else would be affected.

It is relationship-aware and versioning-aware:

* Tracks both direct model relations (e.g. ``Answer.poll``) and relations that
  live inside plugins (e.g. an ``AliasPlugin`` embedded in a page).
* When `djangocms-versioning <https://github.com/django-cms/djangocms-versioning>`_
  is installed, it shows the version **status**, **author** and **modified
  date** of each reference, lets you filter by version state, and only shows
  the latest version of grouped content.
* Ships out-of-the-box integration for
  `djangocms-alias <https://github.com/django-cms/djangocms-alias>`_,
  `djangocms-snippet <https://github.com/django-cms/djangocms-snippet>`_ and
  `djangocms-versioning-filer <https://github.com/django-cms/djangocms-versioning-filer>`_.

Any third-party app can register its own relationships so that its objects
appear in the references view.


Requirements
============

* Python 3.8+
* Django 3.2 – 4.2
* A running `django CMS 4.0 <https://www.django-cms.org>`_ (or higher) project

Optional integrations that are auto-detected when installed:
``djangocms-versioning``, ``djangocms-alias``, ``djangocms-snippet``,
``djangocms-versioning-filer``.


Installation
============

Install the package::

    pip install djangocms-references

Add it to ``INSTALLED_APPS`` in your project settings::

    INSTALLED_APPS = [
        # ...
        "djangocms_references",
    ]

No migrations create database tables — the app defines a single unmanaged
``References`` model that only exists to carry the ``show_references``
permission.


Usage
=====

Once installed, the app exposes references in three places (depending on which
optional integrations you have):

**CMS toolbar**
    When editing any object in the CMS, a **Show References** button appears in
    the toolbar (for users who hold the ``show_references`` permission). It
    opens the references view for the current object in a sideframe.

**Admin changelist actions**
    Alias, snippet and versioned-filer changelist rows get a references
    action/icon that links straight to the references view for that object.

**References view**
    The view lists — grouped by model — every object that references the target
    object, together with extra columns (version status, author, modified date
    when versioning is enabled) and a version-state filter.

Permissions
-----------

Access is controlled by the ``djangocms_references.show_references``
permission. Grant it to the users/groups that should be able to view
references (superusers have it implicitly). The toolbar button, admin actions
and view all enforce this permission.

Unpublish dependencies (with djangocms-versioning)
--------------------------------------------------

When ``djangocms-versioning`` is installed, the unpublish confirmation screen
is augmented with the list of objects that depend on the version being
unpublished, so editors can see the impact before confirming.

By default this integration assumes versioning is enabled. To disable the
versioning-specific behaviour, set in your project settings::

    DJANGOCMS_REFERENCES_VERSIONING_ENABLED = False


What counts as a reference?
===========================

The references view lists **Django relations, but only the ones that an app has
explicitly registered** via ``reference_fields`` — it is a curated registry, not
an automatic scan of your whole relational graph. Each registered
``(Model, "field_name")`` entry means *"``Model.field_name`` points at the
target type"*, so opening the view for an object shows every registered object
whose field points back at it.

Two things make it more than a plain foreign-key lookup:

* **Plugin relations resolve to the owning object.** If the registered model is
  a ``CMSPlugin`` subclass, the app does not list the plugin row. It follows the
  plugin through its placeholder back to the object that *owns* the placeholder
  (the page or content containing the plugin). Plugins in static placeholders
  are ignored.
* **Versioning is understood.** For versioned content the relation is resolved
  through the grouper (see below), only the latest version is shown, and results
  can be filtered by version state.

How grouper (versioned) references are resolved
-----------------------------------------------

Under ``djangocms-versioning`` an object is split into a stable **grouper**
(e.g. ``Alias``) and per-version **content** (e.g. ``AliasContent``). Apps
register the relation against the grouper — ``(AliasPlugin, "alias")`` — because
that is what a plugin's foreign key points at, so the registry is keyed by the
grouper model.

Resolution then depends on what you open the view for:

* **A grouper** (e.g. the alias/snippet admin action) is matched directly:
  ``AliasPlugin.objects.filter(alias=<grouper>)``.
* **A content object** (e.g. the toolbar while editing) is first mapped to its
  grouper to find the registered relations, then the lookup is rewritten to hop
  through the grouper and back down to that exact version —
  ``AliasPlugin.objects.filter(alias__aliascontent=<content>)``.

How this differs from Django's delete cascade
---------------------------------------------

Django's cascade (the ``on_delete`` behaviour it runs before deleting a row) and
this addon look superficially similar — both are about "what points at this
object" — but they serve different purposes:

* **Coverage.** The cascade automatically considers *every* foreign key pointing
  at the row, from database metadata. References only shows relations an app
  chose to register — a deliberate, curated subset.
* **Semantic hop.** The cascade follows literal foreign keys, so a plugin's key
  cascades the *plugin*; it does not express "the page that uses this". This
  addon deliberately resolves plugin relations back to the owning page/content
  object.
* **Versioning.** The cascade operates on concrete rows with no notion of
  versions or groupers (see above); references resolves groupers.
* **Action.** The cascade actually deletes / nulls / protects rows to preserve
  database integrity. References is purely **read-only** — it never modifies
  anything. Its goal is editorial: letting an author judge the impact of
  deleting *or unpublishing* shared content. (Unpublish is not a delete, so it
  triggers no cascade at all, yet references still reports the dependencies.)


Registering your own references
===============================

``djangocms-references`` discovers relationships through django CMS's app
configuration mechanism. In your app, create a ``cms_config.py`` that extends
``CMSAppConfig``, opt in with ``djangocms_references_enabled = True`` and
declare the relationships you want tracked in ``reference_fields``.

Each entry is a ``(model, field_name)`` tuple describing a relation *from*
``model`` *to* the content you want references for::

    # myapp/cms_config.py
    from cms.app_base import CMSAppConfig

    from myapp.models import Answer, PollPlugin


    class MyAppCMSConfig(CMSAppConfig):
        djangocms_references_enabled = True
        reference_fields = [
            # "Answer.poll points at a Poll" -> answers show up as references
            (Answer, "poll"),
            # Relations declared on a plugin model are tracked through the
            # placeholder, so the *page* using the plugin is shown.
            (PollPlugin, "poll"),
        ]

Notes:

* If ``model`` subclasses ``CMSPlugin``, the relation is treated as a plugin
  relation: references are resolved back to the object owning the placeholder
  the plugin lives on.
* Nested/transitive relationships are supported with the Django ``__`` lookup
  syntax, e.g. ``(SomeModel, "foo__bar")``.
* Versioned (grouper/content) models are handled automatically — you register
  the content relation and the app resolves it through the grouper.

Extra columns
-------------

Add custom columns to the references table with
``reference_list_extra_columns``, a list of ``(getter, verbose_name)`` tuples.
``getter`` receives a content object and returns the cell value::

    class MyAppCMSConfig(CMSAppConfig):
        djangocms_references_enabled = True
        reference_list_extra_columns = [
            (lambda obj: str(obj), "Title"),
            (lambda obj: obj.pk, "ID"),
        ]

Queryset modifiers
------------------

Register functions that post-process each reference queryset (for example to
add ``prefetch_related`` for performance) with
``reference_list_queryset_modifiers``. Each function takes a queryset and
returns a queryset::

    def prefetch_author(queryset):
        return queryset.prefetch_related("versions__created_by")

    class MyAppCMSConfig(CMSAppConfig):
        djangocms_references_enabled = True
        reference_list_queryset_modifiers = [prefetch_author]


How it works
============

At startup the ``ReferencesCMSExtension`` collects every registered
``reference_fields`` entry into two registries: one for plain models and one
for plugin models. When the references view is opened for an object, the app:

1. Looks up all models/plugins that declared a relation to the object's model
   (resolving grouper/content models when versioning is involved).
2. Builds and runs the corresponding querysets — for plugin relations it maps
   matching plugins back to the objects owning their placeholders.
3. Combines querysets of the same model, filters out static placeholders,
   optionally filters by the selected version state, keeps only the latest
   version per grouper, and applies any registered queryset modifiers.
4. Renders the result grouped by model, with the configured extra columns.


Running the tests
=================

Install the test requirements and run the suite::

    pip install -r tests/requirements.txt
    python setup.py test


License
=======

Released under the BSD license. See ``LICENSE.txt``.
