# ReBAC Permission Design

## 1. 权限模型说明

当前方案采用 **Relationship Based Access Control（ReBAC）**  
通过实体之间的关系来做权限判断，而不是简单的 Tag / ABAC。

原因：

- Tag 无法表达 **集团 → 医院 → 部门** 的层级关系
- Tag 无法回答：
  - 这个医院属于哪个集团？
  - 这个部门归属于哪个医院？

因此必须显式存储 **Tenant 之间的关系**。

---

## 2. 核心关系模型

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

## 3. Database 设计

### 3.1 user_tenant（用户属于哪个 Tenant）

```text
user_tenant
------------
user_id
tenant_id
role            # owner | admin | normal | invite
```

---

### 3.2 tenant（租户 / 工作区）

```text
tenant
------------
id
parent_id
tenant_type        # 0=集团, 1=医院, 2=部门, 3=用户
inherit_access
path
```

---

### 3.3 tenant_kb

```text
tenant_kb
------------
tenant_id
kb_id
```

---

### 3.4 user_kb

```text
user_kb
------------
user_id
kb_id
role
```

---

### 3.5 document

```text
document
------------
id
kb_id
tenant_id
```

---

## 4. 权限继承策略

Document 权限完全继承 KnowledgeBase 权限  
完全不需要 document_user_role 表。

---

## 5. Action 定义

```python
CREATE, READ, UPDATE, DELETE, INVITE
```

---

## 6. Role 权限映射

```
owner   -> CRUD + INVITE
admin   -> READ + UPDATE + INVITE
normal  -> READ
invite  -> NONE
```

---

## 7. Permit 设计

统一入口：

```python
Permit.check(user, resource, action)
```

---

## 8. Tenant 继承逻辑

```python
get_parent(tenant_id):
    parents = []
    current = Tenant.get(tenant_id)

    while current and current.parent_id:
        parent = Tenant.get(current.parent_id)
        if not parent:
            break
        parents.append(parent)
        current = parent

    return parents
```

---

## 9. 核心 KB 鉴权逻辑

```python
_check_kb(user, kb, action):
    role = get_role(user, kb.tenant_id)

    if role and role_allows(role, action):
        return True

    for parent in get_parent(kb.tenant_id):
        if not parent.inherit_access:
            continue
        role = get_role(user, parent.id)
        if role and role_allows(role, action):
            return True

    return False
```

---

## 10. 总结

- 使用 ReBAC 表达组织关系
- Tenant 表存 parent
- Document 权限继承 KB
- 单一 Permit.check 入口
