# How to verify log entries (logback)
Basic idea is we create a mocked appender, and it to root logger, so that the mock can capture all log entries.
```java
@Test
public void shouldWarnWhenStartHoverflyInstanceTwice() {
    // Given
    // Reference: https://dzone.com/articles/unit-testing-asserting-line
    ch.qos.logback.classic.Logger root = (ch.qos.logback.classic.Logger) LoggerFactory.getLogger(ch.qos.logback.classic.Logger.ROOT_LOGGER_NAME);
    final Appender mockAppender = mock(Appender.class);
    when(mockAppender.getName()).thenReturn("test-shouldWarnWhenStartHoverflyInstanceTwice");
    root.addAppender(mockAppender);

    startDefaultHoverfly();

    // when
    hoverfly.start();

    // then
    verify(mockAppender).doAppend(argThat(new ArgumentMatcher() {
        @Override
        public boolean matches(final Object argument) {
            LoggingEvent event = (LoggingEvent) argument;
            boolean r = event.getLevel().levelStr.equals("WARN") &&
                    event.getMessage().contains("Local Hoverfly is already running");
            return r;
        }
    }));
}
```

# How to mock/spy dependencies of beans
## Differences between `@MockBean` and `@SpyBean`
Both `@MockBean` and `@SpyBean` intercept a "real" bean, so that you can mock some method of beans.
For those methods you don't mock, mocked bean will return null, and spied bean will call the methods
of the underlay "real" bean.
## How to use `@MockBean` and `@SpyBean`
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = TestApplication.class)
@FixMethodOrder(value = MethodSorters.NAME_ASCENDING)
public class InstanceServiceTest {
    
    @Autowired
    private TheServiceYouWantToTest theServiceYouWantToTest; 

    @MockBean
    private ADependencyOfAboveService aDependencyOfAboveService;

    @SpyBean
    private AnotherDependencyOfAboveService anotherDependencyOfAboveService;

    @Test
    public void testOneMethod() {
        when(aDependencyOfAboveService.method1()).thenReturn(someThing);
        when(anotherDependencyOfAboveService.method1()).thenReturn(anotherThing);

        /**
         * theOneMethod will call:
         *   - aDependencyOfAboveService.method1  <---- return someThing
         *   - anotherDependencyOfAboveService.method1  <---- return anotherThing
         *   - anotherDependencyOfAboveService.methodNotMocked  <---- will be called
         */
        theServiceYouWantToTest.theOneMethod();

        verify(aDependencyOfAboveService, times(1)).method1();
        verify(anotherDependencyOfAboveService, times(1)).method1();
        verify(anotherDependencyOfAboveService, times(1)).methodNotMocked();  // <---- un-mocked method call can also be verified
    }
}
```