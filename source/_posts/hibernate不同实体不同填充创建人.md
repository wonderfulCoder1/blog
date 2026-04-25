---
title: hibernate不同实体不同填充创建人
date: 2026-04-25 18:18:42
categories:
  - 技术
tags:
  - java
  - hibernate
---

# hibernate不同实体不同填充创建人

使用的el-admin框架，框架本身填充的使用`@CreatedBy`注解加上`AuditingEntityListener`,
```java
@CreatedBy
@Column(name = "create_by", updatable = false)
@ApiModelProperty(value = "创建人", hidden = true)
private String createBy;


@Component("auditorAware")
public class AuditorConfig implements AuditorAware<String> {

    /**
     * 返回操作员标志信息
     *
     * @return /
     */
    @Override
    public Optional<String> getCurrentAuditor() {
        try {
            // 这里应根据实际业务情况获取具体信息
            return Optional.of(SecurityUtils.getCurrentUsername());
        }catch (Exception ignored){}
        // 用户定时任务，或者无Token调用的情况
        return Optional.of("System");
    }
}
```


但是我的项目createBy是Long类型，所以需要不同的填充方式

### 实体基类
```java
@EntityListeners(CustomAuditingListener.class)
@javax.persistence.MappedSuperclass
@Data
@Accessors(chain = true)
public class BaseEntity implements Serializable {

	/**
	 * 创建人id
	 */
	@Column(name = "create_id")
	protected Long createId;

}
```
> 主要是加入 @EntityListeners(CustomAuditingListener.class)

### 实现CustomAuditingListener

```java
@Configurable
public class CustomAuditingListener implements ConfigurableObject {

    public CustomAuditingListener() {
    }

    @Autowired
    private AuditHandler auditHandler;

    @PrePersist
    private void prePersist(Object obj) {
        auditHandler.prePersist(obj);
    }
    @PreUpdate
    private void preUpdate(Object obj) {
        auditHandler.preUpdate(obj);
    }
}

```

### AuditHandler接口
```java
public interface AuditHandler {
    void prePersist(Object obj);

    void preUpdate(Object obj);
}


```

```java
@Component
public class CustomAuditHandler implements AuditHandler {

    @Override
    public void prePersist(Object obj) {
        if (obj instanceof BaseEntity) {
            BaseEntity entity = (BaseEntity) obj;
            if (entity.getCreateId() == null) {
                this.markForCreate(entity);
            }
            entity.setCreateTime(new Date());
            entity.setDataStatus(1);
        }
    }

    @Override
    public void preUpdate(Object obj) {
        if (obj instanceof BaseEntity) {
            BaseEntity entity = (BaseEntity) obj;
            if(entity.getUpdateId() == null){
                this.markForUpdate(entity);
            }
            entity.setUpdateTime(new Date());
            entity.setDataStatus(1);
        }
    }

    public void markForCreate(BaseEntity entity) {
        Long userId = getLoginUserId();
        entity.setCreateId(userId);
    }

    public void markForUpdate(BaseEntity entity) {
        // TODO 问题 https://stackoverflow.com/questions/54041338/stackoverflow-on-preupdate
        Long userId = getLoginUserId();
        entity.setUpdateId(userId);
    }

    private static Long getLoginUserId() {
        JwtUserDto userCache = SpringUtil.getBean(UserCacheManager.class).getUserCache(SecurityUtils.getCurrentUsername());
        Long userId = Optional.ofNullable(userCache).map(e -> e.getUser()).map(e -> e.getId()).orElse(null);
        return userId;
    }
}
```
