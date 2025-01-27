.. _manual-testing:

##################
Testing Dropwizard
##################

.. highlight:: text

.. rubric:: The ``dropwizard-testing`` module provides you with some handy classes for testing
            your :ref:`representation classes <man-core-representations>`
            and :ref:`resource classes <man-core-resources>`. It also provides
            `an extension for JUnit 5.x <https://junit.org/junit5/docs/5.5.0/user-guide/#extensions-overview>`__ and
            `a rule for JUnit 4.x <https://github.com/junit-team/junit4/wiki/Rules>`__.

.. _man-testing-representations:

Testing Representations
=======================

While Jackson's JSON support is powerful and fairly easy-to-use, you shouldn't just rely on
eyeballing your representation classes to ensure you're producing the API you think you
are. By using the helper methods in `FixtureHelpers`, you can add unit tests for serializing and
deserializing your representation classes to and from JSON.

Let's assume we have a ``Person`` class which your API uses as both a request entity (e.g., when
writing via a ``PUT`` request) and a response entity (e.g., when reading via a ``GET`` request):

.. code-block:: java

    public class Person {
        private String name;
        private String email;

        private Person() {
            // Jackson deserialization
        }

        public Person(String name, String email) {
            this.name = name;
            this.email = email;
        }

        @JsonProperty
        public String getName() {
            return name;
        }

        @JsonProperty
        public void setName(String name) {
            this.name = name;
        }

        @JsonProperty
        public String getEmail() {
            return email;
        }

        @JsonProperty
        public void setEmail(String email) {
            this.email = email;
        }

        // hashCode
        // equals
        // toString etc.
    }

.. _man-testing-representations-fixtures:

Fixtures
--------

First, write out the exact JSON representation of a ``Person`` in the
``src/test/resources/fixtures`` directory of your Dropwizard project as ``person.json``:

.. code-block:: javascript

    {
        "name": "Luther Blissett",
        "email": "lb@example.com"
    }

.. _man-testing-representations-serialization:

Testing Serialization
---------------------

Next, write a test for serializing a ``Person`` instance to JSON:

.. code-block:: java

    import static io.dropwizard.testing.FixtureHelpers.*;
    import static org.assertj.core.api.Assertions.assertThat;
    import io.dropwizard.jackson.Jackson;
    import org.junit.jupiter.api.Test;
    import com.fasterxml.jackson.databind.ObjectMapper;

    public class PersonTest {

        private static final ObjectMapper MAPPER = Jackson.newObjectMapper();

        @Test
        public void serializesToJSON() throws Exception {
            final Person person = new Person("Luther Blissett", "lb@example.com");

            final String expected = MAPPER.writeValueAsString(
                    MAPPER.readValue(fixture("fixtures/person.json"), Person.class));

            assertThat(MAPPER.writeValueAsString(person)).isEqualTo(expected);
        }
    }

