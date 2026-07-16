# Expo + Supabase Auth App — Complete Setup Guide

Flow covered:
- Register (email + password) → Verify Email screen → user taps link in email → redirected into app → auto logged in → Home
- Login (email + password) → Home
- Logout → Login screen
- Session persistence + automatic token refresh

No prior Supabase knowledge assumed — follow top to bottom.

---

# PART 1 — Supabase Dashboard Setup

### 1.1 Create a project
1. Go to https://supabase.com → sign in → "New Project".
2. Pick a name, database password, region → Create. Wait ~2 min for provisioning.

### 1.2 Get your API keys
1. In your project, go to **Project Settings → API**.
2. Copy:
   - **Project URL** (e.g. `https://xxxxx.supabase.co`)
   - **anon public key**
3. You'll paste these into the Expo app in Part 2.

### 1.3 Email auth is on by default
Go to **Authentication → Providers → Email**. Make sure it's enabled (it is by default). Leave "Confirm email" turned ON — this is what forces the verify-email step.

### 1.4 URL Configuration (critical for deep linking)
Go to **Authentication → URL Configuration**.

- **Site URL**: 
  ```
  myauthapp://auth-callback
  ```
- **Redirect URLs** (click "Add URL" and add these):
  ```
  myauthapp://auth-callback
  myauthapp://*
  ```

