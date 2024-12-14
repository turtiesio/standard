# RBAC Permission

요구 사항

1. 기본 권한 시스템

   - Permission은 리소스별로 권한을 가짐
   - Permission은 계층 구조를 가짐 (상위 권한이 하위 권한을 포함)
   - 모든 접근 제어는 순수하게 Permission 기반으로 동작

2. 리소스 관리

   - 각 리소스는 고유한 ID를 가짐 (Azure 스타일)
   - 리소스 종류: workspace, platform accounts (youtube, tiktok, instagram), member 등
   - 리소스 ID는 계층 구조를 가짐 (예: /workspaces/{id}/platforms/{platform}/accounts/{accountId})

3. Workspace 기반 구조

   - SaaS 시스템으로 여러 workspace 존재
   - 각 workspace는 여러 platform account 보유 가능
   - WorkspaceMember 테이블로 멤버 관리

4. Platform Account 접근 제어

   - 특정 계정에만 접근 권한 부여 가능 (예: 666_YOUTUBE만 접근)
   - 와일드카드 패턴 지원
     - "666\_\*": 특정 계정의 모든 플랫폼
     - "\*\_YOUTUBE": 모든 YouTube 계정
     - "\_\_\_": 모든 계정
   - Platform별로 세분화된 권한 관리 (read, write, manage)

5. Permission 계층 구조
   - System 레벨 권한 (system:admin, system:read_workspace 등)
   - Workspace 레벨 권한 (workspace:admin, workspace:manage_member 등)
   - Platform Account 레벨 권한 (platform_account:manage, write, read)

```ts
// Permission 정의
const PERMISSIONS = {
  SYSTEM: {
    ADMIN: "system:admin",
    READ_WORKSPACE: "system:read_workspace",
    MANAGE_WORKSPACE: "system:manage_workspace",
  },
  WORKSPACE: {
    ADMIN: "workspace:admin",
    MANAGE_MEMBER: "workspace:manage_member",
    READ: "workspace:read",
  },
  PLATFORM_ACCOUNT: {
    MANAGE: "platform_account:manage",
    WRITE: "platform_account:write",
    READ: "platform_account:read",
  },
} as const;

// Permission 트리 구조 정의
const PERMISSION_TREE = {
  // System 권한
  [PERMISSIONS.SYSTEM.ADMIN]: {
    description: "시스템 전체 관리자 권한",
    includes: [
      {
        permission: PERMISSIONS.SYSTEM.MANAGE_WORKSPACE,
        description: "워크스페이스 생성/삭제 권한",
      },
      {
        permission: PERMISSIONS.SYSTEM.READ_WORKSPACE,
        description: "워크스페이스 목록 조회 권한",
      },
      {
        permission: PERMISSIONS.WORKSPACE.ADMIN,
        description: "모든 워크스페이스의 관리자 권한",
      },
    ],
  },

  // Workspace 권한
  [PERMISSIONS.WORKSPACE.ADMIN]: {
    description: "워크스페이스 관리자 권한",
    includes: [
      {
        permission: PERMISSIONS.WORKSPACE.MANAGE_MEMBER,
        description: "멤버 관리 권한",
      },
      {
        permission: PERMISSIONS.WORKSPACE.READ,
        description: "워크스페이스 읽기 권한",
      },
      {
        permission: PERMISSIONS.PLATFORM_ACCOUNT.MANAGE,
        description: "플랫폼 계정 관리 권한",
      },
    ],
  },

  // Platform Account 권한
  [PERMISSIONS.PLATFORM_ACCOUNT.MANAGE]: {
    description: "플랫폼 계정 관리 권한",
    includes: [
      {
        permission: PERMISSIONS.PLATFORM_ACCOUNT.WRITE,
        description: "플랫폼 계정 쓰기 권한",
      },
    ],
  },

  [PERMISSIONS.PLATFORM_ACCOUNT.WRITE]: {
    description: "플랫폼 계정 쓰기 권한",
    includes: [
      {
        permission: PERMISSIONS.PLATFORM_ACCOUNT.READ,
        description: "플랫폼 계정 읽기 권한",
      },
    ],
  },

  // Leaf permissions (no includes)
  [PERMISSIONS.SYSTEM.READ_WORKSPACE]: {
    description: "워크스페이스 목록 조회 권한",
    includes: [],
  },
  [PERMISSIONS.SYSTEM.MANAGE_WORKSPACE]: {
    description: "워크스페이스 생성/삭제 권한",
    includes: [],
  },
  [PERMISSIONS.WORKSPACE.MANAGE_MEMBER]: {
    description: "멤버 관리 권한",
    includes: [],
  },
  [PERMISSIONS.WORKSPACE.READ]: {
    description: "워크스페이스 읽기 권한",
    includes: [],
  },
  [PERMISSIONS.PLATFORM_ACCOUNT.READ]: {
    description: "플랫폼 계정 읽기 권한",
    includes: [],
  },
} as const;

// 트리 구조에서 계층 관계 맵 생성
function buildPermissionHierarchy(
  tree: typeof PERMISSION_TREE
): Record<PermissionId, PermissionId[]> {
  const hierarchy: Record<string, string[]> = {};

  function traverseTree(
    permission: string,
    visited: Set<string> = new Set()
  ): string[] {
    if (visited.has(permission)) {
      throw new Error(
        `Circular dependency detected for permission: ${permission}`
      );
    }

    if (!tree[permission]) {
      throw new Error(`Permission not found in tree: ${permission}`);
    }

    visited.add(permission);
    const node = tree[permission];
    const implied: string[] = [];

    for (const include of node.includes) {
      implied.push(include.permission);
      implied.push(...traverseTree(include.permission, new Set(visited)));
    }

    return Array.from(new Set(implied));
  }

  // 각 permission에 대해 계층 관계 구축
  Object.keys(tree).forEach((permission) => {
    hierarchy[permission] = traverseTree(permission);
  });

  return hierarchy;
}

// 계층 관계 맵 생성
const PERMISSION_HIERARCHY = buildPermissionHierarchy(PERMISSION_TREE);

// Permission 관계 시각화 (디버깅/문서화 용도)
function visualizePermissionHierarchy(
  tree: typeof PERMISSION_TREE,
  permission: string,
  depth = 0
) {
  const node = tree[permission];
  const indent = "  ".repeat(depth);
  let result = `${indent}${permission} (${node.description})\n`;

  for (const include of node.includes) {
    result += visualizePermissionHierarchy(tree, include.permission, depth + 1);
  }

  return result;
}

// 사용 예시
const hierarchyVisualization = visualizePermissionHierarchy(
  PERMISSION_TREE,
  PERMISSIONS.SYSTEM.ADMIN
);
console.log("Permission Hierarchy:");
console.log(hierarchyVisualization);

class PermissionManager {
  constructor(private workspaceId: string) {}

  private async hasImpliedPermission(
    permissions: PermissionId[],
    requiredPermission: PermissionId
  ): Promise<boolean> {
    if (permissions.includes(requiredPermission)) return true;

    return permissions.some((permission) => {
      const impliedPermissions = PERMISSION_HIERARCHY[permission] || [];
      return impliedPermissions.includes(requiredPermission);
    });
  }

  // ... 나머지 메서드들은 동일
}
```
