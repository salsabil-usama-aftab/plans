# Apple Sign-In — Implementation Plan

**Goal:** Restrict the "Continue with Apple" button to iOS only, then implement native Apple Sign-In backed by Supabase for user-ID generation.

**Approach:** Use `@invertase/react-native-apple-authentication` to get an Apple identity token natively on device, then exchange it with Supabase via `signInWithIdToken`. This is the Apple-preferred native flow and is required by Apple's App Review guidelines when other social logins are offered.

---

## Phase 0 — iOS-only button (1 change, 5 min)

**File:** `project-ihsan/src/views/screens/AuthScreen/index.tsx`

Change the `screenConfig` entry from:

```ts
showAppleSignIn: true,
```

to:

```ts
showAppleSignIn: Platform.OS === 'ios',
```

Import `Platform` from `react-native` if not already imported.

---

## Phase 1 — Apple Developer Portal setup

Only two things are needed for the native `signInWithIdToken` flow — no key/`.p8` file required.

### 1.1 — Note your Bundle ID
- Go to **Certificates, IDs & Profiles → Identifiers**.
- Find your app — note the exact **Bundle ID** (e.g. `com.salsabiltechnologies.ihsan`).
- If you have separate staging and production App IDs, note both bundle IDs.

### 1.2 — Enable "Sign In with Apple" on your App ID
- Select the App ID from the list.
- Scroll to **Sign In with Apple**, check it, click **Save**.
- Repeat for any staging App ID.

> **No key / `.p8` file is needed.** Supabase verifies Apple identity tokens using Apple's publicly available JWKS keys — no private key required on our side for the native flow.

---

## Phase 2 — Supabase Apple provider configuration

The Supabase UI for Apple has three relevant fields (as seen in the dashboard screenshot):

| Field | What to enter |
|-------|--------------|
| **Enable Sign in with Apple** | Toggle ON (already enabled per screenshot) |
| **Client IDs** | Your app's Bundle ID(s), comma-separated — e.g. `com.salsabiltechnologies.ihsan`. This is the `aud` claim Supabase validates in every Apple identity token. If staging uses a different bundle ID, include both. |
| **Secret Key (for OAuth)** | **Leave blank** — this is only needed for the web/OAuth browser flow. Our native `signInWithIdToken` flow does not use it. |
| **Allow users without an email** | Leave OFF — Apple users always have an email (real or relay). |

> **About the "secret keys expire every 6 months" warning** — this warning only applies if you fill in the Secret Key field for the web OAuth flow. Since we leave it blank, the warning does not affect us.

Click **Save**. Supabase can now verify Apple identity tokens and create/look up user accounts.

---

## Phase 3 — Install the native library

```bash
cd project-ihsan
yarn add @invertase/react-native-apple-authentication
cd ios && pod install
```

Then in **Xcode**:
- Open `project-ihsan/ios/ProjectIhsan.xcworkspace`.
- Select the **ProjectIhsan** target → **Signing & Capabilities**.
- Click **+ Capability** → add **Sign In with Apple**.
- Repeat for the staging target if one exists.

> No Android setup is needed — the library is iOS-only and will not affect the Android build.

---

## Phase 4 — Implement the sign-in flow

### 4.1 — `SocialAuthentication.ts`

Add a `signInWithApple()` method:

```ts
import appleAuth from '@invertase/react-native-apple-authentication';

async signInWithApple(): Promise<AuthResponse> {
  // 1. Request credential from Apple
  const appleAuthRequestResponse = await appleAuth.performRequest({
    requestedOperation: appleAuth.Operation.LOGIN,
    requestedScopes: [appleAuth.Scope.FULL_NAME, appleAuth.Scope.EMAIL],
  });

  if (!appleAuthRequestResponse.identityToken) {
    throw new Error('Apple Sign-In failed: no identity token returned.');
  }

  // 2. Exchange token with Supabase
  const { data, error } = await supabase.auth.signInWithIdToken({
    provider: 'apple',
    token: appleAuthRequestResponse.identityToken,
  });

  if (error) throw error;
  return { data, error: null };
}
```

Also update `signOutFromApple()` to call `supabase.auth.signOut()` (same as the existing sign-out path — Apple does not require a separate revocation call for basic sign-out).

### 4.2 — `LoginController.ts` — `loginWithApple()`

Replace the placeholder with a real call:

```ts
async loginWithApple(): Promise<AuthResponse> {
  return this.socialAuthService.signInWithApple();
}
```

Update the return type on the interface (`ILoginController.ts`) from `Promise<void>` to `Promise<AuthResponse>` to match `loginWithGoogle`.

### 4.3 — `AuthScreen/index.tsx` — `handleAppleSignIn`

Replace the placeholder timeout with:

```ts
const handleAppleSignIn = async () => {
  setAppleLoading(true);
  try {
    const response = await loginController.loginWithApple();
    handleAuthSuccess(response);
  } catch (error) {
    handleAuthError(error);
  } finally {
    setAppleLoading(false);
  }
};
```

Use the same `handleAuthSuccess` / `handleAuthError` helpers already used by Google sign-in.

---

## Phase 4b — User profile creation with profile picture handling

This mirrors the `signUpUsingGoogle` flow in `SignUpController.ts`.

