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

Database设计：
- User-- member_of(role) —> tenant **user_tenant
- Tenant — parent_of  —> tenant *tenant.parent to store the relationship
- Tenant — owns —> knowledge —> tenant_kb
- User — has_role —> knowledge —> user_kb

tenant表要储存 —> 必须要知道关系：
- parent_id
- tenant_type —> 集团 = 0，医院 = 1， 部门 = 2， 用户 = 3
- path —> 需要知道 parent路径 （可选）

document表要储存：
- tenant_id —> 不需要每次都用join（memory vs performance）
	也可以不需要但是每次查询都要用 join document kb tenant

已经实现的设计：
— document_user_role—> 非常大，性能不好卡 
  - 方案：继承kb的权限就好了

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
parent_id          # 保存等级关系 ** 新加 
tenant_type        # 0=集团, 1=医院, 2=部门, 3=用户 **新加
path  （可选）--> 管理复杂 但查询快
```

### 3.3 document

```text
document
------------
id
kb_id
tenant_id # 新加（知道文件属于那个租户） 快速查询但是容量会增加
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

## 6. Role 权限映射 （role_allow逻辑）

```
owner   -> CRUD + INVITE
admin   -> READ + INVITE
normal  -> READ
invite  -> NONE
```

---

## 7. Permit 设计

统一入口：

```python
Permit.check(user, resource, action)

  check(user, resource, action)
  		if superuser return true;
  		
  		if user_id in user_tenant 
  			
  		if isinstance(resource, kb)
  			return check(user, resource, action)
  
  		if isinstance(resource, document)
  			return check(user, resource.kb, action)

  		@default
  		return false
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
# 要 get_role(user, tenant) —> user_tenant.role
# 要 get_tenant_id (tenant) —> tenant.id
# 要新sql query get_parent(tenant_id) return parent: tenant_id

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

# 核心逻辑去寻找等级关系
get_parent(tenant_id):
  parents = []
  current = Tenant.get(tenant_id) —> get all the tenants with this tenant_id (only 1)
  while current and current.parent_id:
    parent = get(current.parent.id)
    if not parent:
      break;
    ancestors.append(parent)
    current = parent
  return ancestors
```

---

## 10. 总结

- 使用 ReBAC 表达组织关系
- Tenant 表存 parent
- Document 权限继承 KB
- 单一 Permit.check 入口
- database 增加和删除逻辑也需要修改
