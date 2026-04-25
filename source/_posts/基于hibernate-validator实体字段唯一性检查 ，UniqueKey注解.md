---
title: 基于hibernate-validator实体字段唯一性检查 ，UniqueKey注解
date: 2026-04-25 18:18:42
categories:
  - 技术
tags:
  - java
  - hibernate
---

# 基于hibernate-validator实体字段唯一性检查 ，UniqueKey注解

前言
> 经常会在新增或修改时，检查某个字段或者多个字段的唯一性，如果重复就需要返回错误信息，重复代码写多了就准备写校验注解解决这个问题，分为两个版本，hibernate和mybatisplus

## 1.mybatisplus

### 注解

``` java
/**
 * 唯一约束
 * <p>
 * <a href="https://juejin.cn/post/7048658743197696008#heading-6">基于注解Uni检验字段是否重复</a><br/>
 * <a href="https://stackoverflow.com/questions/3495368/unique-constraint-with-jpa-and-bean-validation/3499111#3499111">Unique constraint with JPA and Bean Validation</a><br/>
 * <a href="https://stackoverflow.com/questions/17092601/how-to-validate-unique-username-in-spring">How to @Validate unique username in spring?</a><br/>
 * <p>
 *  使用：
 *  <pre>
 *      @UniqueKey(fields = "title")
 *      @UniqueKey.List(value = { @UniqueKey(fields = { "title" }), @UniqueKey(fields = { "author" }) }) // more than one unique keys
 *  </pre>
 * @author zengli
 * @date 2024/06/20
 */
@Documented
@Constraint(validatedBy = UniqueFieldsValidator.class)
@Target({ElementType.TYPE,ElementType.FIELD})
@Retention(RUNTIME)
public @interface UniqueKey {
    String message() default "{desc}--->{values}已存在";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};

    /**
     * 唯一约束的属性
     *
     * @return {@link String[]}
     */
    String[] fields();


    /**
     * 实体类
     * 如果在dto上使用，请将该值设置为数据库实体类
     * @return {@link Class}<{@link ?}>
     */
    Class<?> entityClass() default Object.class;

    /**
     * mapper类 spring bean 名称
     *
     * @return {@link String}
     */
    String mapperName() default "";

    @Target({ ElementType.TYPE })
    @Retention(RUNTIME)
    @Documented
    @interface List {
        UniqueKey[] value();
    }
}


```

### Validator

