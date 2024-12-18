# Project Configuration

- **Package Manager:** `yarn` (strictly enforced).
- **Logging:** `pino` (strictly enforced). Implement structured logging as detailed below.
- **Documentation:** Use JSDoc-style comments for all components, functions, and relevant data structures.
- **Testing:** Use vitest and React Testing Library for unit testing.
- **Style:** Use Tailwind CSS for styling and component library for UI components.
- **Import:** Use absolute imports for types and components in Next.js projects.(@/\*)

## Coding Standards

1. **Architecture Principles**

   - **Component-Based Architecture:** Design the UI with reusable components.
   - **Separation of Concerns:** Separate business logic, UI rendering, and data fetching into distinct parts of your components.
   - **Client-Side State Management (Zustand):** Use Zustand for managing application state that is specific to the client-side, including UI state (e.g. modal open/close), user interaction, etc.
   - **Server-Side State Management & Data Fetching (React Query):** Utilize React Query to handle data fetching, caching, mutations, and optimistic updates when interacting with server APIs.
   - **Clean Code Practices:** Follow clean coding principles including readability, maintainability, and avoiding code duplication.
   - **KISS (Keep It Simple, Stupid):** Prioritize simplicity in design and implementation.
   - **DRY (Don't Repeat Yourself):** Avoid repeating code, create reusable functions and components.
   - **YAGNI (You Aren't Gonna Need It):** Avoid adding functionality until it's required.

2. **Code Documentation**

   - **JSDoc Comments:** Use JSDoc-style comments for:
     - All functions
     - React components (functional and class-based)
     - Types and interfaces
     - Data structures (e.g., objects that represent complex data models)
     - Zustand store definitions
     - React Query query and mutation definitions
   - **Clear Naming Conventions:**
     - **Functions:** Use descriptive, verb-noun naming (e.g., `fetchUserData`, `handleButtonClick`).
     - **Variables:** Use `camelCase` and choose descriptive names.
     - **Components:** Use `PascalCase` for component filenames and names (e.g., `UserList.tsx`, `UserProfile.jsx`).
     - **Files:** Use `kebab-case` for file and directory names (e.g., `user-profile.tsx`, `api-services/`, `stores/`).
   - **Component File Structure:** For each component, maintain a single file structure:
     - `ComponentName.tsx` or `ComponentName.jsx` for the component logic.
     - `ComponentName.module.css` (or scss) for related styling.

3. **Testing Framework**

   - **Unit Tests:** Use Jest and React Testing Library.
   - **Component Tests:** Test component behavior including props, event handlers and interaction logic using React Testing Library.
   - **Test files Naming:** `*.test.tsx` or `*.test.jsx`
   - **Mocking Server Interactions:** Mock API calls made by React Query using tools like `msw` (Mock Service Worker) for reliable testing.

4. **Development Tools**

   - **TypeScript:** Use TypeScript (strict mode enabled).
   - **ESLint + Prettier:** Configure ESLint and Prettier to enforce consistent code style. Integrate with your editor for automated formatting and linting.
   - **Husky for git hooks** (as configured in the project setup)
   - **Logging:** JSON format using `pino`. Configure as outlined below.

5. **`README.md`:**

   - Maintain a clear and up-to-date `README.md` file with instructions for development setup, running the application, and details on the project structure.

6. **Logging Standards**

   ```typescript
   {
       timestamp: ISO8601 string,
       level: 'log' | 'fatal' | 'error' | 'warn' | 'debug' | 'verbose',
       context: string, // component/module, etc that logs the information
       message: string,
       metadata: Record<string, any>
   }

   // fatal: 'Critical error requiring application shutdown',
   // error: 'Business logic failures, exceptions thrown',
   // warn: 'Potential issues, performance degradation',
   // info: 'Important business events',
   // debug: 'Detailed information for development debugging',
   // verbose: 'Most detailed application operational information'
   ```

   - JSON structured logging
   - Detailed metadata logging (e.g., IDs, request parameters when relevant).
   - Redact sensitive information before logging.
   - Use debug and verbose levels where appropriate for more granular information during development.

## Next.js Specifics

- **Data Fetching:**
  - **React Query for Server Data:** Use React Query for all server data fetching, caching, mutations, and invalidations. Avoid directly using fetch or similar methods for data from APIs.
  - **Next.js Data Fetching:** Use Next.js data fetching methods (`getServerSideProps`, `getStaticProps`, `getStaticPaths`) only when strictly necessary for pre-rendering pages. This is usually limited to initial data loading and SEO optimization. Most API interactions should happen via React Query inside components.
- **API Routes:** Use API routes only for server-side logic and direct data processing if necessary. Prefer consuming a dedicated backend API.
- **Image Optimization:** Utilize the `<Image>` component for optimized image loading.
- **Link Component:** Use the `<Link>` component for client-side routing.
- **Performance Optimization:** Pay attention to Next.js optimization techniques such as code splitting, lazy loading, and optimized bundles.

## Zustand & React Query Implementation

1. **Zustand for Client State:**

   - Define stores in `stores/` directory (e.g., `stores/userStore.ts`, `stores/uiStore.ts`).
   - Keep store logic clear and concise.
   - Use meaningful names for actions and state variables.

2. **React Query for Server State:**
   - **Queries:** Define query keys and query functions in service files (e.g., `services/userService.ts`).
   - **Mutations:** Define mutation functions for server-side data updates within services.
   - Use `useQuery`, `useMutation`, and related hooks for data fetching and mutation operations within components.
   - Implement error handling and loading states for React Query queries and mutations.
   - Set sensible `staleTime` and `cacheTime` options for queries.

## Makefile (Example - Adapt as needed)

reference package.json scripts for detailed commands

```makefile
build:
format:
dev:
start:
```

## Example Component with Zustand & React Query (TypeScript)

```typescript
import React, { useState, useEffect } from "react";
import styles from "./UserProfile.module.css";
import { useUserStore } from "../stores/userStore"; // Zustand store
import { useQuery } from "react-query";
import { fetchUser } from "../services/userService";

/**
 * UserProfile component to display a user's profile information.
 *
 * @returns {JSX.Element} The rendered UserProfile component.
 */
const UserProfile: React.FC = () => {
  const { setUserName } = useUserStore();
  const { data: user, isLoading, error } = useQuery("user", fetchUser);
  const [greeting, setGreeting] = useState("");

  useEffect(() => {
    if (user) {
      setGreeting(`Hello, ${user.name}!`);
      setUserName(user.name); // set user name to zustand store
    }
  }, [user, setUserName]);

  /**
   * Handles button click
   *
   * @param {MouseEvent<HTMLButtonElement>} event - The mouse event
   */
  const handleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
    console.log("Button Clicked");
  };

  if (isLoading) {
    return <div>Loading User Profile...</div>;
  }

  if (error) {
    return <div>Error loading user profile: {error.message}</div>;
  }

  return (
    <div className={styles.userProfileContainer}>
      <h2 className={styles.greeting}>{greeting}</h2>
      {user && (
        <>
          <p>
            <strong>ID:</strong> {user.id}
          </p>
          <p>
            <strong>Email:</strong> {user.email}
          </p>
          <button onClick={handleClick}>Click Me</button>
        </>
      )}
    </div>
  );
};

export default UserProfile;
```

**Example Store (stores/userStore.ts):**

```typescript
import { create } from "zustand";

interface UserState {
  userName: string | null;
  setUserName: (name: string) => void;
}

/**
 * User Store using zustand
 * @returns {UserState} The store methods
 */
export const useUserStore = create<UserState>((set) => ({
  userName: null,
  setUserName: (name) => set({ userName: name }),
}));
```

**Example Service (services/userService.ts):**

```typescript
import axios from "axios";

const BASE_URL = "https://api.example.com";

/**
 * Fetch User Data
 *
 * @returns {Promise<any>} The fetched user data
 */
export const fetchUser = async () => {
  const { data } = await axios.get(`${BASE_URL}/user`);
  return data;
};
```