This test uses `AssertJ assertions`_ and JUnit_ to test that when a ``Person`` instance is serialized
via Jackson it matches the JSON in the fixture file. (The comparison is done on a normalized JSON
string representation, so formatting doesn't affect the results.)

.. _AssertJ assertions: https://assertj.github.io/doc/#assertj-core-assertions-guide
.. _JUnit: http://www.junit.org/

.. _man-testing-representations-deserialization:

Testing Deserialization
-----------------------

Next, write a test for deserializing a ``Person`` instance from JSON:

.. code-block:: java

    import static io.dropwizard.testing.FixtureHelpers.*;
    import static org.assertj.core.api.Assertions.assertThat;
    import io.dropwizard.jackson.Jackson;
    import org.junit.jupiter.api.Test;
    import com.fasterxml.jackson.databind.ObjectMapper;

    public class PersonTest {

        private static final ObjectMapper MAPPER = Jackson.newObjectMapper();

        @Test
        public void deserializesFromJSON() throws Exception {
            final Person person = new Person("Luther Blissett", "lb@example.com");
            assertThat(MAPPER.readValue(fixture("fixtures/person.json"), Person.class))
                    .isEqualTo(person);
        }
    }


This test uses `AssertJ assertions`_ and JUnit_ to test that when a ``Person`` instance is
deserialized via Jackson from the specified JSON fixture it matches the given object.

.. _man-testing-resources:

Testing Resources
=================

While many resource classes can be tested just by calling the methods on the class in a test, some
resources lend themselves to a more full-stack approach. For these, use ``ResourceExtension``, which
loads a given resource instance in an in-memory Jersey server:

.. _man-testing-resources-example:

.. code-block:: java

    import io.dropwizard.testing.junit5.DropwizardExtensionsSupport;
    import io.dropwizard.testing.junit5.ResourceExtension;
    import org.junit.jupiter.api.*;
    import javax.ws.rs.core.Response;
    import java.util.Optional;
    import static org.assertj.core.api.Assertions.assertThat;
    import static org.mockito.Mockito.*;

    @ExtendWith(DropwizardExtensionsSupport.class)
    class PersonResourceTest {
        private static final PersonDAO DAO = mock(PersonDAO.class);
        private static final ResourceExtension EXT = ResourceExtension.builder()
                .addResource(new PersonResource(DAO))
                .build();
        private Person person;

        @BeforeEach
        void setup() {
            person = new Person();
            person.setId(1L);
        }

        @AfterEach
        void tearDown() {
            reset(DAO);
        }

        @Test
        void getPersonSuccess() {
            when(DAO.findById(1L)).thenReturn(Optional.of(person));

            Person found = EXT.target("/people/1").request().get(Person.class);

            assertThat(found.getId()).isEqualTo(person.getId());
            verify(DAO).findById(1L);
        }

        @Test
        void getPersonNotFound() {
            when(DAO.findById(2L)).thenReturn(Optional.empty());
            final Response response = EXT.target("/people/2").request().get();

            assertThat(response.getStatusInfo().getStatusCode()).isEqualTo(Response.Status.NOT_FOUND.getStatusCode());
            verify(DAO).findById(2L);
        }
    }


Instantiate a ``ResourceExtension`` using its ``Builder`` and add the various resource instances you
want to test via ``ResourceExtension.Builder#addResource(Object)``. Use the ``@ExtendWith(DropwizardExtensionsSupport.class)`` annotation on the class to tell Dropwizard to find any field of type ``ResourceExtension``.

In your tests, use ``#target(String path)``, which initializes a request to talk to and test
your instances.

This doesn't require opening a port, but ``ResourceExtension`` tests will perform all the serialization,
deserialization, and validation that happens inside of the HTTP process.

This also doesn't require a full integration test. In the above
:ref:`example <man-testing-resources-example>`, a mocked ``PeopleStore`` is passed to the
``PersonResource`` instance to isolate it from the database. Not only does this make the test much
faster, but it allows your resource unit tests to test error conditions and edge cases much more
easily.

.. hint::

    You can trust ``PeopleStore`` works because you've got working unit tests for it, right?

Default Exception Mappers
-------------------------

By default, a ``ResourceExtension`` will register all the default exception mappers (this behavior is new in 1.0). If
``registerDefaultExceptionMappers`` in the configuration yaml is planned to be set to ``false``,
``ResourceExtension.Builder#setRegisterDefaultExceptionMappers(boolean)`` will also need to be set to ``false``. Then,
all custom exception mappers will need to be registered on the builder, similarly to how they are registered in an
``Application`` class.

Test Containers
---------------

Note that the in-memory Jersey test container does not support all features, such as the ``@Context`` injection.
A different `test container`__ can be used via
``ResourceExtension.Builder#setTestContainerFactory(TestContainerFactory)``.

For example, if you want to use the `Grizzly`_ HTTP server (which supports ``@Context`` injections) you need to add the
dependency for the Jersey Test Framework providers to your Maven POM and set ``GrizzlyWebTestContainerFactory`` as
``TestContainerFactory`` in your test classes.

.. code-block:: xml

    <dependency>
        <groupId>org.glassfish.jersey.test-framework.providers</groupId>
        <artifactId>jersey-test-framework-provider-grizzly2</artifactId>
        <scope>test</scope>
    </dependency>


.. code-block:: java

    @ExtendWith(DropwizardExtensionsSupport.class)
    class ResourceTestWithGrizzly {
        private static final ResourceExtension EXT = ResourceExtension.builder()
                .setTestContainerFactory(new GrizzlyWebTestContainerFactory())
                .addResource(new ExampleResource())
                .build();

        @Test
        void testResource() {
            assertThat(EXT.target("/example").request()
                .get(String.class))
                .isEqualTo("example");
        }
    }

.. __: https://jersey.github.io/documentation/latest/test-framework.html
.. _Grizzly: https://javaee.github.io/grizzly/

.. _man-testing-clients:

Testing Client Implementations
==============================

To avoid circular dependencies in your projects or to speed up test runs, you can test your HTTP client code
by writing a JAX-RS resource as test double and let the ``DropwizardClientExtension`` start and stop a simple Dropwizard
application containing your test doubles.

.. _man-testing-clients-example:

.. code-block:: java

    @ExtendWith(DropwizardExtensionsSupport.class)
    class CustomClientTest {
        @Path("/ping")
        public static class PingResource {
            @GET
            public String ping() {
                return "pong";
            }
        }

        private static final DropwizardClientExtension EXT = new DropwizardClientExtension(new PingResource());

        @Test
        void shouldPing() throws IOException {
            final URL url = new URL(EXT.baseUri() + "/ping");
            final String response = new BufferedReader(new InputStreamReader(url.openStream())).readLine();
            assertEquals("pong", response);
        }
    }

.. hint::

    Of course you would use your HTTP client in the ``@Test`` method and not ``java.net.URL#openStream()``.

The ``DropwizardClientExtension`` takes care of:

* Creating a simple default configuration.
* Creating a simplistic application.
* Adding a dummy health check to the application to suppress the startup warning.
* Adding your JAX-RS resources (test doubles) to the Dropwizard application.
* Choosing a free random port number (important for running tests in parallel).
* Starting the Dropwizard application containing the test doubles.
* Stopping the Dropwizard application containing the test doubles.


Integration Testing
===================

It can be useful to start up your entire application and hit it with real HTTP requests during testing.
The ``dropwizard-testing`` module offers helper classes for your easily doing so.
The optional ``dropwizard-client`` module offers more helpers, e.g. a custom JerseyClientBuilder,
which is aware of your application's environment.

JUnit 5
-------
Adding ``DropwizardExtensionsSupport`` annotation and ``DropwizardAppExtension`` extension to your JUnit5 test class will start the app prior to any tests
running and stop it again when they've completed (roughly equivalent to having used ``@BeforeAll`` and ``@AfterAll``).
``DropwizardAppExtension`` also exposes the app's ``Configuration``,
``Environment`` and the app object itself so that these can be queried by the tests.

If you don't want to use the ``dropwizard-client`` module or find it excessive for testing, you can get access to
a Jersey HTTP client by calling the `client` method on the extension. The returned client is managed by the extension
and can be reused across tests.

.. code-block:: java

    @ExtendWith(DropwizardExtensionsSupport.class)
    class LoginAcceptanceTest {

        private static DropwizardAppExtension<TestConfiguration> EXT = new DropwizardAppExtension<>(
                MyApp.class,
                ResourceHelpers.resourceFilePath("my-app-config.yaml")
            );

        @Test
        void loginHandlerRedirectsAfterPost() {
            Client client = EXT.client();

            Response response = client.target(
                     String.format("http://localhost:%d/login", EXT.getLocalPort()))
                    .request()
                    .post(Entity.json(loginForm()));

            assertThat(response.getStatus()).isEqualTo(302);
        }
    }

JUnit 4
-------
Adding ``DropwizardAppRule`` to your JUnit4 test class will start the app prior to any tests
running and stop it again when they've completed (roughly equivalent to having used ``@BeforeClass`` and ``@AfterClass``).
``DropwizardAppRule`` also exposes the app's ``Configuration``,
``Environment`` and the app object itself so that these can be queried by the tests.

If you don't want to use the ``dropwizard-client`` module or find it excessive for testing, you can get access to
a Jersey HTTP client by calling the `client` method on the rule. The returned client is managed by the rule
and can be reused across tests.

.. code-block:: java

    public class LoginAcceptanceTest {

        @ClassRule
        public static final DropwizardAppRule<TestConfiguration> RULE =
                new DropwizardAppRule<>(MyApp.class, ResourceHelpers.resourceFilePath("my-app-config.yaml"));

        @Test
        public void loginHandlerRedirectsAfterPost() {
            Client client = RULE.client();

            Response response = client.target(
                     String.format("http://localhost:%d/login", RULE.getLocalPort()))
                    .request()
                    .post(Entity.json(loginForm()));

            assertThat(response.getStatus()).isEqualTo(302);
        }
    }

.. warning::

    Resource classes are used by multiple threads concurrently. In general, we recommend that
    resources be stateless/immutable, but it's important to keep the context in mind.


Non-JUnit
---------
By creating a DropwizardTestSupport instance in your test you can manually start and stop the app in your tests, you do this by calling its ``before`` and ``after`` methods. ``DropwizardTestSupport`` also exposes the app's ``Configuration``, ``Environment`` and the app object itself so that these can be queried by the tests.

.. code-block:: java

    public class LoginAcceptanceTest {

        public static final DropwizardTestSupport<TestConfiguration> SUPPORT =
                new DropwizardTestSupport<TestConfiguration>(MyApp.class,
                    ResourceHelpers.resourceFilePath("my-app-config.yaml"),
                    ConfigOverride.config("server.applicationConnectors[0].port", "0") // Optional, if not using a separate testing-specific configuration file, use a randomly selected port
                );

        @BeforeAll
        public void beforeClass() {
            SUPPORT.before();
        }

        @AfterAll
        public void afterClass() {
            SUPPORT.after();
        }

        @Test
        public void loginHandlerRedirectsAfterPost() {
            Client client = new JerseyClientBuilder(SUPPORT.getEnvironment()).build("test client");

            Response response = client.target(
                     String.format("http://localhost:%d/login", SUPPORT.getLocalPort()))
                    .request()
                    .post(Entity.json(loginForm()));

            assertThat(response.getStatus()).isEqualTo(302);
        }
    }

.. _man-testing-commands:

Testing Commands
================

:ref:`Commands <man-core-commands>` can and should be tested, as it's important to ensure arguments
are interpreted correctly, and the output is as expected.

Below is a test for a command that adds the arguments as numbers and outputs the summation to the
console. The test ensures that the result printed to the screen is correct by capturing standard out
before the command is ran.

.. code-block:: java

    class CommandTest {
        private final PrintStream originalOut = System.out;
        private final PrintStream originalErr = System.err;
        private final InputStream originalIn = System.in;

        private final ByteArrayOutputStream stdOut = new ByteArrayOutputStream();
        private final ByteArrayOutputStream stdErr = new ByteArrayOutputStream();
        private Cli cli;

        @BeforeEach
        void setUp() throws Exception {
            // Setup necessary mock
            final JarLocation location = mock(JarLocation.class);
            when(location.getVersion()).thenReturn(Optional.of("1.0.0"));

            // Add commands you want to test
            final Bootstrap<MyConfiguration> bootstrap = new Bootstrap<>(new MyApplication());
            bootstrap.addCommand(new MyAddCommand());

            // Redirect stdout and stderr to our byte streams
            System.setOut(new PrintStream(stdOut));
            System.setErr(new PrintStream(stdErr));

            // Build what'll run the command and interpret arguments
            cli = new Cli(location, bootstrap, stdOut, stdErr);
        }

        @AfterEach
        void teardown() {
            System.setOut(originalOut);
            System.setErr(originalErr);
            System.setIn(originalIn);
        }

        @Test
        void myAddCanAddThreeNumbersCorrectly() {
            final boolean success = cli.run("add", "2", "3", "6");

            SoftAssertions softly = new SoftAssertions();
            softly.assertThat(success).as("Exit success").isTrue();

            // Assert that 2 + 3 + 6 outputs 11
            softly.assertThat(stdOut.toString()).as("stdout").isEqualTo("11");
            softly.assertThat(stdErr.toString()).as("stderr").isEmpty();
            softly.assertAll();
        }
    }

.. _man-testing-database-interactions:

Testing Database Interactions
=============================

In Dropwizard, the database access is managed via the ``@UnitOfWork`` annotation used on resource
methods. In case you want to test database-layer code independently, a ``DAOTestExtension`` is provided
which setups a Hibernate ``SessionFactory``.

.. code-block:: java

    @ExtendWith(DropwizardExtensionsSupport.class)
    public class DatabaseTest {

        public DAOTestExtension database = DAOTestExtension.newBuilder().addEntityClass(FooEntity.class).build();

        private FooDAO fooDAO;

        @BeforeEach
        public void setUp() {
            fooDAO = new FooDAO(database.getSessionFactory());
        }

        @Test
        public void createsFoo() {
            FooEntity fooEntity = new FooEntity("bar");
            long id = database.inTransaction(() -> {
                return fooDAO.save(fooEntity);
            });

            assertThat(fooEntity.getId, notNullValue());
        }

        @Test
        public void roundtripsFoo() {
            long id = database.inTransaction(() -> {
                return fooDAO.save(new FooEntity("baz"));
            });

            FooEntity fooEntity = fooDAO.get(id);

            assertThat(fooEntity.getFoo(), equalTo("baz"));
        }
    }

The ``DAOTestExtension``

* Creates a simple default Hibernate configuration using an H2 in-memory database
* Provides a ``SessionFactory`` instance which can be passed to, e.g., a subclass of ``AbstractDAO``
* Provides a function for executing database operations within a transaction

.. _man-testing-configurations:

Testing Configurations
======================

Configuration objects can be tested for correct deserialization and validation. Using the classes
created in :ref:`polymorphic configurations <man-configuration-polymorphic>` as an example, one can
assert the expected widget is deserialized based on the ``type`` field.

.. code-block:: java

    public class WidgetFactoryTest {

        private final ObjectMapper objectMapper = Jackson.newObjectMapper();
        private final Validator validator = Validators.newValidator();
        private final YamlConfigurationFactory<WidgetFactory> factory =
                new YamlConfigurationFactory<>(WidgetFactory.class, validator, objectMapper, "dw");

        @Test
        public void isDiscoverable() throws Exception {
            // Make sure the types we specified in META-INF gets picked up
            assertThat(new DiscoverableSubtypeResolver().getDiscoveredSubtypes())
                    .contains(HammerFactory.class)
                    .contains(ChiselFactory.class);
        }

        @Test
        public void testBuildAHammer() throws Exception {
            final WidgetFactory wid = factory.build(new ResourceConfigurationSourceProvider(), "yaml/hammer.yml");
            assertThat(wid).isInstanceOf(HammerFactory.class);
            assertThat(((HammerFactory) wid).createWidget().getWeight()).isEqualTo(10);
        }

        // test for the chisel factory
    }

If your configuration file contains environment variables or parameters, some additional
config is required. As an example, we will use ``EnvironmentVariableSubstitutor`` on top of
a simplified version of the above test.

If we have a configuration similar to the following:

.. code-block:: yaml

    widgets:
      - type: hammer
        weight: ${HAMMER_WEIGHT:-20}
      - type: chisel
        radius: 0.4

In order to test this, we would require the following in our test class:

.. code-block:: java

    public class WidgetFactoryTest {

        private final ObjectMapper objectMapper = Jackson.newObjectMapper();
        private final Validator validator = Validators.newValidator();
        private final YamlConfigurationFactory<WidgetFactory> factory =
                new YamlConfigurationFactory<>(WidgetFactory.class, validator, objectMapper, "dw");

        // test for discoverability

        @Test
        public void testBuildAHammer() throws Exception {
            final WidgetFactory wid = factory.build(new SubstitutingSourceProvider(
                    new ResourceConfigurationSourceProvider(),
                    new EnvironmentVariableSubstitutor(false)
                ), "yaml/hammer.yaml");
            assertThat(wid).isInstanceOf(HammerFactory.class);
            assertThat(((HammerFactory) wid).createWidget().getWeight()).isEqualTo(20);
        }

        // test for the chisel factory
    }