(`myauthapp` is the custom scheme we'll define in `app.json` in Part 2 — you can rename it, just keep it consistent everywhere.)

### 1.5 Custom "Confirm signup" email template
Go to **Authentication → Email Templates → Confirm signup**. Replace the body with:

```html
<h2>Confirm your email</h2>
<p>Hi,</p>
<p>Thanks for signing up! Tap the button below to verify your email and open the app.</p>
<p>
  <a href="{{ .ConfirmationURL }}"
     style="background:#4F46E5;color:#ffffff;padding:12px 24px;border-radius:8px;
            text-decoration:none;display:inline-block;font-family:sans-serif;">
    Verify Email
  </a>
</p>
<p style="font-family:sans-serif;color:#555;font-size:13px;">
  If the button doesn't work, copy this link into your phone's browser:<br/>
  {{ .ConfirmationURL }}
</p>
<p style="font-family:sans-serif;color:#999;font-size:12px;">
  If you didn't create this account, you can ignore this email.
</p>
```

Do **not** change `{{ .ConfirmationURL }}` — Supabase generates it automatically and it already points back to your app via the redirect URL you set in 1.4.

Click **Save**.

That's the entire Supabase-side setup. Everything else happens in code.

---

# PART 2 — Expo Project Setup

### 2.1 Create the project (skip if you already have one)
```bash
npx create-expo-app myauthapp
cd myauthapp
```

### 2.2 Install dependencies
```bash
npx expo install expo-linking expo-constants
npx expo install @react-native-async-storage/async-storage react-native-url-polyfill
npx expo install react-native-screens react-native-safe-area-context
npm install @supabase/supabase-js @react-navigation/native @react-navigation/native-stack
```

### 2.3 Dev client (required — Expo Go does not support custom scheme deep links reliably)
```bash
npx expo install expo-dev-client
```
You'll run the app with:
```bash
npx expo run:ios
# or
npx expo run:android
```
(or use `eas build --profile development` if you don't have native tooling locally).

### 2.4 `app.json` — add the scheme + deep link config

Open `app.json` and merge this in:

```json
{
  "expo": {
    "name": "myauthapp",
    "slug": "myauthapp",
    "scheme": "myauthapp",
    "ios": {
      "bundleIdentifier": "com.yourcompany.myauthapp",
      "supportsTablet": true
    },
    "android": {
      "package": "com.yourcompany.myauthapp",
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [{ "scheme": "myauthapp" }],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    },
    "plugins": ["expo-dev-client"]
  }
}
```

> The `scheme` value MUST match what you used in Supabase's Redirect URLs (`myauthapp://...`).

---

# PART 3 — Project Structure

Create these files:

```
myauthapp/
├── App.tsx
├── app.json
├── lib/
│   └── supabase.ts
├── context/
│   └── AuthContext.tsx
├── navigation/
│   └── RootNavigator.tsx
└── screens/
    ├── LoginScreen.tsx
    ├── RegisterScreen.tsx
    ├── VerifyEmailScreen.tsx
    └── HomeScreen.tsx
```

---

# PART 4 — Code

### 4.1 `lib/supabase.ts`

```ts
import 'react-native-url-polyfill/auto';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createClient } from '@supabase/supabase-js';

// 🔑 Paste your own values from Supabase → Project Settings → API
const SUPABASE_URL = 'https://xxxxx.supabase.co';
const SUPABASE_ANON_KEY = 'your-anon-public-key';

export const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  auth: {
    storage: AsyncStorage,
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false, // we handle the URL manually on native
  },
});
```

---

### 4.2 `context/AuthContext.tsx`

Handles: session state, sign up, sign in, sign out, deep-link token capture, and auto refresh-on-foreground (recommended by Supabase for React Native).

```tsx
import React, {
  createContext,
  useContext,
  useEffect,
  useRef,
  useState,
} from 'react';
import { AppState, AppStateStatus } from 'react-native';
import * as Linking from 'expo-linking';
import { Session } from '@supabase/supabase-js';
import { supabase } from '../lib/supabase';

type AuthContextType = {
  session: Session | null;
  initializing: boolean;
  signUp: (email: string, password: string) => Promise<{ error: string | null }>;
  signIn: (email: string, password: string) => Promise<{ error: string | null }>;
  signOut: () => Promise<void>;
};

const AuthContext = createContext<AuthContextType | undefined>(undefined);

// Where Supabase should send users after they tap "Verify Email"
const REDIRECT_TO = Linking.createURL('auth-callback'); // -> myauthapp://auth-callback

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [session, setSession] = useState<Session | null>(null);
  const [initializing, setInitializing] = useState(true);

  // ---- 1. Load existing session on app start, and subscribe to changes ----
  useEffect(() => {
    supabase.auth.getSession().then(({ data }) => {
      setSession(data.session);
      setInitializing(false);
    });

    const { data: listener } = supabase.auth.onAuthStateChange((_event, newSession) => {
      setSession(newSession);
    });

    return () => listener.subscription.unsubscribe();
  }, []);

  // ---- 2. Auto refresh token when app comes to foreground / stop in background ----
  // This is Supabase's official recommendation for React Native apps.
  useEffect(() => {
    const onChange = (state: AppStateStatus) => {
      if (state === 'active') {
        supabase.auth.startAutoRefresh();
      } else {
        supabase.auth.stopAutoRefresh();
      }
    };
    const sub = AppState.addEventListener('change', onChange);
    return () => sub.remove();
  }, []);

  // ---- 3. Handle the deep link that arrives after email verification ----
  const handledUrls = useRef(new Set<string>());

  const handleIncomingUrl = async (url: string | null) => {
    if (!url || handledUrls.current.has(url)) return;
    handledUrls.current.add(url);

    // Supabase can return tokens either as a query string (?access_token=...)
    // or a URL fragment (#access_token=...) depending on the flow.
    const fragment = url.split('#')[1];
    const queryString = url.split('?')[1]?.split('#')[0];

    const params = new URLSearchParams(fragment || queryString || '');
    const access_token = params.get('access_token');
    const refresh_token = params.get('refresh_token');

    if (access_token && refresh_token) {
      await supabase.auth.setSession({ access_token, refresh_token });
    }
  };

  useEffect(() => {
    // App opened directly via the link (cold start)
    Linking.getInitialURL().then(handleIncomingUrl);

    // App already running in background, link received
    const sub = Linking.addEventListener('url', ({ url }) => handleIncomingUrl(url));
    return () => sub.remove();
  }, []);

  // ---- 4. Auth actions ----
  const signUp = async (email: string, password: string) => {
    const { error } = await supabase.auth.signUp({
      email,
      password,
      options: { emailRedirectTo: REDIRECT_TO },
    });
    return { error: error?.message ?? null };
  };

  const signIn = async (email: string, password: string) => {
    const { error } = await supabase.auth.signInWithPassword({ email, password });
    return { error: error?.message ?? null };
  };

  const signOut = async () => {
    await supabase.auth.signOut();
  };

  return (
    <AuthContext.Provider value={{ session, initializing, signUp, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used inside AuthProvider');
  return ctx;
}
```

---

### 4.3 `navigation/RootNavigator.tsx`

Switches between the Auth stack (Login/Register/VerifyEmail) and the App stack (Home) based on whether a session exists.

```tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { ActivityIndicator, View } from 'react-native';
import { useAuth } from '../context/AuthContext';

import LoginScreen from '../screens/LoginScreen';
import RegisterScreen from '../screens/RegisterScreen';
import VerifyEmailScreen from '../screens/VerifyEmailScreen';
import HomeScreen from '../screens/HomeScreen';

export type AuthStackParamList = {
  Login: undefined;
  Register: undefined;
  VerifyEmail: { email: string };
};

export type AppStackParamList = {
  Home: undefined;
};

const AuthStack = createNativeStackNavigator<AuthStackParamList>();
const AppStack = createNativeStackNavigator<AppStackParamList>();

function AuthNavigator() {
  return (
    <AuthStack.Navigator screenOptions={{ headerShown: false }}>
      <AuthStack.Screen name="Login" component={LoginScreen} />
      <AuthStack.Screen name="Register" component={RegisterScreen} />
      <AuthStack.Screen name="VerifyEmail" component={VerifyEmailScreen} />
    </AuthStack.Navigator>
  );
}

function AppNavigator() {
  return (
    <AppStack.Navigator screenOptions={{ headerShown: false }}>
      <AppStack.Screen name="Home" component={HomeScreen} />
    </AppStack.Navigator>
  );
}

export default function RootNavigator() {
  const { session, initializing } = useAuth();

  if (initializing) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  return (
    <NavigationContainer>
      {session ? <AppNavigator /> : <AuthNavigator />}
    </NavigationContainer>
  );
}
```

Note: once the verification deep link sets the session (in `AuthContext`), `session` becomes non-null automatically, so the navigator switches straight to `Home` — no manual navigation call needed after verification.

---

### 4.4 `screens/LoginScreen.tsx`

```tsx
import React, { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, StyleSheet, Alert } from 'react-native';
import { useAuth } from '../context/AuthContext';

export default function LoginScreen({ navigation }: any) {
  const { signIn } = useAuth();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const onLogin = async () => {
    if (!email || !password) {
      Alert.alert('Missing info', 'Please enter email and password.');
      return;
    }
    setLoading(true);
    const { error } = await signIn(email.trim(), password);
    setLoading(false);
    if (error) Alert.alert('Login failed', error);
    // On success, RootNavigator auto-switches to Home via session state.
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Log In</Text>

      <TextInput
        style={styles.input}
        placeholder="Email"
        autoCapitalize="none"
        keyboardType="email-address"
        value={email}
        onChangeText={setEmail}
      />
      <TextInput
        style={styles.input}
        placeholder="Password"
        secureTextEntry
        value={password}
        onChangeText={setPassword}
      />

      <TouchableOpacity style={styles.button} onPress={onLogin} disabled={loading}>
        <Text style={styles.buttonText}>{loading ? 'Logging in...' : 'Log In'}</Text>
      </TouchableOpacity>

      <TouchableOpacity onPress={() => navigation.navigate('Register')}>
        <Text style={styles.link}>Don't have an account? Register</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', padding: 24, backgroundColor: '#fff' },
  title: { fontSize: 28, fontWeight: '700', marginBottom: 24, textAlign: 'center' },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 14,
    marginBottom: 12,
    fontSize: 16,
  },
  button: {
    backgroundColor: '#4F46E5',
    borderRadius: 8,
    padding: 14,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  link: { color: '#4F46E5', textAlign: 'center', marginTop: 18 },
});
```

---

### 4.5 `screens/RegisterScreen.tsx`

```tsx
import React, { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, StyleSheet, Alert } from 'react-native';
import { useAuth } from '../context/AuthContext';

export default function RegisterScreen({ navigation }: any) {
  const { signUp } = useAuth();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const onRegister = async () => {
    if (!email || !password) {
      Alert.alert('Missing info', 'Please enter email and password.');
      return;
    }
    if (password.length < 6) {
      Alert.alert('Weak password', 'Password must be at least 6 characters.');
      return;
    }
    setLoading(true);
    const { error } = await signUp(email.trim(), password);
    setLoading(false);

    if (error) {
      Alert.alert('Registration failed', error);
      return;
    }
    navigation.replace('VerifyEmail', { email: email.trim() });
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Create Account</Text>

      <TextInput
        style={styles.input}
        placeholder="Email"
        autoCapitalize="none"
        keyboardType="email-address"
        value={email}
        onChangeText={setEmail}
      />
      <TextInput
        style={styles.input}
        placeholder="Password"
        secureTextEntry
        value={password}
        onChangeText={setPassword}
      />

      <TouchableOpacity style={styles.button} onPress={onRegister} disabled={loading}>
        <Text style={styles.buttonText}>{loading ? 'Creating...' : 'Register'}</Text>
      </TouchableOpacity>

      <TouchableOpacity onPress={() => navigation.navigate('Login')}>
        <Text style={styles.link}>Already have an account? Log In</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', padding: 24, backgroundColor: '#fff' },
  title: { fontSize: 28, fontWeight: '700', marginBottom: 24, textAlign: 'center' },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 14,
    marginBottom: 12,
    fontSize: 16,
  },
  button: {
    backgroundColor: '#4F46E5',
    borderRadius: 8,
    padding: 14,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  link: { color: '#4F46E5', textAlign: 'center', marginTop: 18 },
});
```

---

### 4.6 `screens/VerifyEmailScreen.tsx`

Shown right after registering. Just tells the user to check their inbox — the actual "logged in" transition happens automatically once they tap the email link (handled by `AuthContext` + `RootNavigator`).

```tsx
import React from 'react';
import { View, Text, StyleSheet, TouchableOpacity } from 'react-native';

export default function VerifyEmailScreen({ route, navigation }: any) {
  const { email } = route.params;

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Check your email 📩</Text>
      <Text style={styles.text}>
        We sent a verification link to{'\n'}
        <Text style={styles.email}>{email}</Text>
      </Text>
      <Text style={styles.subText}>
        Tap the "Verify Email" button in that email — it will open this app and log you in
        automatically.
      </Text>

      <TouchableOpacity style={styles.button} onPress={() => navigation.replace('Login')}>
        <Text style={styles.buttonText}>Back to Login</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 24, backgroundColor: '#fff' },
  title: { fontSize: 24, fontWeight: '700', marginBottom: 16, textAlign: 'center' },
  text: { fontSize: 16, textAlign: 'center', marginBottom: 12 },
  email: { fontWeight: '700' },
  subText: { fontSize: 14, color: '#666', textAlign: 'center', marginBottom: 24 },
  button: { backgroundColor: '#4F46E5', borderRadius: 8, padding: 14, paddingHorizontal: 32 },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

### 4.7 `screens/HomeScreen.tsx`

```tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useAuth } from '../context/AuthContext';

export default function HomeScreen() {
  const { session, signOut } = useAuth();

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Welcome 👋</Text>
      <Text style={styles.email}>{session?.user.email}</Text>

      <TouchableOpacity style={styles.button} onPress={signOut}>
        <Text style={styles.buttonText}>Logout</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 24, backgroundColor: '#fff' },
  title: { fontSize: 28, fontWeight: '700', marginBottom: 8 },
  email: { fontSize: 16, color: '#666', marginBottom: 32 },
  button: { backgroundColor: '#EF4444', borderRadius: 8, padding: 14, paddingHorizontal: 32 },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

### 4.8 `App.tsx`

```tsx
import React from 'react';
import { AuthProvider } from './context/AuthContext';
import RootNavigator from './navigation/RootNavigator';

export default function App() {
  return (
    <AuthProvider>
      <RootNavigator />
    </AuthProvider>
  );
}
```

---

# PART 5 — How Logout Works

`signOut()` in `AuthContext` calls `supabase.auth.signOut()`, which clears the session and fires `onAuthStateChange` with `session = null`. `RootNavigator` is watching `session` — the moment it becomes `null`, it renders `AuthNavigator` instead of `AppNavigator`, which lands the user back on the `Login` screen. No manual navigation call needed.

---

# PART 6 — How Session & Refresh Token Are Managed

- `persistSession: true` + AsyncStorage → session survives app restarts.
- `autoRefreshToken: true` → Supabase JS refreshes the access token automatically while the app is active.
- The `AppState` listener in `AuthContext` calls `startAutoRefresh()` / `stopAutoRefresh()` when the app goes to foreground/background — this is Supabase's official recommendation for React Native so refresh timers don't run pointlessly in the background.
- `onAuthStateChange` keeps the `session` state in the context always up to date (login, logout, token refresh, and the post-verification `setSession` call all flow through it).

---

# PART 7 — Testing the Full Flow

1. Fill in your real Supabase URL/anon key in `lib/supabase.ts`.
2. Build a dev client and run on a real device or simulator:
   ```bash
   npx expo run:ios
   # or
   npx expo run:android
   ```
3. Tap **Register**, enter an email you can access + a password.
4. You'll land on **Verify Email**. Open your inbox.
5. Tap **Verify Email** in the email → the app opens → you should land directly on **Home** (auto-logged in).
6. Kill and reopen the app → you should stay logged in (session persisted).
7. Tap **Logout** → you should land back on **Login**.
8. Log in again with the same credentials → should go straight to **Home**.

---

# Common Gotchas

- **Nothing happens when tapping the email link** → Expo Go doesn't support this; you must use a dev client (`expo-dev-client`) or a full build.
- **`access_token` not found** → some email clients "wrap" links and strip the fragment; if this happens often, switch to the Universal Links (https) approach instead of a custom scheme — ask me and I'll set that up too.
- **Redirect URL mismatch error from Supabase** → double check the scheme in `app.json`, `AuthContext.tsx` (`Linking.createURL`), and the Supabase Dashboard → Redirect URLs all match exactly.


https://github.com/Lugdu84/supabase-expo-todos
