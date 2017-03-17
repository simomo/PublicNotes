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