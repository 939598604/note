# optional



## 2.optional不推荐使用

```
String a=new String("a");
Optional<String> optional=Optional.ofNullable(a);
if(optional.isPresent()){
    optional.get();
}
```

## 3.optional推荐使用

optional一般使用函数式的方式，传统的方式不建议使用

```
String str="a";
Optional<String> optional=Optional.ofNullable(str);
optional.ifPresent(s->System.out.println(s));
```

## 4.optional漂亮的代码

```
Emplooy emplooy1 = new Emplooy("zhangsan");
Emplooy emplooy2 = new Emplooy("lisi");

List<Emplooy> emplooys = Arrays.asList(emplooy1, emplooy2);

Company company = new Company("思创",emplooys);
Optional<Company> optional=Optional.ofNullable(company);
System.out.println(
    optional.map(
    	com->(com.getEmplooys())
    ).orElse(
    	Collections.emptyList()
    )
);
```

