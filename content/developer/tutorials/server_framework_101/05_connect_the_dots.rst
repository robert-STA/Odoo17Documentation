===========================
Chapter 5: Connect the dots
===========================

In this chapter, we'll add business logic to the models to automate the processes of our application
and turn it into a dynamic and useful tool. This will involve defining actions, constraints,
automatic computations, and other model methods.

.. todo: explain the env (self.env.uid, self.env.user, self.env.ref(xml_id), self.env[model_name])
.. todo: explain magic commands
.. todo: 6,0,0 to associate tags to properties in data
.. todo: create (create offer -> offer received state) and write methods
.. todo: auto-update property state based on received offers state (write)

.. _tutorials/server_framework_101/computed_fields:

Automate field computations
===========================

So far, we have built an object model capable of handling the essential data for a real estate
business. However, requiring users to manually set every field value can be inconvenient, especially
when some values are derived from other fields. Fortunately, the server framework provides the
ability to define **computed fields**.

Computed fields are a special type of field whose value are derived through programmatic computation
rather than stored directly in the database. The server computes these values on the fly whenever
the field is accessed. This makes computed fields highly convenient for tasks such as displaying
calculation results in views, automating repetitive processes, or assisting users during data entry.

In Odoo, computed fields are implemented by defining a Python method and linking it to a field
declaration using the `compute` argument. This method operates on a **recordset** :dfn:`a collection
of records from the same model` accessible through the `self` argument. The method must iterate over
the records to compute and set the field's value for each one. Additionally, compute methods must be
decorated with the :code:`@api.depends()` decorator when they depend on other fields. This decorator
specifies the fields that trigger an automatic recomputation whenever their values change, ensuring
that computed fields remain consistent.

.. example::
   In the example below, the computed `margin`, `is_profitable` and `breadcrumb` fields are added to
   the `product` model.

   .. code-block:: python

      from odoo import api, fields, models


      class Product(models.Model):
          _name = 'product'

          name = fields.Char(string="Name", required=True)
          price = fields.Float(string="Sales Price", required=True)
          cost = fields.Float(string="Manufacturing Cost")
          margin = fields.Float(string="Profit Margin", compute='_compute_margin')
          is_profitable = fields.Boolean(string="Profitable", compute='_compute_is_profitable')
          category_id = fields.Many2one(
              string="Category", comodel_name='product.category', ondelete='restrict', required=True
          )
          breadcrumb = fields.Char(
              string="Breadcrumb",
              help="The path to the product. For example: 'Home Decor / Coffee table'",
              compute='_compute_breadcrumb',
          )

          @api.depends('price', 'cost')
          def _compute_margin(self):
              for product in self:
                  product.margin = product.price - product.cost

          @api.depends('margin')
          def _compute_is_profitable(self):
              for product in self:
                  product.is_profitable = product.margin > 0

          @api.depends('name', 'category_id.name')
          def _compute_breadcrumb(self):
              for product in self:
                  category = product.category_id
                  product.breadcrumb = f"{category.name} / {product.name}"

   .. note::

      - Compute methods are referenced using their names as strings in the `compute` field argument,
        rather than directly linking the method object. This allows placing them after the field
        declaration.
      - Model methods should be private :dfn:`prefixed with an underscore` to keep them hidden from
        the :doc:`external API <../../reference/external_api>`.
      - Numeric field values default to `0` when not explicitly set.
      - A compute method can depend on another computed field.
      - Field values for related models can be accessed via the `Many2one`, `One2many`, or
        `Many2many` field.
      - Variables used for relational field values are typically not suffixed with `_id` or `_ids`.
        While the field itself represents the stored ID(s), the variable holds the corresponding
        recordset in memory.

.. seealso::
   - :ref:`Reference documentation for computed fields <reference/fields/compute>`
   - :ref:`Reference documentation for recordsets <reference/orm/recordsets>`
   - :ref:`Coding guidelines on naming and ordering the members of model classes
     <contributing/coding_guidelines/model_members>`

Our real estate models can benefit from several computed fields to automate common calculations.
Let's implement them.