```java
public class UniqueFieldsValidator implements ConstraintValidator<UniqueKey, Object> {
    
    
    private String[] fields;
    private Class<?> clazz;
    private BaseMapper baseMapper;

    @Override
    public void initialize(UniqueKey constraintAnnotation) {
        this.fields = constraintAnnotation.fields();
        clazz = constraintAnnotation.entityClass();
        String mapperName = constraintAnnotation.mapperName();
        if(StrUtil.isNotEmpty(mapperName)){
            baseMapper = SpringUtil.getBean(mapperName, BaseMapper.class);
        }
    }

    @Override
    public boolean isValid(Object object, ConstraintValidatorContext context) {
        QueryWrapper queryWrapper = new QueryWrapper<>();
        if(ObjectUtil.equals(clazz,Object.class)){
            clazz = object.getClass();
        }
        TableInfo tableInfo = TableInfoHelper.getTableInfo(clazz);
        
        // 更新时主键不相等
        Field keyField = ReflectUtil.getField(clazz, tableInfo.getKeyProperty());
        String keyColumn = tableInfo.getKeyColumn();
        Object id = ReflectUtil.getFieldValue(object, keyField.getName());
        queryWrapper.ne(id!=null,keyColumn, id);
        
        Assert.notNull(tableInfo,"不存在实体{}对应的表", clazz); 
        if(baseMapper==null){
            baseMapper = getMapperByEntityClass(clazz);
        }
        List<TableFieldInfo> tableInfoFieldList = tableInfo.getFieldList();
        Map<String, TableFieldInfo> fieldInfoMap = tableInfoFieldList.stream().collect(Collectors.toMap(e -> e.getField().getName(), e -> e));
        Set<String> propertyList = tableInfoFieldList.stream().map(e -> e.getProperty()).collect(Collectors.toSet());
        Assert.isTrue(CollUtil.containsAll(propertyList, Arrays.asList(fields)), "注解中存在不是实体类{}的字段", clazz);
        Arrays.stream(fields).forEach(field -> {
            Object value = ReflectUtil.getFieldValue(object, field);
            String column = fieldInfoMap.get(field).getColumn();
            queryWrapper.eq(column, value);
        });

        int count = baseMapper.selectCount(queryWrapper);
        boolean b = count == 0;
        if(!b){
            // 错误消息
            String desc =Arrays.stream(fields).map(e->{
                Field field = fieldInfoMap.get(e).getField();
                String value = field.getAnnotation(ApiModelProperty.class).value();
                return value;
            }).collect(Collectors.joining(","));
            String values=Arrays.stream(fields).map(e->{
                Object value = ReflectUtil.getFieldValue(object, e);
                return Convert.toStr(value);
            }).collect(Collectors.joining(","));
            context.unwrap(HibernateConstraintValidatorContext.class)
                    .addMessageParameter("desc", desc)
                    .addMessageParameter("values", values);
        }
        return b;
    }

    public <T> BaseMapper<T> getMapperByEntityClass(Class<T> entityClass) {
        String mapperName = entityClass.getSimpleName() + "Mapper";
        // 将首字母小写
        mapperName = Character.toLowerCase(mapperName.charAt(0)) + mapperName.substring(1);
        return (BaseMapper<T>) SpringUtil.getBean(mapperName);
    }
}
```

### 使用

```java
@Data
@UniqueKey(fields = {"contractNo"},entityClass = SalesOrder.class)
public class SalesOrderOverviewDto {
    
    /** 主键id */
    @TableId(value = "id", type = IdType.AUTO)
    private String id;


    // @NotEmpty(message = "合同编号不能为空") 可以为空
    @ApiModelProperty(value = "合同编号")
    private String contractNo;


}
```





## hibernate

### 注解

``` java

import java.io.Serializable;
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import javax.validation.Constraint;
import javax.validation.Payload;

/**
 * <pre>
 *     @UniqueKey(property = "title")
 *     @UniqueKey.List(value = { @UniqueKey(property = { "title" }), @UniqueKey(property = { "author" }) }) // more than one unique keys
 * </pre>
 * 
 */
@Constraint(validatedBy = { UniqueKeyValidator.class})
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface UniqueKey {

	/**
	 * 唯一约束的属性
	 *
	 * @return {@link String[]}
	 */
	String[] fields();
	
	/**
	 * 实体类
	 * 如果在dto上使用，请将该值设置为数据库实体类
	 * @return {@link Class}<{@link ?}>
	 */
	Class<?> entityClass() default Object.class;


	String message() default "{desc}--->{values}已存在";

	Class<?>[] groups() default {};

	Class<? extends Payload>[] payload() default {};

	@Target({ ElementType.TYPE })
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@interface List {
		UniqueKey[] value();
	}

}


```

### Validator

