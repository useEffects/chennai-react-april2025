---
author: Joel Samuel
date: 6 april 2025
---
# Building Universal Apps with React: One Codebase, Every Platform!

```
Joel Samuel Raj A
https://useEffects.vercel.app
Software Engineer C1X Inc.
```

---
# Why Companies Choose React Native for Universal Apps
- Companies like Meta, Shopify, Discord, Microsoft, and Coinbase use React Native.
- Their web platforms are already built with React, making React Native a natural extension.
- React being a library (not a full framework) enables reuse of core business logic, UI components, and architectural patterns across web and mobile.
- Speeds up development cycles, improves consistency, and reduces context switching for developers.

---
# But can you use React?
## What the Official React Docs Say?
Quotes from the official docs:
>   "React is a library. It lets you put components together, but it doesn’t prescribe how to do routing and data fetching. To build an entire app with React, we recommend a full-stack React framework like Next.js or Remix."    
    <br />
    "We believe that the best way to experience React Native is through a Framework, a toolbox with all the necessary APIs to let you build production ready apps."

---
# Presentation Agenda
- Reference Blog post: https://blogs.codingclubgct.in/useEffects/building-universal-apps/
- UI
- Data Fetching
- Targeting Native Libraries 

---
# Our Approach
- React as a flexible UI library.
- Monorepo with Turborepo for clean separation of platform and shared code.
- Core UI Components live in a shared package (packages/app).
- Use platform-specific navigation/rendering logic (React Navigation for Expo, App Router for Next.js).

---
# Core UI Components
Have all your core UI components and screens in a package outside of the Next.js and Expo apps. 
```tsx
// packages/app/screens/foobar.tsx
export default function Foobar() {
    return (
        <View>
            <Text>Foobar</Text>
        </View>
    );
}
```

---
# Usage in Next.js
Use the shared components in your Next.js app like this: 
```tsx
// apps/next/src/app/foobar/page.tsx
export { Foobar as default } from "app/screens/foobar";
```

---
# Usage in Expo
Use the shared components in your Expo app like this: 
```tsx
// apps/expo/app/foobar/index.tsx
export { Foobar as default } from "app/screens/foobar";
```

---
# Optimizing Data Fetching in Next.js and Expo
## Next.js Benefits
Server Components: Offload heavy computation or data fetching to the server.

## Expo Benefits
Offline Support with @tanstack/react-query: Cache-first data fetching with background sync.

---
# Anti pattern!
```tsx
export default function Foobar({ id }: { id: string }) {
    const [data, setData] = useState(null);
    useEffect(() => {
        fetchData(id).then(setData);
    }, [id]);

    return data && (
        <View>
            <Hamburger data={data} />
        </View>
    );
}
```

---
# Our design choice! Smart and Dumb components
```tsx
export default function Foobar({ initialData }: { initialData: Data }) {
    return <Hamburger data={initialData} />;
}
```

---
# Next.js: Server Components and Incremental Static Generation
```tsx
// apps/next/src/app/foobar/page.tsx
import { Suspense } from 'react';
import { fetchData, fetchAllData } from 'app/lib/data-fetching';
import Foobar from 'app/screens/foobar';

export default function FoobarPage({ params: { id } }: { params: { id: string } }) {
    const data = fetchData(id);

    return (
        <Suspense fallback={<div>Loading...</div>}>
            <Foobar initialData={data} />
        </Suspense>
    );
}

// generateStaticParams example
export async function generateStaticParams() {
    const data = await fetchAllData();
    return data.map(item => ({ id: item.id }));
}
```

---
# Building Offline Apps in Expo with React Query
```tsx
// apps/expo/app/foobar/index.tsx
import { useQuery } from '@tanstack/react-query';
import Foobar from 'app/screens/foobar';
import { fetchData } from 'app/lib/data-fetching';

export default function FoobarScreen() {
    const { id } = useParams<{ id: string }>();
    const { data, isLoading } = useQuery({
        queryKey: [id],
        queryFn: () => fetchData(id),
        initialData: [],
        staleTime: 1000 * 60 * 60 * 24 * 7, // 1 week
        gcTime: 1000 * 60 * 60 * 24 * 7, // 1 week
    });

    if (isLoading) return <View><Text>Loading...</Text></View>;

    return <Foobar initialData={data} />;
}
```

---
# Leveraging Platform-Specific Code and Native Libraries in shared codebase
Consider the case of applying themes to the app,
- In Next.js we use the next-themes library.
- In Expo we use the useColorScheme hook from React Native.

We make use of the strategy pattern in building UI components that make use of platform specific code

---
# Building the ToggleTheme component.
```tsx
// packages/components/toggle-theme/base

export default function ToggleThemeBase ({toggleTheme}: {toggleTheme: () => void}) {
    return <Button onPress={toggleTheme}>
        <Text>Toggle Theme</Text>
    </Button>
}
```

---
# Make use of Metro’s file extensions for bundling platform specific code:
```tsx
// packages/components/toggle-theme.tsx

import ToggleThemeBase from "./base"

export default function ToggleTheme () {
    const { toggleTheme } = useColorScheme()
    return <ToggleThemeBase toggleTheme={toggleTheme}>
}
```

---
For the web:

```tsx
// packages/components/toggle-theme.web.tsx

import ToggleThemeBase from "./base"

export default function ToggleTheme () {
    const { toggleTheme } = useContext(ThemeContext)
    return <ToggleThemeBase toggleTheme={toggleTheme}>
}
```

---
# Handling Framework-Specific Code: Routing and Beyond
## What Can’t Be Shared and Why That’s Okay
While we aim to share as much code as possible, certain parts—like routing, navigation, and app lifecycle management—are framework-specific.
## Example: Routing
- Next.js uses the App Router and file-based routing system.
- Expo (React Native) uses React Navigation with stack, tab, and drawer navigators.

---
# The Context API Pattern
## Abstracting useRouter with React Context

```tsx
// packages/app/router.ts
export interface Router {
  push: (path: string) => void;
  back: () => void;
}

export const RouterContext = React.createContext<Router | null>(null);

export const useRouter = (): Router => {
  const ctx = React.useContext(RouterContext);
  if (!ctx) throw new Error("RouterContext not provided");
  return ctx;
};
```

---
## Next.js 
```tsx
// apps/next/src/app/providers/router-provider.tsx
import { useRouter as useNextRouter } from "next/navigation";
import { RouterContext } from "app/router";

export function RouterProvider({ children }) {
  const router = useNextRouter();
  return (
    <RouterContext.Provider value={{ push: router.push, back: router.back }}>
      {children}
    </RouterContext.Provider>
  );
}
```

---
## Expo
```tsx
// apps/expo/app/_layout.tsx
import { useNavigation } from "@react-navigation/native";
import { RouterContext } from "app/router";

export function RouterProvider({ children }) {
  const nav = useNavigation();
  return (
    <RouterContext.Provider value={{ push: nav.navigate, back: nav.goBack }}>
      {children}
    </RouterContext.Provider>
  );
}
```

---
# Thank you!