.. exercise::
   Add the following fields to the corresponding models and relevant views:

   - **Total Area** (`real.estate.property`): The sum of the floor and garden areas.
   - **Expiry Date** (`real.estate.offer`): The start date offset by the validity period.
   - **Best Offer** (`real.estate.property`): The maximum amount of all offers.

   .. tip::
      - Import the `odoo.tools.date_utils` package to simplify operations on `Date` fields.
      - Use the :meth:`mapped <odoo.models.Model.mapped>` method to extract a recordset's field
        values into a list.

.. spoiler:: Solution

   .. code-block:: python
      :caption: `real_estate_property.py`
      :emphasize-lines: 1,8,13,16-27

      from odoo import api, fields, models

      class RealEstateProperty(models.Model):
          [...]
          garden_area = fields.Integer(
              string="Garden Area", help="The garden area excluding the building."
          )
          total_area = fields.Integer(string="Total Area", compute='_compute_total_area')
          [...]
          offer_ids = fields.One2many(
              string="Offers", comodel_name='real.estate.offer', inverse_name='property_id'
          )
          best_offer_amount = fields.Float(string="Best Offer", compute='_compute_best_offer_amount')
          tag_ids = fields.Many2many(string="Tags", comodel_name='real.estate.tag')

          @api.depends('floor_area', 'garden_area')
          def _compute_total_area(self):
              for property in self:
                  property.total_area = property.floor_area + property.garden_area

          @api.depends('offer_ids.amount')
          def _compute_best_offer_amount(self):
              for property in self:
                  if property.offer_ids:
                      property.best_offer_amount = max(property.offer_ids.mapped('amount'))
                  else:
                      property.best_offer_amount = 0

   .. code-block:: xml
      :caption: `real_estate_property_views.xml`
      :emphasize-lines: 5,15,22

      <record id="real_estate.property_list" model="ir.ui.view">
          [...]
              <list>
                  [...]
                  <field name="total_area" optional="hide"/>
              </list>
          [...]
      </record>

      <record id="real_estate.property_form" model="ir.ui.view">
          [...]
              <group string="Listing Information">
                  <field name="type_id"/>
                  <field name="selling_price"/>
                  <field name="best_offer_amount"/>
                  <field name="availability_date"/>
                  <field name="active"/>
              </group>
              <group string="Building Specifications">
                  [...]
                  <field name="garden_area"/>
                  <field name="total_area"/>
                  [...]
              </group>
          [...]
      </record>

.. _tutorials/server_framework_101/inverse_methods:

Make computed fields editable
-----------------------------

You might have noticed that computed fields are read-only by default. This is expected since their
values are typically determined programmatically rather than set manually by users. However, this
behavior can be limiting when users need to adjust the computed value themselves. **Inverse
methods** address this limitation by allowing edits to computed fields and propagating the changes
back to their dependent fields.

To make a computed field editable, a Python method must be defined and linked to the field
declaration using the `inverse` argument. This method specifies how updates to the computed field
should be applied to its dependencies.

.. seealso::
   :ref:`Reference documentation for related fields <reference/fields/related>`

.. todo: inverse: offer deadline

.. _tutorials/server_framework_101/store_computed_fields:

Store computed fields
---------------------

As computed fields are calculated on the fly, recalculating their values repeatedly can become
inefficient, especially when they are frequently accessed or used in models with large datasets.
Another consequence is that they cannot be used in search queries by default. **Stored computed
fields** address both these issues by saving their values in the database and automatically updating
them only when their dependent data changes. Storing a computed field also enables the database to
index the field's column, significantly improving query performance for large datasets.

Computed fields can be stored in the database by including the `store=True` argument in their field
declaration. The :code:`@api.depends()` decorator ensures that computed fields remain consistent not
only in the cache, but also when they are stored in the database.

However, storing computed fields should be carefully considered. Every update to a dependency
triggers a recomputation, which can significantly impact performance on production servers with a
large number of records.

.. example::
   store `margin`

.. seealso::
   Reference documentation for the :meth:`@api.depends() <odoo.api.depends>` decorator

.. _tutorials/server_framework_101/search_methods:

Search computed fields
----------------------

