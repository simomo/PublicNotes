# How to test private method
```java
Method method = CommonDataJob.class.getDeclaredMethod("parseTableFilterCondition", String.class);
method.setAccessible(true);

List<String> r1 = (List<String>) method.invoke(commonDataJob, "all");
Assert.assertEquals(r1.size(), 0);
```

# How to set private field
```java
private void setCurCacheNum(Integer curCacheNum) throws NoSuchFieldException, IllegalAccessException {
    new Expectations() {{
        Field f = LocalCacheInstanceId2Mac.class.getDeclaredField("curCacheNum");
        f.setAccessible(true);
        f.set(localCacheInstanceId2Mac, curCacheNum);
    }};
}
```

# How to verify param
```java
@Test
public void parseNodeFilterConditionTest2() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
    new Expectations() {{
        interfaceDoMapper.selectByExample((InterfaceDoExample) any);
        result = Arrays.asList(new InterfaceDo(), new InterfaceDo());
        times = 1;
    }};

    Method method = CommonDataJob.class.getDeclaredMethod("parseNodeFilterCondition", String.class);
    method.setAccessible(true);

    List<InterfaceDo> r1 = (List<InterfaceDo>) method.invoke(commonDataJob, "enabled");
    Assert.assertEquals(r1.size(), 2);

    new Verifications() {{
        InterfaceDoExample interfaceDoExample;
        interfaceDoMapper.selectByExample(interfaceDoExample = withCapture());
        // verify interfaceDoExample, e.g. Assert
    }};
}
```

# How to verify exception
```java
try {
    method.invoke(commonDataJob, "[table1, table2,table3 , table4 ");
} catch (InvocationTargetException e) {
    Assert.assertEquals("tableFilterCondition must be either 'all' or '[tableName1,tableName2]'", e.getCause().getMessage());
}
```

# How to mock private method
```java
public class CommonDataJobTest {
    @Tested(fullyInitialized = true)
    private CommonDataJob commonDataJob;
    @Test
	public void executeTest1(@Mocked JobExecutionContext jobExecutionContext) throws JobExecutionException {
        new MockUp<CommonDataJob> () {
            @Mock void syncDataComplexMode(List<InterfaceDo> dos) {}
        };
        new Expectations() {{
            jobExecutionContext.getJobDetail();
            JobDetail jobDetail = new JobDetailImpl();
            JobDataMap jobDataMap = jobDetail.getJobDataMap();
            jobDataMap.put("filterMode", JobFactory.JobMode.COMPLEX.getValue());
            result = jobDetail;
            times = 1;
        }};
        commonDataJob.execute(jobExecutionContext);
    }
}
```

# How to verify mybatis query condition (the example object)
```java
new Verifications() {{
    InterfaceDoExample interfaceDoExample;
    interfaceDoMapper.selectByExample(interfaceDoExample = withCapture());
    List<InterfaceDoExample.Criteria> criterias = interfaceDoExample.getOredCriteria();
    Assert.assertEquals(criterias.size(), 1);

    InterfaceDoExample.Criteria criteria1 = criterias.get(0);
    Assert.assertEquals(criteria1.getAllCriteria().get(0).getCondition(), "is_management =");
    Assert.assertEquals(criteria1.getAllCriteria().get(0).getValue(), true);

    Assert.assertEquals(criteria1.getAllCriteria().get(1).getCondition(), "status =");
    Assert.assertEquals(criteria1.getAllCriteria().get(1).getValue(), true);
}};
```