### Platform limitation — Apple does not provide a profile picture

Unlike Google (which returns `avatar_url`/`picture` in `user_metadata`), Apple Sign-In **never** provides a profile picture URL. Apple only returns name + email, and only on the very first sign-in. All subsequent sign-ins return no user info at all — just the opaque user identifier. The `user.user_metadata.avatar_url` and `user.user_metadata.picture` fields will always be `undefined` for Apple users.

**Consequence:** `profilePictureUrl` is always `null` for Apple sign-ins. The rest of the flow is identical to Google.

### 4b.1 — Add `signUpUsingApple()` in `SignUpController.ts`

Modelled directly on the existing `signUpUsingGoogle()` method. Key differences:
- No profile picture download/upload step (nothing to download).
- Name is extracted from `appleAuthRequestResponse.fullName` (available only on first sign-in; store it before calling Supabase, as it won't be in `user_metadata` on subsequent logins).

```ts
async signUpUsingApple(
  user: User,
  appleFullName: AppleRequestResponse['fullName'],
): Promise<void> {
  const isNewUser = /* check user_profiles for existing record, same pattern as Google */;

  // Apple only provides fullName on the very first sign-in.
  // On subsequent logins, fall back to what is already saved in user_profiles.
  const firstName = appleFullName?.givenName ?? user.user_metadata?.full_name?.split(' ')[0] ?? '';
  const lastName  = appleFullName?.familyName ?? user.user_metadata?.full_name?.split(' ').slice(1).join(' ') ?? '';
  const displayName = [firstName, lastName].filter(Boolean).join(' ') || user.email ?? 'Apple User';
  const email = user.email ?? '';

  // No profile picture available from Apple — pass null
  const profilePictureStoragePath: string | null = null;

  if (isNewUser) {
    await this.supabaseEdgeFunctions.postUserProfile(
      user.id,
      displayName,
      email,
      profilePictureStoragePath,
    );
  } else {
    await this.supabaseEdgeFunctions.updateUserProfile({
      user_id: user.id,
      displayName,
      email,
      profilePictureUrl: undefined, // don't overwrite existing picture on re-login
    });
  }
}
```

> **Why not overwrite `profilePictureUrl` on re-login?** The user may have uploaded their own picture after the initial sign-in. Passing `undefined` (not `null`) leaves the existing `profile_picture_url` untouched in the backend.

### 4b.2 — Pass `fullName` from `SocialAuthentication.ts` through to `SignUpController`

Update `signInWithApple()` in `SocialAuthentication.ts` to return the raw `appleAuthRequestResponse.fullName` alongside the Supabase auth response, so `SignUpController` can use it:

```ts
async signInWithApple(): Promise<{ authResponse: AuthResponse; fullName: AppleRequestResponse['fullName'] }> {
  const appleAuthRequestResponse = await appleAuth.performRequest({ ... });
  const { data, error } = await supabase.auth.signInWithIdToken({ ... });
  if (error) throw error;
  return {
    authResponse: { data, error: null },
    fullName: appleAuthRequestResponse.fullName,
  };
}
```

Update `LoginController.loginWithApple()` and `ILoginController` return type accordingly.

### 4b.3 — Call `signUpUsingApple()` from `AuthScreen/index.tsx` `handleAppleSignIn`

After the auth response is returned, call `signUpController.signUpUsingApple(user, fullName)` before navigating, exactly mirroring how `handleGoogleSignIn` calls `signUpController.signUpUsingGoogle(user)`.

---

## Phase 5 — Testing

1. **iOS Simulator** — Apple Sign-In works on Simulator in Xcode 13+ (simulated Apple ID). Use `yarn ios:staging` and tap the button.
2. **Real device** — Sign in with a real Apple ID; verify a new user appears in Supabase → Authentication → Users.
3. **Repeat sign-in** — Sign in a second time with the same Apple ID; confirm the same Supabase user ID is returned (no duplicate).
4. **Android build check** — Run `yarn android:staging` and confirm the Apple button is absent.

---

## Files changed summary

| File | Change |
|------|--------|
| `src/views/screens/AuthScreen/index.tsx` | `Platform.OS === 'ios'` guard + real `handleAppleSignIn` calling both login and signUp controllers |
| `src/services/SocialAuthentication.ts` | Add `signInWithApple()` returning auth response + `fullName`; update `signOutFromApple()` |
| `src/controllers/implementations/LoginController.ts` | Real `loginWithApple()` returning auth response + `fullName` |
| `src/controllers/interfaces/ILoginController.ts` | Fix return type to include `fullName` |
| `src/controllers/implementations/SignUpController.ts` | Add `signUpUsingApple(user, fullName)` — mirrors `signUpUsingGoogle` but skips profile picture |
| `ios/ProjectIhsan.xcworkspace` (Xcode) | Add "Sign In with Apple" capability |
| `package.json` / `ios/Podfile.lock` | New dependency `@invertase/react-native-apple-authentication` |

---

## Information you will need to provide (during implementation)

- Exact Bundle ID(s) for staging and production targets (from Apple Developer portal → Identifiers)
- No Team ID, Key ID, or `.p8` private key needed for the native `signInWithIdToken` flow
