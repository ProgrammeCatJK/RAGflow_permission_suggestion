# Permission System Design (ReBAC)

## 1. Access Control Model

本系统采用 **Relationship-Based Access Control (ReBAC)**  
通过实体之间的关系来实现权限控制：

- User ↔ Tenant
- Tenant ↔ Tenant（层级关系）
- Tenant ↔ KnowledgeBase
- KnowledgeBase ↔ Document

---

## 2. 为什么不使用 Tag / ABAC

- Tag 只能描述属性（如：corp / hospital / department）
- Tag 无法表达：
  - 医院属于哪个集团
  - 部门属于哪个医院
- 无法回答「上下级归属关系」

结论：  
**必须使用 ReBAC + Tenant 层级关系**

---

## 3. 实体关系模型

```
User
 └── member_of (role)
      └── Tenant (集团 / 医院 / 部门)
           └── parent_of
                └── Tenant
                     └── owns
                          └── KnowledgeBase
                               └── contains
                                    └── Document
```

---

## 4. Database Design

### 4.1 user_tenant

```text
user_tenant
------------
user_id
tenant_id
role            # owner | admin | normal | invite
```

---

### 4.2 tenant

```text
tenant
------------
id
parent_id          # 上级 tenant（NULL = 集团）
tenant_type        # 0=集团, 1=医院, 2=部门, 3=用户
inherit_access     # 是否允许上级继承访问权限（default=true）
path               # 层级路径（可选）
```

---

### 4.3 tenant_kb

```text
tenant_kb
------------
tenant_id
kb_id
```

---

### 4.4 user_kb

```text
user_kb
------------
user_id
kb_id
role
```

---

### 4.5 document

```text
document
------------
id
kb_id
tenant_id
```

---

## 5. 权限继承规则

### 5.1 KnowledgeBase 权限

| Role   | Permissions |
|------|------------|
| owner | CRUD + INVITE |
| admin | READ + UPDATE + INVITE |
| normal| READ |
| invite| none |

---

### 5.2 Document 权限

Document 权限完全继承所属 KnowledgeBase 权限

---

## 6. Action 定义

```python
class Action(Enum):
    CREATE
    READ
    UPDATE
    DELETE
    INVITE
```

---

## 7. Permit 统一鉴权入口

```text
Permit.check(user, resource, action)

1. superuser → true
2. Document → check Document.kb
3. KnowledgeBase → _check_kb
4. default → false
```

---

## 8. Tenant 父级查询逻辑

```python
def get_parents(tenant_id):
    parents = []
    current = Tenant.get(tenant_id)
    while current and current.parent_id:
        parent = Tenant.get(current.parent_id)
        parents.append(parent)
        current = parent
    return parents
```

---

## 9. 总结

- ReBAC 表达集团 / 医院 / 部门层级
- Tenant 记录 parent 关系
- KB 管权限，Document 继承
- 单一 Permit.check 鉴权入口
