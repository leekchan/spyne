
.. _manual-sqlalchemy:

SQLAlchemy Integration
======================

This tutorial builds on the :ref:`manual-user-manager` tutorial. If you haven't
done so, we recommended you to read it first.

In this tutorial, we talk about using Spyne tools that make it easy to deal with
database-related operations. We will show how to integrate SQLAlchemy and Spyne
object definitions, and how to do painless transaction management using Spyne events.

The full example is available here: http://github.com/arskom/spyne/blob/master/examples/user_manager/server_sqlalchemy.py

::

    import logging
    logging.basicConfig(level=logging.DEBUG)
    logging.getLogger('spyne.protocol.xml').setLevel(logging.DEBUG)
    logging.getLogger('sqlalchemy.engine.base.Engine').setLevel(logging.DEBUG)

    import sqlalchemy

    from sqlalchemy import create_engine
    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy.orm import sessionmaker

    from sqlalchemy import MetaData
    from sqlalchemy import Column

    from spyne.application import Application
    from spyne.decorator import rpc
    from spyne.interface.wsdl import Wsdl11
    from spyne.protocol.soap import Soap11
    from spyne.model.complex import Iterable
    from spyne.model.primitive import Integer
    from spyne.model.table import TableSerializer
    from spyne.server.wsgi import WsgiApplication
    from spyne.service import ServiceBase

    _user_database = create_engine('sqlite:///:memory:')
    metadata = MetaData(bind=_user_database)
    DeclarativeBase = declarative_base(metadata=metadata)
    Session = sessionmaker(bind=_user_database)

    class User(TableSerializer, DeclarativeBase):
        __namespace__ = 'spyne.examples.user_manager'
        __tablename__ = 'spyne_user'

        user_id = Column(sqlalchemy.Integer, primary_key=True)
        user_name = Column(sqlalchemy.String(256))
        first_name = Column(sqlalchemy.String(256))
        last_name = Column(sqlalchemy.String(256))

    # this is the same as the above user object. Use this method of declaring
    # objects for tables that have to be defined elsewhere.
    class AlternativeUser(TableSerializer, DeclarativeBase):
        __namespace__ = 'spyne.examples.user_manager'
        __table__ = User.__table__

    class UserManagerService(ServiceBase):
        @rpc(User, _returns=Integer)
        def add_user(ctx, user):
            ctx.udc.session.add(user)
            ctx.udc.session.flush()

            return user.user_id

        @rpc(Integer, _returns=User)
        def get_user(ctx, user_id):
            return ctx.udc.session.query(User).filter_by(user_id=user_id).one()

        @rpc(User)
        def set_user(ctx, user):
            ctx.udc.session.merge(user)

        @rpc(Integer)
        def del_user(ctx, user_id):
            ctx.udc.session.query(User).filter_by(user_id=user_id).delete()

        @rpc(_returns=Iterable(AlternativeUser))
        def get_all_user(ctx):
            return ctx.udc.session.query(User)

    class UserDefinedContext(object):
        def __init__(self):
            self.session = Session()

        def __del__(self):
            self.session.close()

    def _on_method_call(ctx):
        ctx.udc = UserDefinedContext()

    def _on_method_return_object(ctx):
        # we don't do this in UserDefinedContext.__del__ simply to be able to alert
        # the client in case the commit fails.
        ctx.udc.session.commit()

    application = Application([UserManagerService], 'spyne.examples.user_manager',
                interface=Wsdl11(), in_protocol=Soap11(), out_protocol=Soap11())

    application.event_manager.add_listener('method_call', _on_method_call)
    application.event_manager.add_listener('method_return_object', _on_method_return_object)

    if __name__=='__main__':
        try:
            from wsgiref.simple_server import make_server
        except ImportError:
            print "Error: example server code requires Python >= 2.5"

        wsgi_app = WsgiApplication(application)
        server = make_server('127.0.0.1', 7789, wsgi_app)

        metadata.create_all()
        print "listening to http://127.0.0.1:7789"
        print "wsdl is at: http://localhost:7789/?wsdl"

        server.serve_forever()

Again, focusing on what's different from previous :ref:`manual-user-manager`
example: ::

    class User(TableModel, DeclarativeBase):
        __namespace__ = 'spyne.examples.user_manager'
        __tablename__ = 'spyne_user'

        user_id = Column(sqlalchemy.Integer, primary_key=True)
        user_name = Column(sqlalchemy.String(256))
        first_name = Column(sqlalchemy.String(256))
        last_name = Column(sqlalchemy.String(256))

Defined this way, SQLAlchemy objects are regular Spyne objects that can be used
anywhere the regular Spyne types go. The definition for the `User` object is
quite similar to vanilla SQLAlchemy declarative syntax, save for two elements:

    #. The object also bases on :class:`spyne.model.table.TableModel`, which
       bridges SQLAlchemy and Spyne types.
    #. It has a namespace declaration, which is just so the service looks good
       on wsdl.

The SQLAlchemy integration is far from perfect at the moment:

    * SQL constraints are not reflected to the interface document.
    * It's not possible to define additional schema constraints.
    * Object attributes defined by mechanisms other than Column and a limited
      form of `relationship` (no string arguments) are not supported.

If you need any of the above features, you need to separate the spyne and
sqlalchemy object definitions.

Spyne makes it easy to an extent with the following syntax: ::

    class AlternativeUser(TableSerializer, DeclarativeBase):
        __namespace__ = 'spyne.examples.user_manager'
        __table__ = User.__table__

Here, The AlternativeUser object is automatically populated using columns from
the table definition.

The context object is also a little bit different -- we start a transaction for
every call in the constructor of the UserDefinedContext object, and close it in
its destructor: ::

    class UserDefinedContext(object):
        def __init__(self):
            self.session = Session()

        def __del__(self):
            self.session.close()

We implement an event handler that instantiates the UserDefinedContext object
for every method call: ::

    def _on_method_call(ctx):
        ctx.udc = UserDefinedContext()

We also implement an event handler that commits the transaction once the method
call is complete. ::

    def _on_method_return_object(ctx):
        ctx.udc.session.commit()

We register those handlers to the application's 'method_call' handler: ::

    application.event_manager.add_listener('method_call', _on_method_call)
    application.event_manager.add_listener('method_return_object', _on_method_return_object)

Note that the ``method_return_object`` event is only fired when the method call
completes without throwing any exceptions.

What's next?
------------

This tutorial walks you through most of what you need to know to expose your
services. You can read the :ref:`manual-metadata` section where service metadata
management apis are introduced.

Otherwise, you can refer to the reference of the documentation or the mailing
list if you have further questions.
