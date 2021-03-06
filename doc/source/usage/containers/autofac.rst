Configuring Autofac
===================

.. attention:: **This page is obsolete!**

   New documentation is located at http://masstransit-project.com/MassTransit.

   The latest version of this page can be found here_.

.. _here: http://masstransit-project.com/MassTransit/usage/containers/autofac.html

Autofac is a powerful and fast container, and is well supported by MassTransit. Nested lifetime scopes are used
extensively to encapsulate dependencies and ensure clean object lifetime management. The following examples show the
various ways that MassTransit can be configured, including the appropriate interfaces necessary.

.. note::
    Requires NuGets ``MassTransit``, ``MassTransit.AutoFac``, and ``MassTransit.RabbitMQ``

.. note::

    Consumers should not typically depend upon ``IBus`` or ``IBusControl``. A consumer should use the ``ConsumeContext``
    instead, which has all of the same methods as ``IBus``, but is scoped to the receive endpoint. This ensures that
    messages can be tracked between consumers, and are sent from the proper address.

.. sourcecode:: csharp

    using System;
    using System.Reflection;
    using System.Threading.Tasks;
    using Autofac;
    using MassTransit;

    namespace Example
    {
        public class UpdateCustomerAddressConsumer : MassTransit.IConsumer<object>
        {
            public async Task Consume(ConsumeContext<object> context)
            {
                //do stuff
            }
        }

        class Program
        {

            public static void Main(string[] args)
            {
                var builder = new ContainerBuilder();

                // register a specific consumer
                builder.RegisterType<UpdateCustomerAddressConsumer>();

                // just register all the consumers
                builder.RegisterConsumers(Assembly.GetExecutingAssembly());

                builder.Register(context =>
                {
                    var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
                    {
                        var host = cfg.Host(new Uri("rabbitmq://localhost/"), h =>
                        {
                            h.Username("guest");
                            h.Password("guest");
                        });

                        cfg.ReceiveEndpoint("customer_update_queue", ec =>
                        {
                            ec.LoadFrom(context);
                        });
                    });

                    return busControl;
                })
                    .SingleInstance()
                    .As<IBusControl>()
                    .As<IBus>();

                var container = builder.Build();

                var bc = container.Resolve<IBusControl>();
                bc.Start();
            }
        }
    }


Using a Module
--------------

Autofac modules are great for encapsulating configuration, and that is equally true when using MassTransit. An example of
using modules with Autofac is shown below.

.. sourcecode:: csharp

    class ConsumerModule :
        Module
    {
        protected override void Load(ContainerBuilder builder)
        {
            builder.RegisterType<UpdateCustomerAddressConsumer>();

            builder.RegisterType<SqlCustomerRegistry>()
                .As<ICustomerRegistry>();
        }
    }

    class BusModule :
        Module
    {
        protected override void Load(ContainerBuilder builder)
        {
            builder.Register(context =>
            {
                var busSettings = context.Resolve<BusSettings>();

                var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
                {
                    var host = cfg.Host(busSettings.HostAddress, h =>
                    {
                        h.Username(busSettings.Username);
                        h.Password(busSettings.Password);
                    });

                    sbc.ReceiveEndpoint(busSettings.QueueName, ec =>
                    {
                        ec.LoadFrom(context);
                    })
                });
            })
                .SingleInstance()
                .As<IBusControl>()
                .As<IBus>();
        }
    }

    public IContainer CreateContainer()
    {
         var builder = new ContainerBuilder();

        builder.RegisterModule<BusModule>();
        builder.RegisterModule<ConsumerModule>();

        return builder.Build();
    }

    public void CreateContainer()
    {
        _container = new Container(x =>
        {
            x.AddRegistry(new BusRegistry());
            x.AddRegistry(new ConsumerRegistry());
        });
    }

Registering State Machine Sagas
-------------------------------

By using an additional package `MassTransit.Automatonymous.Autofac` you can also register state machine sagas 

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    // register everything else

    // register saga state machines, assuming Saga1 and Saga2 are in different assemblies
    builder.RegisterStateMachineSagas(typeof(Saga1).Assembly, typeof(Saga2).Assembly);

    // registering saga state machines from current assembly
    builder.RegisterStateMachineSagas(Assembly.GetExecutingAssembly());

    // do not forget registering saga repositories (example for NHibernate)
    var mappings = mappingsAssembly
        .GetTypes()
        .Where(t => t.BaseType != null && t.BaseType.IsGenericType &&
            (t.BaseType.GetGenericTypeDefinition() == typeof(SagaClassMapping<>) ||
            t.BaseType.GetGenericTypeDefinition() == typeof(ClassMapping<>)))
        .ToArray();    
    builder.Register(c => new SqlServerSessionFactoryProvider(connString, mappings).GetSessionFactory())
        .As<ISessionFactory>()
        .SingleInstance();
    builder.RegisterGeneric(typeof(NHibernateSagaRepository<>))
        .As(typeof(ISagaRepository<>));

and load them from a contained when configuring the bus.

.. sourcecode:: csharp

    var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
    {
        var host = cfg.Host(busSettings.HostAddress, h =>
        {
            h.Username(busSettings.Username);
            h.Password(busSettings.Password);
        });

        sbc.ReceiveEndpoint(busSettings.QueueName, ec =>
        {
            // loading consumers
            ec.LoadFrom(context);

            // loading saga state machines
            ec.LoadStateMachineSagas(context);
        })
    });
