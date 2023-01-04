

```java
@Data
public class Person {
    private String name;  // 姓名
    private int salary; // 薪资
    private int age; // 年龄
    private String sex; //性别
    private String area;  // 地区

    public Person(String name, int salary, String sex, String area) {
        this.name = name;
        this.salary = salary;
        this.sex = sex;
        this.area = area;
    }
}
```

```java
        //对应实体类集合
        List<Person> personList = new ArrayList<Person>();
        personList.add(new Person("Tom", 8900, "male", "New York"));
        personList.add(new Person("Jack", 7000, "male", "Washington"));
        personList.add(new Person("Lily", 7800, "female", "Washington"));
        personList.add(new Person("Anni", 8200, "female", "New York"));
        personList.add(new Person("Owen", 9500, "male", "New York"));
        personList.add(new Person("Alisa", 7900, "female", "New York"));


        //  foreach/find/match
        List<Integer> list = Arrays.asList(7, 6, 9, 3, 8, 2, 1);
        //遍历输出符合条件的元素
        list.stream().filter(x->x>6).forEach(System.out::println);
        //匹配第一个
        Optional<Integer> first = list.stream().filter(x -> x > 6).findFirst();
        //匹配任意（适用于并行流）
        Optional<Integer> any = list.parallelStream().filter(x -> x > 6).findAny();
        //判断是否包含符合特定条件的元素
        System.out.println(list.stream().anyMatch(x -> x > 10));
        //筛选出薪资大于8000的List对象并输出
        List<Person> collect = personList.stream().filter(x -> x.getSalary() > 8000).collect(Collectors.toList());
        collect.forEach(System.out::println);


        //  max/min/count
        List<String> listMMC = Arrays.asList("adnm", "admmt", "pot", "xbangd", "weoujgsd");
        //最长的字符串
        System.out.println(listMMC.stream().max(Comparator.comparing(String::length)).get());
        //获取集合中最大值
        List<Integer> listMax = Arrays.asList(7, 6, 9, 4, 11, 6);
        //自然排序
        System.out.println(listMax.stream().max(Integer::compareTo).get());
        //获取员工工资最高的人
        System.out.println(personList.stream().max(new Comparator<Person>() {
            @Override
            public int compare(Person o1, Person o2) {
                return o1.getSalary() - o2.getSalary();
            }
        }));
        System.out.println(personList.stream().max(Comparator.comparingInt(Person::getSalary)));
        //计算Integer集合中大于6的元素个数
        System.out.println(listMax.stream().filter(x -> x > 6).count());


        //  map/flatMap
        //映射，将一个流的元素按照一定映射规则映射到另一个流中，分为map与flatMap:
        //map:接收一个函数作为参数，该函数将会被应用到每一个元素上，并将其映射成一个新的元素
        //flatMap:接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流

        //将英文字符串数组的元素全部改成大写，整数数组每个元素+3
        String[] strArr = { "abcd", "bcdd", "defde", "fTr" };
        Arrays.asList(strArr).stream().map(String::toUpperCase).collect(Collectors.toList()).forEach(System.out::println);
        List<Integer> intList = Arrays.asList(1, 3, 5, 7, 9, 11);
        intList.stream().map(x->x+3).collect(Collectors.toList()).forEach(System.out::println);
        //员工薪资全部加1000
        //通过创建新的实体类进行更新-不影响原有集合参数
        List<Person> collectNew1 = personList.stream().map(person -> {
            Person personNew = new Person(person.getName(), 0, person.getSex(), person.getArea());
            personNew.setSalary(person.getSalary() + 1000);
            return personNew;
        }).collect(Collectors.toList());
        collectNew1.forEach(System.out::println);
        //在原有的实体类上更新部分参数-原有集合参数也更新了
        List<Person> collectNew2 = personList.stream().map(person -> {
            person.setSalary(person.getSalary() + 1000);
            return person;
        }).collect(Collectors.toList());
        collectNew2.forEach(System.out::println);

        //将两个字符数组合成新的字符数组
        List<String> listNum = Arrays.asList("m,k,l,a", "1,3,5,7");
        listNum.stream().flatMap(s->{
            String[] split = s.split(",");
            Stream<String> stream = Arrays.stream(split);
            return stream;
        }).collect(Collectors.toList()).forEach(System.out::println);




        //归约
        //归约也称缩减，将一个流缩减成一个值，实现对集合求和求乘积和求最值操作
        List<Integer> listN = Arrays.asList(1, 3, 2, 8, 11, 4);
        //求和
        System.out.println(listN.stream().reduce((x, y) -> x + y));
        System.out.println(listN.stream().reduce(Integer::sum));
        //0作为初始值（默认值），当集合为空时会返回这个默认值，当集合不为空时，该值也会参与计算
        System.out.println(listN.stream().reduce(0, Integer::sum));
        //求乘积
        System.out.println(listN.stream().reduce((s, y) -> s * y));
        //求最大值
        System.out.println(listN.stream().reduce((x, y) -> x > y ? x : y));
        System.out.println(listN.stream().reduce(1, Integer::max));
        //求所有员工工资之和和最高工资
        System.out.println(personList.stream().map(Person::getSalary).reduce(Integer::sum));
        System.out.println(personList.stream().reduce(0, (x, y) -> x += y.getSalary(), (sum1, sum2) -> sum1 + sum2));
        System.out.println(personList.stream().reduce(0, (x, y) -> x += y.getSalary(), Integer::sum));
        //求工资最高
        System.out.println(personList.stream().reduce(0, (x, y) -> x > y.getSalary() ? x : y.getSalary(), Integer::max));
        System.out.println(personList.stream().reduce(0, (max, p) -> max > p.getSalary() ? max : p.getSalary(), (max1, max2) -> max1 > max2 ? max1 : max2));


        //收集-主要依赖于stream.Collectors类内置的静态方法
        //  toList/toSet/toMap
        //因为流不存储数据，在流中的数据完成处理后需要将流中的数据重新归集到新的集合中：
        Map<?, Person> map = personList.stream().filter(p -> p.getSalary() > 10000)
                .collect(Collectors.toMap(Person::getName, p -> p));
        System.out.println(map);


        //统计
        //计数：count
        //
        //平均值：averagingInt、averagingLong、averagingDouble
        //
        //最值：maxBy、minBy
        //
        //求和：summingInt、summingLong、summingDouble
        //
        //统计以上所有：summarizingInt、summarizingLong、summarizingDouble

        //求总数
        System.out.println(personList.stream().collect(Collectors.counting()));
        //求平均值
        System.out.println(personList.stream().collect(Collectors.averagingDouble(Person::getSalary)));
        //求最高工资
        System.out.println(personList.stream().map(Person::getSalary).collect(Collectors.maxBy(Integer::compare)));
        //求工资之和
        System.out.println(personList.stream().collect(Collectors.summingInt(Person::getSalary)));
        //统计所有信息
        System.out.println(personList.stream().collect(Collectors.summarizingDouble(Person::getSalary)));


        //分组 partittioningBy/groupingBy
        //分区：将stream按条件分为两个Map
        //分组：将集合分为多个map

        //将员工按薪资是否高于8000分为两部分；
        System.out.println(personList.stream().collect(Collectors.partitioningBy(x -> x.getSalary() > 8000)));
        //将员工按性别分组
        System.out.println(personList.stream().collect(Collectors.groupingBy(Person::getSex)));
        //先按性别分组，再按地区分组
        System.out.println(personList.stream().collect(Collectors.groupingBy(Person::getSex, Collectors.groupingBy(Person::getArea))));


        //接合 joining
        //可以将stream中的元素用特定的连接符连接成一个字符串
        System.out.println(personList.stream().map(p -> p.getName()).collect(Collectors.joining(",")));
        System.out.println(Arrays.asList("A", "B", "C").stream().collect(Collectors.joining("-")));


        //归约 reducing
        //Collectors类提供的reducing相比于stream自身的reduce增加了对自定义归约的支持
        System.out.println(personList.stream().collect(Collectors.reducing(0, Person::getSalary, (i, j) -> (i + j - 5000))));
        //stream的reduce
        System.out.println(personList.stream().map(Person::getSalary).reduce(Integer::sum));


        //排序 sorted
        //sorted()：自然排序,流中需实现Comparable接口
        //sorted(Comparator com)：自定义排序
        //按工资进行升序排列
        System.out.println(personList.stream().sorted(Comparator.comparingInt(Person::getSalary)).map(Person::getName).collect(Collectors.toList()));
        //按工资进行降序排列
        System.out.println(personList.stream().sorted(Comparator.comparing(Person::getSalary).reversed()).map(Person::getName).collect(Collectors.toList()));
        //先按工资再按年龄自定义排序，从大到小
        System.out.println(personList.stream().sorted((p1, p2) -> {
            if (p1.getSalary() == p2.getSalary()) {
                return p2.getAge() - p1.getAge();
            } else {
                return p2.getSalary() - p1.getSalary();
            }
        }).map(Person::getName).collect(Collectors.toList()));



        //提取/组合
        String[] arr1 = { "a", "b", "c", "d" };
        String[] arr2 = { "d", "e", "f", "g" };

        Stream<String> stream1 = Stream.of(arr1);
        Stream<String> stream2 = Stream.of(arr2);
        // concat:合并两个流 distinct：去重
        List<String> newList = Stream.concat(stream1, stream2).distinct().collect(Collectors.toList());
        // limit：限制从流中获得前n个数据
        List<Integer> collect1 = Stream.iterate(1, x -> x + 2).limit(10).collect(Collectors.toList());
        // skip：跳过前n个数据
        List<Integer> collect2 = Stream.iterate(1, x -> x + 2).skip(1).limit(5).collect(Collectors.toList());

        System.out.println("流合并：" + newList);
        System.out.println("limit：" + collect1);
        System.out.println("skip：" + collect2);
```