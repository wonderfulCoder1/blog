---
title: java list转树形列表
date: 2026-04-25 18:18:42
categories:
  - 技术
tags:
  - java
  - 部署
  - 博客
---
### 树形接口

#### 接口类

> 如果觉得接口对json序列化有影响，可以使用 `@JsonIgnore` 来消除影响 

```java
/**
 * 树节点父类，所有需要使用{@linkplain TreeUtils}工具类形成树形结构等操作的节点都需要实现该接口
 *
 * @param <T> 节点id类型
 */
public interface TreeNode<T> {
    /**
     * 获取节点id
     *
     * @return 树节点id
     */
    T id();

    /**
     * 获取该节点的父节点id
     *
     * @return 父节点id
     */
    T parentId();

    /**
     * 是否是根节点
     *
     * @return true：根节点
     */
    boolean root();

    /**
     * 设置节点的子节点列表
     *
     * @param children 子节点
     */
    void setChildren(List<? extends TreeNode<T>> children);

    /**
     * 获取所有子节点
     *
     * @return 子节点列表
     */
    List<? extends TreeNode<T>> getChildren();
}
```

#### 树形列表工具

```java
import org.springframework.util.CollectionUtils;

import java.util.*;

/**
 * 树形结构工具类
 *
 * @author meilin.huang
 * @version 1.0
 * @date 2019-08-24 1:57 下午
 */
public class TreeUtils {
    /**
     * 根据所有树节点列表，生成含有所有树形结构的列表
     *
     * @param nodes 树形节点列表
     * @param <T>   节点类型
     * @return 树形结构列表
     */
    public static <T extends TreeNode<?>> List<T> generateTrees(List<T> nodes) {
        List<T> roots = new ArrayList<>();
        for (Iterator<T> ite = nodes.iterator(); ite.hasNext(); ) {
            T node = ite.next();
            if (node.root()) {
                roots.add(node);
                // 从所有节点列表中删除该节点，以免后续重复遍历该节点
                ite.remove();
            }
        }
        roots.forEach(r -> {
            setChildren(r, nodes);
        });
        return roots;
    }

    /**
     * 从所有节点列表中查找并设置parent的所有子节点
     *
     * @param parent 父节点
     * @param nodes  所有树节点列表
     */
    @SuppressWarnings("all")
    public static <T extends TreeNode> void setChildren(T parent, List<T> nodes) {
        List<T> children = new ArrayList<>();
        Object parentId = parent.id();
        for (Iterator<T> ite = nodes.iterator(); ite.hasNext(); ) {
            T node = ite.next();
            if (Objects.equals(node.parentId(), parentId)) {
                children.add(node);
                // 从所有节点列表中删除该节点，以免后续重复遍历该节点
                ite.remove();
            }
        }
        // 如果孩子为空，则直接返回,否则继续递归设置孩子的孩子
        if (children.isEmpty()) {
            return;
        }
        parent.setChildren(children);
        children.forEach(m -> {
            // 递归设置子节点
            setChildren(m, nodes);
        });
    }

    /**
     * 获取指定树节点下的所有叶子节点
     *
     * @param parent 父节点
     * @param <T>    实际节点类型
     * @return 叶子节点
     */
    public static <T extends TreeNode<?>> List<T> getLeafs(T parent) {
        List<T> leafs = new ArrayList<>();
        fillLeaf(parent, leafs);
        return leafs;
    }

    /**
     * 将parent的所有叶子节点填充至leafs列表中
     *
     * @param parent 父节点
     * @param leafs  叶子节点列表
     * @param <T>    实际节点类型
     */
    @SuppressWarnings("all")
    public static <T extends TreeNode> void fillLeaf(T parent, List<T> leafs) {
        List<T> children = parent.getChildren();
        // 如果节点没有子节点则说明为叶子节点
        if (CollectionUtils.isEmpty(children)) {
            leafs.add(parent);
            return;
        }
        // 递归调用子节点，查找叶子节点
        for (T child : children) {
            fillLeaf(child, leafs);
        }
    }

    /**
     * 获取指定节点到根节点链路上的所有节点
     * @param node      指定节点
     * @param id2Node   所有节点的id到节点的映射
     * @return
     * @param <T>       节点类型
     * @param <M>       节点id类型
     */
    public static <T extends TreeNode<M>,M> List<T> getChain(T node, Map<M,T> id2Node) {
        List<T> chain = new ArrayList<>();
        chain.add(node);
        M parentId = node.parentId();
        T parent = id2Node.get(parentId);
        while (parent != null) {
            chain.add(parent);
            parentId = parent.parentId();
            parent = id2Node.get(parentId);
        }
        return chain;
    }

    /**
     * 获得所有叶子节点的到根节点链路上的所有节点的集合
     * @param nodes     节点集合
     * @param id2Node   所有节点的id到节点的映射
     * @return
     * @param <T>       节点类型
     * @param <M>       节点id类型
     */
    public static <T extends TreeNode<M>,M> Map<T,List<T>> getChains(List<T> nodes, Map<M,T> id2Node) {
        Map<T,List<T>> map = new LinkedHashMap<>();
        for (T node : nodes) {
            List<T> chain = getChain(node, id2Node);
            map.put(node,chain);
        }
        return map;
    }
}
```

> `generateTrees` 方法生成树，可以自行添加其他方法，比如遍历树方法

#### 实体类实现接口

<details>
<summary>点击查看代码</summary>

```java
/**
 * <p>
 * 模板
 * </p>
 *
 * @author zengli
 * @since 2023-05-16
 */
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Accessors(chain = true)
@Builder
@TableName(value = "template",autoResultMap = true)
public class Template implements TreeNode<Long> {

    private static final long serialVersionUID = 1L;
    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    @ApiModelProperty(value = "父级id，0没有父级")
    @TableField("parent_id")
    private Long parentId;

    @ApiModelProperty(value = "模板名称")
    private String templateName;

    @ApiModelProperty(value = "子模板列表")
    @TableField(exist = false)
    private List<Template> children;

    //==============TreeNode================
    @Override
    public Long id() {
        return id;
    }

    @Override
    public Long parentId() {
        return parentId;
    }

    @Override
    public boolean root() {
        return Objects.equals(0L,parentId);
    }

    @Override
    public void setChildren(List children) {
        this.children=children;
    }
    //==============TreeNode================
}
```
</details>



#### Service代码使用

```java
List<Template> templateList=templateMapper.selectList(null);
List<Template> ret= TreeUtils.generateTrees(templateList);
```