As mentioned before, computed fields cannot be used in search queries unless they are stored in the
database. This limitation arises because searches are performed at the database level, which is not
aware of the existence of non-stored computed fields. However, storing every field that we wish to
search on would be inefficient. **Search methods** provide a way to overcome this limitation.

To enable searching on a computed field, a Python method must be defined and linked to the field
declaration using the `search` argument. This method receives the search query's `operator` and
`value` and should return a search domain that specifies how the query should filter records. The
domain must be constructed using stored fields only.

.. example::
   In the example below, a search method is added to allow searching on the `is_profitable` field.

   .. code-block:: python

      from odoo import api, fields, models


      class Product(models.Model):
          _name = 'product'

          margin = fields.Float(string="Profit Margin", compute='_compute_margin', store=True)
          is_profitable = fields.Boolean(
              string="Profitable", compute='_compute_is_profitable', search='_search_is_profitable'
          )

          def _search_is_profitable(self, operator, value):
              if (operator == '=' and value is True) or (operator == '!=' and value is False):
                  return [('margin', '>', 0)]
              elif (operator == '=' and value is False) or (operator == '!=' and value is True):
                  return [('margin', '<=', 0)]
              else:
                  raise NotImplementedError()

.. _tutorials/server_framework_101/related_fields:

Simplify related record access
------------------------------

While computed fields make it easier to derive values programmatically, there are cases where the
desired data already exists in related records. Manually computing such values would be redundant
and error-prone. **Related fields** solve this by dynamically fetching data from related records. As
a special case of computed fields, they simplify access to information without requiring explicit
computation.

In practice, related fields are defined like regular fields, but with the `related` argument set to
the path of the related record's field. Related fields can also be stored with the `store=True`
argument, just like regular computed fields.

.. seealso::
   :ref:`Reference documentation for related fields <reference/fields/related>`

