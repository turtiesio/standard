# RBAC Permission

[코드 예시](#코드-예시)

## 요구사항

1. Permission 기반의 권한 관리 시스템

   - Role이 아닌 순수 Permission 기반으로 동작
   - Permission 간의 계층 구조 지원 (상위 권한이 하위 권한을 포함)

2. Resource 단위의 권한 관리

   - 각 Resource는 고유 ID를 가짐 (Azure 스타일)
   - Workspace 기반 멀티테넌시 지원
   - Platform Account 연동 (YouTube, TikTok, Instagram)

3. Workspace & Platform Account

   - 하나의 Workspace가 여러 플랫폼 계정 보유 가능
   - 특정 플랫폼 계정에 대한 선별적 접근 권한 부여 가능
   - 와일드카드를 통한 유연한 권한 관리 (예: "666\__", "_\_YOUTUBE")

4. DB 구조
   - WorkspaceMember 테이블에 권한 정보 저장
   - Platform Account 정보 저장

## 코드 예시

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

//
//
//

// 2. Resource Types & Platform Account IDs
type ResourceType =
  | "workspace"
  | "youtube"
  | "tiktok"
  | "instagram"
  | "member"
  | "system";

type PlatformAccountId = `${string}_${(typeof SYSTEM.PLATFORM_TYPES)[number]}`;
type WildcardPattern =
  | `${string}_${(typeof SYSTEM.PLATFORM_TYPES)[number] | "*"}`
  | `${string | "*"}_${(typeof SYSTEM.PLATFORM_TYPES)[number]}`;

// 3. DB Schema
interface DBWorkspaceMember {
  id: string;
  workspaceId: string;
  userId: string;
  permissions: PermissionId[]; // 기본 권한 목록
  platformPermissions: {
    pattern: WildcardPattern;
    permissions: PermissionId[];
  }[];
  createdAt: Date;
  updatedAt: Date;
}

interface DBPlatformAccount {
  id: PlatformAccountId;
  workspaceId: string;
  platform: (typeof SYSTEM.PLATFORM_TYPES)[number];
  accountId: string;
  name: string;
  accessToken: string;
  refreshToken: string;
  createdAt: Date;
  updatedAt: Date;
}

// 4. Permission Manager
class PermissionManager {
  constructor(private workspaceId: string) {}

  // 권한 체크 (계층 구조 고려)
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

  // 와일드카드 패턴 매칭
  private matchesPattern(
    accountId: PlatformAccountId,
    pattern: WildcardPattern
  ): boolean {
    if (pattern === "*_*") return true;

    const [patternAccount, patternPlatform] = pattern.split("_");
    const [targetAccount, targetPlatform] = accountId.split("_");

    return (
      (patternAccount === "*" || patternAccount === targetAccount) &&
      (patternPlatform === "*" || patternPlatform === targetPlatform)
    );
  }

  // 플랫폼 계정 접근 권한 체크
  async checkPlatformAccountAccess(
    memberId: string,
    accountId: PlatformAccountId,
    requiredPermission: PermissionId
  ): Promise<boolean> {
    const member = await this.findWorkspaceMember(memberId);

    // workspace:admin 권한이 있다면 모든 플랫폼 접근 가능
    if (await this.hasPermission(memberId, PERMISSIONS.WORKSPACE.ADMIN)) {
      return true;
    }

    // 매칭되는 패턴에 대한 권한 확인
    for (const perm of member.platformPermissions) {
      if (this.matchesPattern(accountId, perm.pattern)) {
        if (
          await this.hasImpliedPermission(perm.permissions, requiredPermission)
        ) {
          return true;
        }
      }
    }

    return false;
  }
}

// 5. 사용 예시
async function example() {
  const permManager = new PermissionManager("ws_123");

  // 특정 YouTube 계정만 접근 허용
  const canAccessYoutube = await permManager.checkPlatformAccountAccess(
    "user_789",
    "666_YOUTUBE",
    PERMISSIONS.PLATFORM_ACCOUNT.READ
  );

  // 특정 계정의 모든 플랫폼 접근 허용
  const member = {
    platformPermissions: [
      {
        pattern: "666_*",
        permissions: [PERMISSIONS.PLATFORM_ACCOUNT.READ],
      },
    ],
  };
}
```
