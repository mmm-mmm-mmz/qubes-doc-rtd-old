==========================
AwesomeWM (window manager)
==========================

Qubes-specific features
=======================

-  support for the Qubes OS window colors
-  rudimentary support for the Qubes application menu entries following
   the freedesktop standard
-  support for custom filters and menu entries

Installation
============

AwesomeWM can be installed with the standard dom0 installation
mechanisms.

.. code:: shell_session

   $ sudo qubes-dom0-update awesome

That’s it. After logging out, you can select AwesomeWM in the login
manager.

Development
===========

To :doc:`contribute code </introduction/contributing>` you may clone the AwesomeWM
repository as follows:

.. code:: shell_session

   $ git clone https://github.com/QubesOS/qubes-desktop-linux-awesome

For build instructions please check the repository *README*.

The repository attempts to follow the upstream Fedora repository.

Common customizations
=====================

This section focuses on Qubes-specific customizations. For generic
AwesomeWM customizations you might want to have a look at the `AwesomeWM website <https://awesomewm.org>`__.

Customizations for AwesomeWM are usually done at
``~/.config/awesome/rc.lua``. The default file can be found at
``/etc/xdg/awesome/rc.lua``.

Application menu
----------------

Starting from Qubes 4.0 application menu entries specific to AwesomeWM
can be put into ``~/.config/awesome/xdg-menu/`` following the
freedesktop standard. The folder might have to be created.

Focus steal hardening
---------------------

The default Qubes OS AwesomeWM installation comes with the defaults set
by the AwesomeWM developers for focus changes. Some users may want more
tight control over window focus changes - especially since focus changes
can have security implications when sensitive data is provided to an
incorrect application or even qube.

Definition
~~~~~~~~~~

For the below example we’ll define *wanted focus changes* as one of the
below:

-  mouse move & click afterwards
-  workspace/tag change
-  pre-defined key combinations for focus changes (e.g. Mod-j & Mod-k)
-  tag assignments and unassignments

Everything else is considered an unwanted *focus steal*.

In particular the following events are not meant to cause a focus
change:

-  new window created
-  a window was closed
-  application request
-  mouse move without click (sloppy focus)

For the below example other requests from applications to the window
manager are meant to be ignored in general as well, e.g.:

-  windows shouldn’t be able to maximize themselves without the user
   giving a respective command to the WM (simple test: Firefox F11 next
   to another window)
-  windows shouldn’t be able to change their size themselves
-  windows shouldn’t be able to modify their borders in any way

Users may want to adjust their definitions and respective
implementations according to their needs.

Implementation
~~~~~~~~~~~~~~

The implementation may be specific to the AwesomeWM version you’re
running. This guide refers to AwesomeWM version 3.5.9 which is available
to Qubes 4.0 users.

Please keep in mind that this guide may not be conclusive. Your mileage
may vary.

Change the autofocus implementation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The line ``require("awful.autofocus")`` in your *rc.lua* implements
various focus-related features for your AwesomeWM instance.

In order to customise these, you can copy the file
``/usr/share/awesome/lib/awful/autofocus.lua`` to
e.g. ``~/.config/awesome/autofocus_custom.lua`` and replace the line
above with ``require("autofocus_custom")``.

Then you can customise the focus behavior. According to our above
definitions it would look as follows:

.. code:: lua

   ---autofocus_custom.lua
   local client = client
   local screen = screen
   local aclient = require("awful.client")
   local atag = require("awful.tag")

   --- When loaded, this module makes sure that there's always a client that will have focus
   -- on events such as tag switching, client unmanaging, etc.
   -- awful.autofocus

   -- Give focus when clients appear/disappear and no one else has focus.
   -- @param obj An object that should have a .screen property.
   function check_focus(obj)
       -- When no visible client has the focus...
       if not client.focus or not client.focus:isvisible() then
           local c = aclient.focus.history.get(obj.screen, 0)
           if c then client.focus = c end
       end
   end

   -- Give focus on tag selection change.
   -- @param tag A tag object
   function check_focus_tag(t)
       local s = atag.getscreen(t)
       if not s then return end
       check_focus({ screen = s })
       if client.focus and client.focus.screen ~= s then
           local c = aclient.focus.history.get(s, 0)
           if c then client.focus = c end
       end
   end

   --made above functions global & removed some focus switches below (user interaction required instead)

   --clear any focus
   function clear_focus()
       --unfortunately this doesn't work at the moment
       --cf. https://github.com/awesomeWM/awesome/issues/164
       --(Qubes uses an older AwesomeWM version that doesn't have the fix yet)
       --client.focus = nil
   end

   atag.attached_connect_signal(nil, "property::selected", check_focus_tag)
   client.connect_signal("unmanage", clear_focus)
   client.connect_signal("tagged", check_focus)
   client.connect_signal("untagged", check_focus)
   client.connect_signal("property::hidden", clear_focus)
   client.connect_signal("property::minimized", clear_focus)

Remove unwanted focus changing key bindings
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The mouse bindings

.. code:: lua

   awful.button({ }, 4, awful.tag.viewnext),
   awful.button({ }, 5, awful.tag.viewprev)

in the default *rc.lua* may cause tag and thus focus changes without
keyboard interaction and tend to happen accidentally. This doesn’t suit
our definition from above and should therefore be removed or commented
out.

Adjust client rules
^^^^^^^^^^^^^^^^^^^

The default client rule allows certain focus changes via
``focus = awful.client.focus.filter``. These changes can be prevented
entirely by setting ``focus = false``.

Alternatively users may provide their own focus filter functions.

Disable sloppy focus
^^^^^^^^^^^^^^^^^^^^

In your *rc.lua* you’ll find a section such as

.. code:: lua

       -- Enable sloppy focus
       c:connect_signal("mouse::enter", function(c)
           if awful.layout.get(c.screen) ~= awful.layout.suit.magnifier
               and awful.client.focus.filter(c) then
               client.focus = c
           end
       end)

These enable *sloppy focus* aka focus changes on mouse movements
(without clicking) and should be removed or commented out to disable
that behaviour.

Ignore requests from applications to the window manager
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Handling of such requests is currently mostly implemented by AwesomeWM
in the file ``/usr/share/awesome/lib/awful/ewmh.lua``. You can either
comment out the respective ``client.connect_singal()`` lines in that
file (it will change back after each AwesomeWM update though) or
disconnect the signals in your *rc.lua*.

As of AwesomeWM 3.5.9 this however is apparently only possible for
signals connected to global functions, i.e. currently only the below
signals can be disconnected in the *rc.lua*:

.. code:: lua

   local ewmh = require("awful.ewmh")

   client.disconnect_signal("request::activate", ewmh.activate)
   client.disconnect_signal("request::tag", ewmh.tag)

The signal names may change across AwesomeWM versions.