```java


import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.collection.ListUtil;
import cn.hutool.core.convert.Convert;
import cn.hutool.core.util.ArrayUtil;
import cn.hutool.core.util.ReflectUtil;
import io.swagger.annotations.ApiModelProperty;
import org.hibernate.validator.constraintvalidation.HibernateConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowire;
import org.springframework.beans.factory.annotation.Configurable;
import org.springframework.stereotype.Component;

import java.io.Serializable;
import java.lang.reflect.Field;
import java.util.*;
import java.util.stream.Collectors;

import javax.annotation.Resource;
import javax.inject.Inject;
import javax.persistence.EntityManager;
import javax.persistence.Id;
import javax.persistence.PersistenceContext;
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;
import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import javax.validation.ConstraintValidatorContext.ConstraintViolationBuilder;
import javax.validation.ConstraintValidatorContext.ConstraintViolationBuilder.NodeBuilderDefinedContext;


/**
 * 参考：<br/>
 * <a href="https://stackoverflow.com/questions/3495368/unique-constraint-with-jpa-and-bean-validation/3499111#3499111">Unique constraint with JPA and Bean Validation</a><br/>
 * <a href="https://lucasterdev.altervista.org/wordpress/2012/07/28/unique-constraint-validation-part-1/">example</a>
 * <p>
 * 存在一个问题，@Validated 会在controller执行，并且会在hibernate持久化之前执行，也就是执行了两次
 * entityManager 在hibernate持久化时会为空
 * 解决办法：<br/>
 * <a href="https://stackoverflow.com/questions/24955817/jsr303-custom-validators-being-called-twice">jsr303-custom-validators-being-called-twice</a>
 * <a href="https://stackoverflow.com/questions/65071004/how-to-do-custom-validation-on-entity-for-multitenant-setup-using-spring-hiber">how-to-do-custom-validation-on-entity-for-multitenant-setup-using-spring-hiber</a>
 * <pre><code>
 *     spring.jpa.properties.javax.persistence.validation.mode=none
 * </code></pre>
 * <pre>{@code 
 * @Component
 * public class HibernateCustomizer implements HibernatePropertiesCustomizer {
 *
 *     private final ValidatorFactory validatorFactory;
 *
 *     public HibernateCustomizer(ValidatorFactory validatorFactory) {
 *         this.validatorFactory = validatorFactory;
 *     }
 *
 *     public void customize(Map<String, Object> hibernateProperties) {
 *         hibernateProperties.put("javax.persistence.validation.factory", validatorFactory);
 *     }
 * }
 * 
 * }
 * {@code 
 * @Configuration
 * public class BeanValidationConfig {
 *    @Bean
 *    public LocalValidatorFactoryBean getValidator() {
 *        return new LocalValidatorFactoryBean();
 *    }
 * }   
 * }
 * </pre>
 */
// @Component
public class UniqueKeyValidator implements
		ConstraintValidator<UniqueKey, Object> {

	@PersistenceContext
	private EntityManager entityManager;

	private UniqueKey constraintAnnotation;

	public UniqueKeyValidator() {}

	public UniqueKeyValidator(final EntityManager entityManager) {
		this.entityManager = entityManager;
	}

	public EntityManager getEntityManager() {
		return entityManager;
	}

	@Override
	public void initialize(final UniqueKey constraintAnnotation) {
		this.constraintAnnotation = constraintAnnotation;
	}

	@Override
	public boolean isValid(final Object target,
			final ConstraintValidatorContext context) {

		if (entityManager == null) {
			// eclipselink may be configured with a BeanValidationListener that
			// validates an entity on prePersist
			// In this case we don't want to and we cannot check anything (the
			// entityManager is not set)
			//
			// Alternatively, you can disalbe bean validation during jpa
			// operations
			// by adding the property "javax.persistence.validation.mode" with
			// value "NONE" to persistence.xml
			// throw new RuntimeException("entityManager 为空");
			// TODO 可以测试分组校验是否能解决这个问题
			return true;
		}
		Class clazz = constraintAnnotation.entityClass();
		if(Objects.equals(clazz,Object.class)){
			clazz=target.getClass();
		}
		final Class<?> entityClass = clazz;

		final CriteriaBuilder criteriaBuilder = entityManager
				.getCriteriaBuilder();

		final CriteriaQuery<Object> criteriaQuery = criteriaBuilder
				.createQuery();

		final Root<?> root = criteriaQuery.from(entityClass);

		try {
			List<String> fields = Arrays.asList(constraintAnnotation.fields());
			if(CollUtil.isEmpty(fields))return true;
			Map<String, Object> field2ValueMap = fields.stream().collect(Collectors.toMap(e -> e, field -> {
				Object value = ReflectUtil.getFieldValue(target, field);
				return value;
			}));

			List<Predicate> predicateList = field2ValueMap.entrySet().stream().map(e->{
				String key = e.getKey();
				Object value = e.getValue();
				return criteriaBuilder.equal(root.get(key), value);
			}).collect(Collectors.toList());

			final Field idField = getIdField(entityClass);
			final String idProperty = idField.getName();
			final Object idValue = ReflectUtil.getFieldValue(target, idProperty);

			if (idValue != null) {
				final Predicate idNotEqualsPredicate = criteriaBuilder
						.notEqual(root.get(idProperty), idValue);
				predicateList.add(idNotEqualsPredicate);
				criteriaQuery.select(root).where(predicateList.toArray(new Predicate[0]));
			} else {
				criteriaQuery.select(root).where(predicateList.toArray(new Predicate[0]));
			}

			final List<Object> resultSet = entityManager.createQuery(criteriaQuery)
					.getResultList();

			if (!resultSet.isEmpty()) {
				// ConstraintViolationBuilder cvb = context
				// 		.buildConstraintViolationWithTemplate(constraintAnnotation
				// 				.message());
				// NodeBuilderDefinedContext nbdc = cvb.addNode(constraintAnnotation
				// 		.property());
				// ConstraintValidatorContext cvc = nbdc.addConstraintViolation();
				// cvc.disableDefaultConstraintViolation();

				// 错误消息
				String desc =fields.stream().map(e->{
					Field field = ReflectUtil.getField(entityClass, e);
					String value = Optional.ofNullable(field.getAnnotation(ApiModelProperty.class)).map(t->t.value()).orElse(e);
					return value;
				}).collect(Collectors.joining(","));
				String values=fields.stream().map(e->{
					Object value = field2ValueMap.get(e);
					return Convert.toStr(value);
				}).collect(Collectors.joining(","));
				context.unwrap(HibernateConstraintValidatorContext.class)
						.addMessageParameter("desc", desc)
						.addMessageParameter("values", values);
				return false;
			}

		} catch (final Exception e) {
			throw new RuntimeException(
					"An error occurred when trying to create the jpa predicate for the @UniqueKey '"
							+ Arrays.toString(constraintAnnotation.fields())
							+ "' on bean "
							+ entityClass + ".", e);
		}

		return true;
	}

	/**
	 * 获取实体类主键
	 *
	 * @param clazz
	 * @return
	 */
	public static Field getIdField(Class<?> clazz) {
		Field[] fields = clazz.getDeclaredFields();
		Field item = null;
		for (Field field : fields) {
			Id id = field.getAnnotation(Id.class);
			if (id != null) {
				field.setAccessible(true);
				item = field;
				break;
			}
		}
		if (item == null) {
			Class<?> superclass = clazz.getSuperclass();
			if (superclass != null) {
				item = getIdField(superclass);
			}
		}
		return item;
	}

}

```

### 使用

```java
@UniqueKey(fields = {"projectId","elevatorCode"},message = "该电梯编码已存在请重新输入")
@Entity
@Data
@Table(name="elevators",uniqueConstraints ={@UniqueConstraint(columnNames= {"project_id","elevator_code"})})
public class Elevators extends BaseEntity{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "`id`")
    @ApiModelProperty(value = "主键id 自增")
    private Long id;

    @Column(name = "`project_id`")
    private Long projectId;
    
    @Column(name = "`elevator_code`",updatable = false)
    private String elevatorCode;
    

}

```

- 注意

  >**除非你获取了对整个表的锁**，否则基本上不可能使用 SQL 查询来检查单一性（任何并发事务都可以在手动检查后但在正在进行的事务提交之前修改数据）。换言之，不可能在 Java 级别实现有效的唯一验证，从而提供验证实现。检查单一性的唯一可靠方法是在提交事务时。

