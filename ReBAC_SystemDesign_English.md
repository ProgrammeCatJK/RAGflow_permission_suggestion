# ReBAC Permission Design
---
## 1. Permission System Design

Using **Relationship Based Access Control（ReBAC）**  
Through perserving the relationship between tenant(workspace)，rather than Tag / ABAC。

Reason：

- Tag is unable to show hierarchy **Group → Hospital → Department** coperational relationship
- Tag is unable to answer these questions：
  - which group does this hospital belong to？
  - which hopsital does this department belong to？

Thus there must be a way to save the relatioship between **Tenant relationship**。

Database design：
- User-- member_of(role) —> tenant **user_tenant
- Tenant — parent_of  —> tenant *tenant.parent to store the relationship
- Tenant — owns —> knowledge —> tenant_kb
- User — has_role —> knowledge —> user_kb

tenant table design —> store DAG relationship：
- parent_id
- tenant_type —> Group = 0，hopital = 1， department = 2， user = 3
- path —> parent path （optional）

document table design：
- tenant_id —> reduce the number of join（memory vs performance）
	reduce the use join document kb tenant during search of document

current implmeneted design：
— document_user_role—> database would grow exponentially，performance is bad 
  - solution：inherit relatinship from tenant

---

## 2. core relatinship model

```
User
 └── member_of (role)
      └── Tenant (Group / Hospital / Department)
           └── parent_of
                └── Tenant
                     └── owns
                          └── KnowledgeBase
                               └── contains
                                    └── Document
```

---

## 3. Database design

### 3.1 user_tenant（Tenant is belonged to which user）

```text
user_tenant
------------
user_id
tenant_id
role            # owner | admin | normal | invite
```

---

### 3.2 tenant（workspace）

```text
tenant
------------
id
parent_id          # save hierarchy ** new
tenant_type        # 0=group, 1=hospital, 2=department, 3=user **new
path  （optional）--> complicated CRUD for tenant, but fast lookup
```

### 3.3 document

```text
document
------------
id
kb_id
tenant_id # new（ownership of document for tenant） fast loopuk, large memory space
```

---

## 4. Permission inherit design for document

Document completely inherit the KnowledgeBase permission  
remove **document_user_role ** table。

---

## 5. Action Define

```python
CREATE, READ, UPDATE, DELETE, INVITE
```

---

## 6. Role permission （role_allow logic）

```
owner   -> CRUD + INVITE
admin   -> READ + INVITE
normal  -> READ
invite  -> NONE
```

---

## 7. Permit design

Common input：
- resource(teant, knowledgebase, document)
```python
Permit.check(user, resource, action)

  check(user, resource, action)
  		if superuser return true; 
  			
  		if isinstance(resource, kb)
  			return _check_kb(user, resource, action)
  
  		if isinstance(resource, document)
  			return _check_kb(user, resource.kb, action)

  		@default
  		return false
```

---

## 8. Tenant inherit logic

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

## 9. core KB permission logic

```python
# get_role(user, tenant) —> user_tenant.role
# get_tenant_id (tenant) —> tenant.id
# new sql query get_parent(tenant_id) return parent: tenant_id

_check_kb(user, kb, action):
    role = get_role(user, kb.tenant_id)

    if role and role_allows(role, action):
        return True

    for parent in get_parent(kb.tenant_id):
        role = get_role(user, parent.id)
        if role and role_allows(role, action):
            return True

    return False

# core logic to find relationship only save the parent relationship(I belong to which parent)
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
- pyhton implementation requires while loop --> performance trade-off
- while-loop can be run in SQL, reduce implementation in python（solution）

Optimization：
- use Recursive CTE in database to lookup for tenant ancestors
- return user's tenant with all ancestor tenant' role list
- Python only implements **role_allows** action decision
- using cycle_checking to prevent loops in the relationship

---

## 10. summary

- using ReBAC represent the relationship between tenant
- Tenant table add parent_id
- Document inherits permission of KB
- Only Permit.check input

### 11.1 Login workflow
```
user login  
    │
    ▼
user Auth (Auth Service)  
    │
    ▼
Get user info (`useFetchUserInfo`)  
    ├─ user_id  
    ├─ is_superuser  
    └─ other info  
```

---

### 11.2 List user's tenants（Tenant List）
```
get tenant List (`useListTenant`)  
    │
    ▼
Search `user_tenant` table  
    │
    └─ return user's tenant list  
        ├─ tenant_id  
        ├─ tenant_name  
        ├─ tenant_type  
        ├─ user_role  
        └─ joined_at  
```

---

### 11.3 Switch workspace（Tenant）
```
user select Tenant  
    │
    ▼
get current tenant info (`useFetchTenantInfo`)  
    ├─ tenant_id  
    ├─ tenant_type  
    ├─ parent_id  
    └─ other info  

 (`useListTenantUser`)  
    │
    ▼
search `user_tenant` table  
    │
    └─ return member list  
        ├─ user_id
        ├─ role (owner/admin/normal/invite)  
        └─ permission control  
```

---

### 11.4 user ACTION on Document（Inherit user permisison in KB）
```
user ACTION on Document（CREATE / READ / UPDATE / DELETE）  
    │
    ▼
 `Permit.check(user, document, action)`  
    │
    ▼
 KB check： `_check_kb(user, document.kb, action)`  
    │
    ├─ **True** → allowed  
    └─ **False** → not allowed  
```


---

### 11.5 Permission check
```
any user with acitons → `Permit.check(user, resource, action)`  
    │
    ├─ **superuser** → True  
    ├─ **Document** →  KB permisison  
    ├─ **KnowledgeBase** → Tenant ReBAC   
    └─ **default** → False
```
