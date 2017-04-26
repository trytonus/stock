================================
Stock Shipment Internal Scenario
================================

Imports::

    >>> import datetime
    >>> from dateutil.relativedelta import relativedelta
    >>> from decimal import Decimal
    >>> from proteus import config, Model, Wizard
    >>> from trytond.modules.company.tests.tools import create_company, \
    ...     get_company
    >>> today = datetime.date.today()
    >>> yesterday = today - relativedelta(days=1)
    >>> tomorrow = today + relativedelta(days=1)

Create database::

    >>> config = config.set_trytond()
    >>> config.pool.test = True

Install stock Module::

    >>> Module = Model.get('ir.module')
    >>> module, = Module.find([('name', '=', 'stock')])
    >>> module.click('install')
    >>> Wizard('ir.module.install_upgrade').execute('upgrade')

Create company::

    >>> _ = create_company()
    >>> company = get_company()

Reload the context::

    >>> User = Model.get('res.user')
    >>> Group = Model.get('res.group')
    >>> config._context = User.get_preferences(True, config.context)

Create product::

    >>> ProductUom = Model.get('product.uom')
    >>> ProductTemplate = Model.get('product.template')
    >>> Product = Model.get('product.product')
    >>> unit, = ProductUom.find([('name', '=', 'Unit')])
    >>> product = Product()
    >>> template = ProductTemplate()
    >>> template.name = 'Product'
    >>> template.default_uom = unit
    >>> template.type = 'goods'
    >>> template.list_price = Decimal('20')
    >>> template.cost_price = Decimal('8')
    >>> template.save()
    >>> product.template = template
    >>> product.save()

Get stock locations::

    >>> Location = Model.get('stock.location')
    >>> lost_found_loc, = Location.find([('type', '=', 'lost_found')])
    >>> storage_loc, = Location.find([('code', '=', 'STO')])
    >>> internal_loc = Location(name='Internal', type='storage')
    >>> internal_loc.save()

Create stock user::

    >>> stock_user = User()
    >>> stock_user.name = 'Stock'
    >>> stock_user.login = 'stock'
    >>> stock_user.main_company = company
    >>> stock_group, = Group.find([('name', '=', 'Stock')])
    >>> stock_user.groups.append(stock_group)
    >>> stock_user.save()

Create Internal Shipment::

    >>> config.user = stock_user.id
    >>> Shipment = Model.get('stock.shipment.internal')
    >>> StockMove = Model.get('stock.move')
    >>> shipment = Shipment()
    >>> shipment.planned_date = today
    >>> shipment.from_location = internal_loc
    >>> shipment.to_location = storage_loc
    >>> move = shipment.moves.new()
    >>> move.product = product
    >>> move.quantity = 1
    >>> move.from_location = internal_loc
    >>> move.to_location = storage_loc
    >>> move.currency = company.currency
    >>> shipment.click('wait')
    >>> shipment.state
    u'waiting'
    >>> shipment.click('assign_try')
    False

Create Internal Shipment from lost_found location::

    >>> lost_found_shipment = Shipment()
    >>> lost_found_shipment.planned_date = today
    >>> lost_found_shipment.company = company
    >>> lost_found_shipment.from_location = lost_found_loc
    >>> lost_found_shipment.to_location = internal_loc
    >>> move = StockMove()
    >>> move = lost_found_shipment.moves.new()
    >>> move.product = product
    >>> move.oum = unit
    >>> move.quantity = 2
    >>> move.from_location = lost_found_loc
    >>> move.to_location = internal_loc
    >>> move.currency = company.currency
    >>> lost_found_shipment.click('wait')
    >>> lost_found_shipment.click('assign_try')
    True
    >>> lost_found_shipment.state
    u'assigned'
    >>> lost_found_shipment.click('done')
    >>> lost_found_shipment.state
    u'done'

Check that now whe can finish the older shipment::

    >>> shipment.click('assign_try')
    True
    >>> shipment.click('done')
    >>> shipment.state
    u'done'

Add lead time inside the warehouse::

    >>> config.user = 1
    >>> LeadTime = Model.get('stock.location.lead_time')
    >>> lead_time = LeadTime()
    >>> lead_time.from_warehouse = storage_loc.warehouse
    >>> lead_time.to_warehouse = storage_loc.warehouse
    >>> lead_time.lead_time = datetime.timedelta(1)
    >>> lead_time.save()

Create Internal Shipment with lead time::

    >>> config.user = stock_user.id
    >>> shipment = Shipment()
    >>> shipment.planned_date = tomorrow
    >>> shipment.from_location = internal_loc
    >>> shipment.to_location = storage_loc
    >>> shipment.planned_start_date == today
    True
    >>> move = shipment.moves.new()
    >>> move.product = product
    >>> move.quantity = 1
    >>> move.from_location = internal_loc
    >>> move.to_location = storage_loc
    >>> shipment.click('wait')
    >>> len(shipment.moves)
    2
    >>> outgoing_move, = shipment.outgoing_moves
    >>> outgoing_move.quantity
    1.0
    >>> outgoing_move.from_location == internal_loc
    True
    >>> outgoing_move.to_location == shipment.transit_location
    True
    >>> incoming_move, = shipment.incoming_moves
    >>> incoming_move.quantity
    1.0
    >>> incoming_move.from_location == shipment.transit_location
    True
    >>> incoming_move.to_location == storage_loc
    True

    >>> shipment.click('assign_try')
    True
    >>> shipment.click('ship')
    >>> shipment.outgoing_moves[0].state
    u'done'
    >>> shipment.click('done')
    >>> shipment.incoming_moves[0].state
    u'done'
