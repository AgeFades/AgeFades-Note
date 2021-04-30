[TOC]

# 黑马 - SpringDataJpa

## ORM 思想

### 	主要目的

​		操作实体类就相当于操作数据库表

### 	映射关系

​		实体类和表的映射关系

​		实体类中属性和表中字段的映射关系

### 	实现框架

​		MyBatis , Hibernate

## Jpa 注解

### 	类与表

​		@Entity : 声明类与表的映射关系

​		@Table : 配置实体类和表的映射关系

​			name : 表名

### 	属性与字段

​		@Id : 声明主键的配置

​		@GeneratedValue : 配置主键的生成策略

​			IDENTITY : 自增（MySQL）

​				底层数据库必须支持自增

​			SEQUENCE : 序列（Oracle）

​				递增数据库必须支持序列

​			TABLE : 

​				通过一张数据库表的形式帮助我们完成主键自增

​			AUTO :

​				由程序自动的帮助我们选择主键生成策略

​		@Column : 配置属性和字段的映射关系

​			name : 字段名

## 延迟加载与立即加载

### 	延迟加载

​		得到的是一个动态代理对象

​		什么时候用，什么时候才会将数据查出来

​		get* 方法

### 	立即加载

​		一次查询出所有数据

​		find* 方法

## SpringDataJpa

### 	Dao 层

​		JpaRepository<实体类类型，主键类型> :

​			封装了基本 CRUD 操作

​		JpaSpecificationExecutor<实体类类型>	:

​			封装复杂查询（比如 : 分页查询）

### 	执行过程分析

![ZC6DWn.png](https://s2.ax1x.com/2019/06/23/ZC6DWn.png)

### 	JPQL

​		@Query : 查询

​			value : SQL 语句

​		@Modifying : 更新

### 	方法名称规则查询

​		findBy : 根据属性查询（默认等于）

​		findByXxxLike : 根据属性模糊查询

​		findByXxxAndXxx : 多条件查询，用 And | or 连接

### 	动态查询

​		Specification : 查询条件

​			自定义 Specification 实现类

​				root : 查询的根对象（查询的任何属性都可以从根对象中获取）

​				CriteriaQuery : 顶层查询对象，自定义查询方法

​				CriteriaBuilder : 查询的构造器，封装了很多查询条件

```java
private Specification<T> specification(CurrencySearch<T> currencySearch){
        Specification<T> specification = ((root, query, builder) -> {
            List<Predicate> predicateList = new ArrayList<Predicate>();
            //查询条件eq
            if(currencySearch.getQuery() != null && currencySearch.getQuery().size() > 0){
                List<Condition> query1 = currencySearch.getQuery();
                query1.forEach( v -> {
                    if(!StringUtils.isEmpty(v.getField()) && !"string".equals(v.getField())){
                        predicateList.add(builder.equal(root.get(v.getField()),v.getValue()));
                    }
                });
            }
            //循环in条件，将查询条件为in的参数作为查询条件
            if (currencySearch.getIn() != null && currencySearch.getIn().size() > 0) {
                List<InCondition> query1 = currencySearch.getIn();
                query1.forEach(v -> {
                    if (v.getField() != null && v.getField() != "string") {
                        CriteriaBuilder.In<Object> in = builder.in(root.get(v.getField()));
                        v.getValue().forEach(z -> {
                            in.value(z);
                        });
                        predicateList.add(builder.and(in));
                    }
                });
            }
            //查询条件like
            if (currencySearch.getLike() != null && currencySearch.getLike().size() > 0) {
                currencySearch.getLike().forEach(v -> {
                    predicateList.add(builder.and(builder.like(root.get(v.getField()), "%" + (String) v.getValue() + "%")));
                });
            }
            //查询条件not
            if (currencySearch.getNot() != null && currencySearch.getNot().size() > 0) {
                currencySearch.getNot().forEach(v -> {
                    predicateList.add(builder.notEqual(root.get(v.getField()), v.getValue()));

                });
            }
            //查询条件between
            if (currencySearch.getBetween() != null && currencySearch.getBetween().size() > 0) {
                currencySearch.getBetween().forEach(v -> {
                    if (v.getStart() != null) {
                        predicateList.add(builder.ge(root.get(v.getField()), (Number) v.getStart()));
                    }
                    if (v.getEnd() != null) {
                        predicateList.add(builder.le(root.get(v.getField()), (Number) v.getEnd()));
                    }
                });
            }
            //按照时间查询
            if (currencySearch.getDateParam() != null) {
                try {
                    predicateList.add(builder.between(root.get("createTime"), currencySearch.getDateParam().start(), currencySearch.getDateParam().end()));
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
            return builder.and(predicateList.toArray(new Predicate[predicateList.size()]));
        });
        return specification;
    }
```

## 多表操作

### 	表关系

#### 		一对一

#### 		一对多

​			1 : 主表

​			n : 从表

​			外键 : 在从表上新建一列作为外键，取值来源主键Id

#### 		多对多

​			中间表 : 两个字段，分别作为外键指向两表主键，又组成了联合主键。

### 	Jpa多表操作

#### 		一对多

​			使用注解的形式配置多表关系

​				声明关系

​					1 : @OneToMany 

​						mappedBy = "xxx" : n 表配置关系的属性名

​							放弃外键维护权

​					n : @ManyToOne

​						targetEntity : Xxx.class	（n 实体的字节码对象）

​						cascade : CascadeType.All			所有（推荐）

​											MERGE		改

​											PERSIST		增

​											REMOVE		删

​				配置外键

​					@JoinColumn : 配置外键

​						name : n 外键字段

​						referenceColumnName : 1 主键字段

​					此注解加在 1 的一端，即 1 来维护外键关系（不推荐，两条Insert，一条 Update）

​					此注解加在 n 的一端，即 n 来维护外键关系（推荐，两条Insert）

​					可只加一端，为单向维护关系。

​					可两端都加，为双向维护关系。（不推荐）

​					最好在逻辑里维护关系，不使用这种外键关系。

#### 		多对多​			![ZPdUKg.png](https://s2.ax1x.com/2019/06/23/ZPdUKg.png)

​			多表，一方放弃维护关系，与一对多中，1 放弃维护关系 一致。

## 对象导航查询

​	从一方查询多方 : 

​		默认 : 延迟加载

​	从多方查询一方 :

​		默认 : 立即加载

​	在被维护端 @OneToMany(fetch = EAGER | LAZY)  : 立即加载 | 延迟加载

​			

​						



​				