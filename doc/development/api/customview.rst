.. highlight:: python
   :linenothreshold: 5

.. _`customview`:

Creating custom views
=====================

This page describes how to provide a custom view from within your plugin. Before you start
reading this page, please read and understand how :ref:`URL handling <urlconf>` works in
pretix.

Control panel views
-------------------

If you want to add a custom view to the control area of an event, just register an URL in your
``urls.py`` that lives in the ``/control/`` subpath::

    from django.conf.urls import url

    from . import views

    urlpatterns = [
        url(r'^control/event/(?P<organizer>[^/]+)/(?P<event>[^/]+)/mypluginname/',
            views.admin_view, name='backend'),
    ]

It is required that your URL paramaters are called ``organizer`` and ``event``. If you want to
install a view on organizer level, you can leave out the ``event``.

You can then implement the view as you would normally do. Our middleware will automatically
detect the ``/control/`` subpath and will ensure the following things if this is an URL with
both the ``event`` and ``organizer`` parameters:

* The user is logged in
* The ``request.event`` attribute contains the current event
* The ``request.organizer`` attribute contains the event's organizer
* The user has permission to access view the current event

If only the ``organizer`` parameter is present, it will be ensured that:

* The user is logged in
* The ``request.organizer`` attribute contains the event's organizer
* The user has permission to access view the current organizer

If you want to require specific permission types, we provide you with a decorator or a mixin for
your views::

    from pretix.control.permissions import (
        event_permission_required, EventPermissionRequiredMixin
    )

    class AdminView(EventPermissionRequiredMixin, View):
        permission = 'can_view_orders'

        ...


    @event_permission_required('can_view_orders')
    def admin_view(request, organizer, event):
        ...

Similarly, there is ``organizer_permission_required`` and ``OrganizerPermissionRequiredMixin``.

Frontend views
--------------

Including a custom view into the participant-facing frontend is a little bit different as there is
no path prefix like ``control/``.

First, define your URL in your ``urls.py``, but this time in the ``event_patterns`` section::

    from django.conf.urls import url

    from . import views

    event_patterns = [
        url(r'^mypluginname/', views.frontend_view, name='frontend'),
    ]

You can then implement a view as you would normally do, but you need to apply a decorator to your
view if you want pretix's default behavior::

    from pretix.presale.utils import event_view

    @event_view
    def some_event_view(request, *args, **kwargs):
        ...

This decorator will check the URL arguments for their ``event`` and ``organizer`` parameters and
correctly ensure that:

* The requested event exists
* The requested event is activated (can be overridden by decorating with ``@event_view(require_live=False)``)
* The event is accessed via the domain it should be accessed
* The ``request.event`` attribute contains the correct ``Event`` object
* The ``request.organizer`` attribute contains the correct ``Organizer`` object
* The locale is set correctly

REST API viewsets
-----------------

Our REST API is built upon `Django REST Framework`_ (DRF). DRF has two important concepts that are different from
standard Django request handling: There are `ViewSets`_ to group related views in a single class and `Routers`_ to
automatically build URL configurations from them.

To integrate a custom viewset with pretix' REST API, you can just register with one of our routers within the
``urls.py`` module of your plugin::


    from pretix.api.urls import event_router, router, orga_router

    router.register('global_viewset', MyViewSet)
    orga_router.register('orga_level_viewset', MyViewSet)
    event_router.register('event_level_viewset', MyViewSet)

Routes registered with ``router`` are inserted into the global API space at ``/api/v1/``. Routes registered with
``orga_router`` will be included at ``/api/v1/organizers/(organizer)/`` and routes registered with ``event_router``
will be included at ``/api/v1/organizers/(organizer)/events/(event)/``.

In case of ``orga_router`` and ``event_router``, permission checking is done for you similarly as with custom views
in the control panel. However, you need to make sure on your own only to return the correct subset of data! ``request
.event`` and ``request.organizer`` are available as usual.

To require a special permission like ``can_view_orders``, you do not need to inherit from a special ViewSet base
class, you can just set the ``permission`` attribute on your viewset::

    class MyViewSet(ModelViewSet):
        permission = 'can_view_orders'
        ...

If you want to check the permission only for some methods of your viewset, you have to do it yourself. Note here that
API authentications can be done via user sessions or API tokens and you should therefore check something like the
following::


    perm_holder = (request.auth if isinstance(request.auth, TeamAPIToken) else request.user)
    if perm_holder.has_event_permission(request.event.organizer, request.event, 'can_view_orders'):
        ...


.. warning:: It is important that you do this in the ``yourplugin.urls`` module, otherwise pretix will not find your
             routes early enough during system startup.

.. _Django REST Framework: http://www.django-rest-framework.org/
.. _ViewSets: http://www.django-rest-framework.org/api-guide/viewsets/
.. _Routers: http://www.django-rest-framework.org/api-guide/routers/