.. todo: related fields (buyer's phone)

.. _tutorials/server_framework_101/onchanges:

Provide real-time feedback
==========================

**Onchange methods** are a feature of the server framework designed to respond to changes in field
values directly within the user interface. They are executed when a user modifies a field in a form
view, even before saving the record to the database. This allows for real-time updates of other
fields and provides immediate user feedback, such as blocking user errors, non-blocking warnings, or
suggestions. However, because onchange methods are only triggered by changes made in the UI,
specifically from a form view, they are best suited for assisting with data entry and providing
feedback, rather than implementing core business logic in a module.

In Odoo, onchange methods are implemented as Python methods and linked to one or more fields using
the :code:`@api.onchange()` decorator. These methods are triggered when the specified fields' values
are altered. They operate on the in-memory representation of a single-record recordset received
through `self`. If field values are modified, the changes are automatically reflected in the UI.

.. seealso::
   - Reference documentation for the :meth:`@api.onchange() <odoo.api.onchange>` decorator
   - Reference documentation for the :class:`UserError <odoo.exceptions.UserError>` exception

.. todo: raise UserError + translation
.. todo: if garden checked -> show total area
.. todo: mention that the method is public so it can be called directly by the client.
.. todo: always return something in public methods as they are part of the :ref:external API and can be called through XML-RPC

.. _tutorials/server_framework_101/constraints:

Enforce data integrity
======================

**Constraints** are rules that enforce data integrity by validating field values and relationships
between records. They ensure that the data stored in your application remains consistent and meets
business requirements, preventing invalid values, duplicate entries, or inconsistent relationships
from being saved to the database.

In Odoo, constraints can be implemented at two different levels: directly in the database schema
using **SQL constraints**, or in the model's logic using **Python constraints**. Each type has its
own advantages and use cases, allowing developers to choose the most appropriate validation method
based on their specific needs.

.. _tutorials/server_framework_101/sql_constraints:

SQL constraints
---------------

SQL constraints are database-level rules that are enforced directly by PostgreSQL when records are
created or modified. They are highly efficient in terms of performance, but they cannot handle
complex logic or access individual records. As a result, they are best suited for straightforward
use cases, such as ensuring that a field value is unique or falls within a specific range.

.. todo: Update for https://github.com/odoo/odoo/pull/175783 in 18.1

SQL constraints are defined in the model using the `_sql_constraints` class attribute. This
attribute contains a list of tuples, with each tuple specifying the constraint's name, the SQL
expression to validate, and the error message to display if the constraint is violated.

.. seealso::
   - Reference documentation for the :attr:`_sql_constraints
     <odoo.models.BaseModel._sql_constraints>` class attribute
   - `Reference documentation for PostgreSQL's constraints
     <https://www.postgresql.org/docs/current/ddl-constraints.html>`_

.. todo: price more than zero
.. todo: unique tag constraint

.. _tutorials/server_framework_101/python_constraints:

Python constraints
------------------

Python constraints are record-level rules implemented through Python methods defined on the model.
Unlike SQL constraints, they allow for flexible and context-aware validations based on business
logic, at the expense of higher performance impact than SQL constraints, as they are evaluated
server-side on recordsets. Use cases include ensuring that certain fields align with a specific
condition or that multiple fields work together in a valid combination.

Python constraints are defined in the model as methods decorated with :code:`@api.constrains()`,
which specifies the fields that trigger the validation. These methods are triggered automatically
when a record is created or updated, performing custom validation and raising validation errors if
the constraint is violated.

.. seealso::
   - Reference documentation for the :meth:`@api.constrains <odoo.api.constrains()>` decorator
   - Reference documentation for the :class:`ValidationError <odoo.exceptions.ValidationError>`
     exception

.. todo: accept only one offer

.. _tutorials/server_framework_101/defaults:

Set default field values
========================

When creating new records, pre-filling certain fields with default values can simplify data entry
and reduces the likelihood of errors. Defaults are particularly useful when values are derived from
the system or context, such as the current date, time, or logged-in user.

Fields can be assigned a default value by including the `default` argument in their declaration.
This argument can be set to a static value or dynamically generated using a callable function, such
as a model method or a lambda function. In both cases, the `self` argument provides access to the
environment but does not represent the current record, as no record exists yet during the creation
process.

.. todo: salesperson_id = fields.Many2one(default=lambda self: self.env.user)
.. todo: availability_date = fields.Date(default=lambda self: date_utils.add(fields.Date.today(), months=2))
.. todo: real.estate.offer.amount::default -> property.selling_price (add related?)
.. todo: real.estate.tag.color -> default=_default_color ;  def _default_color(self): return random.randint(1, 11)  (check if lambda works)
.. todo: copy=False on some fields

.. _tutorials/server_framework_101/action_buttons:

Trigger business workflows
==========================

**Action buttons** allow users to trigger specific workflows directly from the user interface. These
buttons can be of type **action**, defined in XML, or **object**, implemented in the model.
Together, these types of buttons facilitate the integration of user interactions with business
logic.

.. todo: "assign myself as salesperson" action
.. todo: "view best offer" statbutton
.. todo: accept/refuse offer buttons
.. todo: accepting offer refuses others
.. todo: action name=...

.. _tutorials/server_framework_101/action_type_actions:

XML-defined actions
-------------------

Action-type buttons link to actions defined in XML and are typically used to display specific views
or trigger server actions. These buttons allow developers to link workflows to the UI without
writing Python code, making them ideal for simple, preconfigured tasks.

We already saw :ref:`how to link XML-defined window actions to menu items
<tutorials/server_framework_101/define_window_actions>`. To link a button to an XML-defined action,
a `button` element must be added to the view, with its `type` attribute set to `action`. The `name`
attribute should reference the XML ID of the action to execute.

.. exercise::
   .. tip::
      Rely on the reference documentation for :ref:`action buttons
      <reference/view_architectures/form/button>` and :ref:`headers
      <reference/view_architectures/form/header>` in form views.

.. _tutorials/server_framework_101/object_type_actions:

Model-defined actions
---------------------

Object-type buttons link to model methods that execute custom business logic. These methods enable
more complex workflows, such as processing the current records, configuring actions depending on
these records, or integrating with external systems.

To link a button to a model-defined action, its `type` attribute must be set to `object`, and its
`name` attribute must be set to the name of the model method to call when the button is clicked. The
method receives the current recordset through `self` and should return a dictionary acting as an
action descriptor.

----

.. todo: add incentive for chapter 6
