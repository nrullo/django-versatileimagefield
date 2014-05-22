=============================================
Specifying a Primary Point of Interest (PPOI)
=============================================

The :ref:`crop Sizer<crop-sizer>` is super-useful for creating images at a specific
size/aspect-ratio however, sometimes you want the 'crop centerpoint' to
be somewhere other than the center of a particular image. In fact, the
initial inspiration for ``django-versatileimagefield`` came as a result
of tackling this very problem.

PIL's
`ImageOps.fit <http://pillow.readthedocs.org/en/latest/reference/ImageOps.html#PIL.ImageOps.fit>`__
method (by `Kevin Cazabon <http://www.cazabon.com/>`__) is what powers
the image manipulation of the ``crop`` Sizer and it takes an optional
keyword argument, ``centering``, which expects a 2-tuple comprised of
floats which are greater than or equal to 0 and less than or equal to 1. These two values
together form a cartesian coordinate system that dictates where to
center the crop:

+----------+--------------+--------------+--------------+
|          |Left          |Center        |Right         |
+==========+==============+==============+==============+
|**Top**   |``(0.0, 0.0)``|``(0.0, 0.5)``|``(0.0, 1.0)``|
+----------+--------------+--------------+--------------+
|**Middle**|``(0.5, 0.0)``|``(0.5, 0.5)``|``(0.5, 1.0)``|
+----------+--------------+--------------+--------------+
|**Bottom**|``(1.0, 0.0)``|``(1.0, 0.5)``|``(1.0, 1.0)``|
+----------+--------------+--------------+--------------+

.. note:: At present, only the ``crop`` Sizer changes how it creates images
    based on PPOI but a ``VersatileImageField`` makes its PPOI value
    available to ALL its attached Filters and Sizers. Get creative!

The PPOIField
=============

Each image managed by a ``VersatileImageField`` can store its own,
unique PPOI in the database via the easy-to-use ``PPOIField``. Here's
how to integrate it into our example model (relevant lines highlighted in the code block below):

.. code-block:: python
    :emphasize-lines: 4,5,17,29,30,31

    # models.py with `VersatileImageField` & `PPOIField`
    from django.db import models

    from versatileimagefield.fields import VersatileImageField, \
        PPOIField

    class ImageExampleModel(models.Model):
        name = models.CharField(
            'Name',
            max_length=80
        )
        image = VersatileImageField(
            'Image',
            upload_to='images/testimagemodel/',
            width_field='width',
            height_field='height',
            ppoi_field='ppoi'
        )
        height = models.PositiveIntegerField(
            'Image Height',
            blank=True,
            null=True
        )
        width = models.PositiveIntegerField(
            'Image Width',
            blank=True,
            null=True
        )
        ppoi = PPOIField(
            'Image PPOI'
        )

        class Meta:
            verbose_name = 'Image Example'
            verbose_name_plural = 'Image Examples'

As you can see, you'll need to add a new ``PPOIField`` field to your
model and then include the name of that field in the
``VersatileImageField``'s ``ppoi_field`` keyword argument. That's it!

.. note:: ``PPOIField`` is fully-compatible with
    `south <http://south.readthedocs.org/en/latest/index.html>`_ so
    migrate to your heart's content!

How PPOI is Stored in the Database
----------------------------------

The **Primary Point of Interest** is stored in the database as a string
with the x and y coordinates limited to two decimal places and separated
by an 'x' (for instance: ``'0.5x0.5'`` or ``'0.62x0.28'``).

Setting PPOI
============

PPOI is set via the ``ppoi`` attribute on a ``VersatileImageField``. You
should **always** set an image's PPOI here (as opposed to directly on a
``PPOIField`` attribute) since a ``VersatileImageField`` will ensure
updated values are passed-down to all its attached Filters & Sizers.

When you save a model instance, ``VersatileImageField`` will ensure its
currently-assigned PPOI value is 'sent' to the ``PPOIField`` associated
with it (if any) prior to writing to the database.

Via The Shell
-------------

.. code-block:: python

    # Importing our example Model
    >>> from someapp.models import ImageExampleModel
    # Retrieving a model instance
    >>> example = ImageExampleModel.objects.all()[0]
    # Retrieving the current PPOI value associated with the image field
    # A `VersatileImageField`'s PPOI value is ALWAYS associated with the `ppoi`
    # attribute, irregardless of what you named the `PPOIField` attribute on your model
    >>> example.image.ppoi
    (0.5, 0.5)
    # Creating a cropped image
    >>> example.image.crop['400x400'].url
    u'/media/__sized__/images/testimagemodel/test-image-crop-c0-5__0-5-400x400.jpg'
    # Changing the PPOI value
    >>> example.image.ppoi = (1, 1)
    # Creating a new cropped image with the new PPOI value
    >>> example.image.crop['400x400'].url
    u'/media/__sized__/images/testimagemodel/test-image-crop-c1__1-400x400.jpg'
    # PPOI values can be set as either a tuple or a string
    >>> example.image.ppoi = '0.1x0.55'
    >>> example.image.ppoi
    (0.1, 0.55)
    >>> example.image.ppoi = (0.75, 0.25)
    >>> example.image.crop['400x400'].url
    u'/media/__sized__/images/testimagemodel/test-image-crop-c0-75__0-25-400x400.jpg'
    # u'0.75x0.25' is written to the database in the 'ppoi' column associated with
    # our example model
    >>> example.save()

As you can see, changing an image's PPOI changes the filename of the
cropped image. This ensures updates to a ``VersatileImageField``'s PPOI
value will result in unique cache entries for each unique image it
creates.

.. note:: Each time a field's PPOI is set, its attached Filters & Sizers will
    be immediately updated with the new value.

.. _ppoi-formfield:

FormField/Admin Integration
================================

It's pretty hard to accurately set a particular image's PPOI when
working in the Python shell so ``django-versatileimagefield`` ships with
an admin-ready formfield. Simply add an image, click 'Save and continue
editing', click where you'd like the PPOI to be and then save your model
instance again. A helpful translucent red square will indicate where the
PPOI value is currently set to on the image:

.. figure:: /_static/images/ppoi-admin-example.png
   :alt: django-versatileimagefield PPOI admin widget example

   django-versatileimagefield PPOI admin widget example