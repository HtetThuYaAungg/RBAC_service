# 🛡️ Role Creation Service - NestJS + Prisma

This document outlines the design and implementation of the **Role Creation Service**, which dynamically creates roles and assigns permissions in a structured, nested way.

---

## 1. 🔧 Main Role Creation Function

```ts
async function createRole(createRoleDto: CreateRoleDto, req: Request): Promise<Role> {
  // Step 1: Get authenticated user
  const loginUser = await getUser(req);
  
  // Step 2: Create the base role
  const role = await createBaseRole(createRoleDto, loginUser);
  
  // Step 3: Establish permission relationships
  await setupRolePermissions(role.id, createRoleDto.permissions, req);
  
  return role;
}
```

---

## 2. 🧩 Helper Functions Breakdown

### 🔨 Create the Base Role

```ts
async function createBaseRole(dto: CreateRoleDto, user: User) {
  return prisma.role.create({
    data: {
      role_code: dto.role_code,
      role_name: dto.role_name,
      permissions: JSON.stringify(dto.permissions),
      created_by: { connect: { id: user.id } }
    }
  });
}
```

### 🔗 Setup Role-Permission Relationships

```ts
async function setupRolePermissions(roleId: number, permissions: PermissionDto[], req: Request) {
  const user = await getUser(req);
  const permissionNames = getPermissionNames(permissions);

  for (const name of permissionNames) {
    try {
      const permission = await getOrCreatePermission(name, user.id);
      await linkRoleToPermission(roleId, permission.id, user.id);
    } catch (error) {
      if (error.code !== 'DUPLICATE') throw error;
    }
  }
}
```

---

## 3. 📚 Permission Processing Logic

### 🧮 Extract Permission Names

```ts
function getPermissionNames(permissions: PermissionDto[]): string[] {
  const names: string[] = [];
  
  permissions.forEach(permission => {
    if (permission.menuName && permission.actions) {
      addActionPermissions(names, permission.menuName, permission.actions);
    }

    permission.subMenus?.forEach(subMenu => {
      if (subMenu.menuName && subMenu.actions) {
        addActionPermissions(names, subMenu.menuName, subMenu.actions);
      }
    });
  });
  
  return names;
}
```

### 🔠 Format Permissions with Actions

```ts
function addActionPermissions(names: string[], module: string, actions: PermissionActions) {
  const ACTIONS = ['create', 'read', 'edit', 'delete', 'list', 'export', 'approve', 'download'];
  
  ACTIONS.forEach(action => {
    if (actions[action]) {
      names.push(`${module.toLowerCase()}:${action}`);
    }
  });
}
```

---

## 4. 🔐 Permission Management

### 📌 Get or Create Permission Record

```ts
async function getOrCreatePermission(name: string, userId: number) {
  const [module, action] = name.split(':');
  
  return prisma.permission.upsert({
    where: { name },
    update: {},
    create: {
      name,
      module,
      action,
      description: `Permission for ${name}`,
      created_by: { connect: { id: userId } }
    }
  });
}
```

### 🔗 Link Role to Permission

```ts
async function linkRoleToPermission(roleId: number, permissionId: number, userId: number) {
  return prisma.rolePermission.create({
    data: {
      role_id: roleId,
      permission_id: permissionId,
      created_by: userId
    }
  });
}
```

---

## Usage Example

```typescript
const newRole = await createRole({
  role_code: 'ADMIN',
  role_name: 'Administrator',
  permissions: [{
    menuName: 'Users',
    actions: { create: true, read: true, edit: true, delete: true },
    subMenus: [{
      menuName: 'Permissions',
      actions: { read: true, edit: true }
    }]
  }]
}, request);
```

---

## ✅ Summary

- Dynamically creates roles and assigns permissions
- Supports nested menus and sub-menus
- Prevents duplication using `upsert`
- Tracks creator for auditability
- Easy to maintain and extend

