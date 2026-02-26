# BaekjoonHub Organization 지원 - 변경 사항 및 기술 문서

이 문서는 BaekjoonHub를 GitHub Organization 저장소에 접근·업로드할 수 있도록 커스터마이징한 변경 내용과 사용된 기술을 정리합니다.

---

## 목차

1. [변경 개요](#1-변경-개요)
2. [Phase 1: OAuth 앱 설정](#2-phase-1-oauth-앱-설정)
3. [Phase 2: 조직 저장소 목록 확장](#3-phase-2-조직-저장소-목록-확장)
4. [Phase 3: 조직 내 저장소 생성](#4-phase-3-조직-내-저장소-생성)
5. [Phase 4: 문서화 및 에러 처리](#5-phase-4-문서화-및-에러-처리)
6. [사용 기술 요약](#6-사용-기술-요약)

---

## 1. 변경 개요

### 배경

- **원본 BaekjoonHub**: 개인 계정의 저장소에만 접근 가능
- **문제**: GitHub Organization은 "Third-party application access policy"로 인해 OAuth 앱 승인이 필요
- **목적**: 코테 스터디 운영 시 조직 내 멤버별 저장소에 PS 풀이를 자동 업로드

### 변경된 파일 목록

| 파일 | 변경 내용 |
|------|----------|
| `src/scripts/constants/url.ts` | OAuth 자격 증명 환경 변수화 |
| `src/settings.ts` | 조직 저장소 목록, 생성 로직, 에러 메시지 |
| `src/settings.html` | 도움말 문구 추가 |
| `src/scripts/commons/github.ts` | 조직 저장소 API, 에러 처리 강화 |
| `src/scripts/commons/platformhub-base.ts` | 업로드 실패 시 상세 에러 메시지 표시 |
| `.env.example` | 환경 변수 예시 (신규) |
| `.gitignore` | `.env` 제외 추가 |

---

## 2. Phase 1: OAuth 앱 설정

### 기술: Vite 환경 변수 (import.meta.env)

Vite는 빌드 시 `import.meta.env.VITE_*` 변수를 클라이언트 코드에 주입합니다.

### 구현

**파일**: `src/scripts/constants/url.ts`

```typescript
GITHUB_CLIENT_ID:
  (import.meta as { env?: Record<string, string | undefined> }).env?.VITE_GITHUB_CLIENT_ID ||
  "975f8d5cf6686dd1faed",
GITHUB_CLIENT_SECRET:
  (import.meta as { env?: Record<string, string | undefined> }).env?.VITE_GITHUB_CLIENT_SECRET ||
  "934b2bfc3bb3ad239bc67bdfa81a378b1616dd1e",
```

- `VITE_` 접두사: Vite가 클라이언트에 노출하는 환경 변수에 필수
- `||` 폴백: 미설정 시 원본 BaekjoonHub 기본값 사용
- `import.meta.env`: ES 모듈 표준, Vite가 빌드 시 치환

### 사용 방법

1. `.env.example`을 `.env`로 복사
2. [GitHub OAuth App](https://github.com/settings/developers) 생성
3. `.env`에 `VITE_GITHUB_CLIENT_ID`, `VITE_GITHUB_CLIENT_SECRET` 설정
4. `npm run build` 실행

---

## 3. Phase 2: 조직 저장소 목록 확장

### 기술: GitHub REST API

| API | 용도 |
|-----|------|
| `GET /user/repos?affiliation=owner,collaborator,organization_member` | 개인 + 조직 저장소 통합 조회 |
| `GET /user/orgs` | 사용자가 속한 조직 목록 |
| `GET /orgs/{org}/repos` | 조직별 저장소 (user/repos에 누락된 경우 보완) |

### 구현

**파일**: `src/settings.ts`

#### 3.1 데이터 구조 확장

```typescript
interface GitHubRepository {
  name: string;
  fullName: string;
  description: string | null;
  private: boolean;
  ownerLogin?: string;   // 추가: 조직명 표시용
  isOrgRepo?: boolean;   // 추가: 조직 저장소 여부
}
```

#### 3.2 저장소 목록 수집 로직

1. **user/repos** 호출: `affiliation`으로 개인·협업·조직 저장소 포함
2. **403 응답**: 조직 OAuth 미승인 시 즉시 에러 메시지 반환
3. **user/orgs** 호출: 조직 목록 조회
4. **orgs/{org}/repos** 호출: 조직별로 저장소 추가 (중복 제거)
5. `owner.type === "Organization"`으로 조직 저장소 판별

#### 3.3 UI 표시

```typescript
const orgLabel = repo.isOrgRepo && repo.ownerLogin ? ` (${repo.ownerLogin})` : "";
option.textContent = `${repo.name}${orgLabel} ${repo.private ? "(비공개)" : ""}`;
```

- 조직 저장소: `repo-name (org-name) (비공개)` 형태로 표시

---

## 4. Phase 3: 조직 내 저장소 생성

### 기술: GitHub REST API

| API | 용도 |
|-----|------|
| `POST /orgs/{org}/repos` | 조직 내 새 저장소 생성 |
| `GET /repos/{owner}/{repo}` | 저장소 존재 여부 확인 |

### 구현

**파일**: `src/scripts/commons/github.ts`

#### 4.1 createOrgRepository()

```typescript
export async function createOrgRepository(
  org: string,
  name: string,
  token: string,
  options: { private?: boolean; description?: string } = {}
): Promise<{ full_name: string }>
```

- `auto_init: true`: 초기 커밋으로 빈 저장소 생성
- 기본값: `private: true`, `description: "PS solutions - BaekjoonHub"`

#### 4.2 repoExists()

```typescript
export async function repoExists(hook: string, token: string): Promise<boolean>
```

- `GET /repos/{hook}` 응답의 `response.ok`로 존재 여부 판단

#### 4.3 연결 플로우 (handleRepoConnection)

1. `owner/repo` 형식 파싱
2. `repoExists()`로 저장소 존재 확인
3. **미존재 시**:
   - `owner !== username` → 조직 저장소로 간주
   - `createOrgRepository()` 호출
   - 403 등 실패 시 에러 메시지 표시 후 중단
4. **개인 저장소** (`owner === username`)이고 미존재 시: GitHub에서 직접 생성하라는 안내

---

## 5. Phase 4: 문서화 및 에러 처리

### 5.1 checkResponseAndThrow() - github.ts

**기술**: Fetch API `Response` 검증 및 사용자 친화적 에러 메시지

```typescript
async function checkResponseAndThrow(response: Response, context: string): Promise<void> {
  if (response.ok) return;
  const body = await response.text();
  // ...
  if (response.status === 403) {
    throw new Error(
      `${context}: ${message}. 조직 저장소인 경우, 조직 관리자에게 OAuth 앱 승인을 요청하세요. (조직 설정 → Third-party access)`
    );
  }
  throw new Error(`${context}: ${message}`);
}
```

- `getDefaultBranchOnRepo`, `createOrUpdateFile`, `createOrgRepository`에서 사용
- 403 시 조직 OAuth 승인 안내 메시지 추가

### 5.2 FetchUserInfoResult - settings.ts

```typescript
interface FetchUserInfoResult {
  userInfo: GitHubUserInfo | null;
  errorMessage?: string;  // 403 등 시 구체적 안내
}
```

- `fetchGitHubUserInfo()`가 403 시 `errorMessage` 반환
- UI에서 "조직 관리자에게 OAuth 앱 승인을 요청하세요" 등으로 표시

### 5.3 업로드 실패 메시지 - platformhub-base.ts

```typescript
} catch (error) {
  const message =
    error instanceof Error ? error.message : `${this.config.platformName} 업로드 중 오류가 발생했습니다.`;
  Toast.raiseToast(message);
}
```

- 기존: 고정 메시지만 표시
- 변경: `Error` 객체의 `message`를 그대로 토스트에 표시 (403 시 조직 안내 포함)

### 5.4 설정 화면 도움말 - settings.html

- 저장소 형식: `조직명/저장소명` 예시 추가
- 조직 저장소 사용 시: "조직 설정 → Third-party access에서 이 OAuth 앱을 승인해 주세요" 안내

---

## 6. 사용 기술 요약

| 분류 | 기술 | 용도 |
|------|------|------|
| 빌드 | Vite `import.meta.env` | OAuth 자격 증명 환경 변수 주입 |
| API | GitHub REST API v3 | 저장소, 조직, OAuth 연동 |
| 인증 | OAuth 2.0 (Authorization Code) | GitHub 토큰 발급 |
| 스토리지 | Chrome Storage API | 토큰, HOOK, 설정 저장 |
| 에러 처리 | `response.ok`, `response.status` | 403 등 API 실패 시 분기 |
| UI | DOM API, `fetch` | 설정 화면, 저장소 목록, 토스트 |

### GitHub API 엔드포인트 참조

- [Repositories](https://docs.github.com/en/rest/repos)
- [Organizations](https://docs.github.com/en/rest/orgs/orgs)
- [OAuth Apps](https://docs.github.com/en/apps/oauth-apps)

---

## 부록: 스터디 운영 시나리오

1. **운영자**: GitHub Organization 생성 → OAuth App 등록 → 조직 설정에서 앱 승인
2. **운영자**: 조직에 `member1-ps`, `member2-ps` 등 저장소 생성 (수동 또는 확장 프로그램에서)
3. **멤버**: BaekjoonHub 커스텀 빌드 설치 → OAuth 인증 → `org/member1-ps` 연결
4. **멤버**: 백준/프로그래머스에서 문제 풀이 → 자동으로 조직 저장소에 푸시
