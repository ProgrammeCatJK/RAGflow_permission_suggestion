# RAGFlow 权限管理方案

## Tenant 等级与层级
- **集团** = 0
- **医院** = 1
- **科室** = 2

Tenant 层级关系:
- 集团账户 → 旗下医院的知识库 → 下属科室的知识库
- 医院账户 → 下属科室的知识库
- 科室账户 → 用户的知识库

Tenant 角色等级:
- owner = 0
- admin = 1
- normal = 2
- invite = 3

`user_tenant.role`：储存用户在该 tenant 内的角色。

Tenant 上下级关系:
- `parent_id` 指向上级 tenant
- 集团的 `parent_id = NULL`
- 可通过 UNION 查询路径归属

上级是否有继承访问权：tenant 表中设置的关键词。

直接用 `kb_user_role` 来判断用户是否有权限访问 KB。

---

## 1. 用户想 CRUD 知识库 (owner, admin)
**步骤:**
1. 查看 user 是否在 `user_tenant` → 如果不存在，返回 0
2. 查看 `user_tenant.role` 是否为 owner 或 admin → 如果不是，返回 0
3. 否则返回 1 → 创建 KB (`kb_id`) → 添加 `kb_user_role`，角色为 owner

---

## 2. 用户想读取知识库 (owner, admin, normal)
**步骤:**
1. 查看 user 是否在 `user_tenant` → 如果不存在，返回 0
2. 查看 `kb_user_role(user_id, kb_id)` → 如果 role 在 (owner, admin, normal)，返回 1，否则继续
3. 没有显式权限 → 查询 KB 的 tenant 和 `user_tenant.role` (owner, admin, normal) → 如果符合，返回 1，否则继续
4. 检查继承权限且用户角色符合规则 → 如果符合，返回 1
5. 否则返回 0

---

## 3. Owner 添加/删除 Admin
**步骤:**
1. 查询 `user_tenant`
   - 如果存在 → 检查角色是否允许添加 KB 用户
   - 如果不存在 → 创建 `user_tenant(user_id, tenant_id, role)`
2. 查询 `kb_user_role(user_id, kb_id)`
   - 如果已存在 → 更新角色 / 提示已存在
   - 如果不存在 → 插入 `kb_user_role(user_id, kb_id, role)`
3. 返回 0 / 1

---

## 4. Owner/Admin Invite 用户
**步骤:**
1. 查询 `user_tenant`
   - 如果存在 → 检查操作人权限
   - 如果不存在 → 创建 `user_tenant(user_id, tenant_id, role)`
2. 查询 `kb_user_role(user_id, kb_id)`
   - 如果不存在 → 插入 `kb_user_role(user_id, kb_id, role=Invite)`
   - 如果存在 → 更新角色为 Invite（可选）
3. 返回 0 / 1
