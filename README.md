# Vercel OAuth Mobile Integration

A production-ready OAuth implementation for React Native apps integrating with Vercel's API, featuring secure token management and seamless mobile authentication.

## Overview

This system enables mobile apps to authenticate with Vercel using OAuth 2.0, bridging the gap between web-based OAuth redirects and native mobile deep linking. It provides:

- ✅ Seamless browser-to-app handoff
- ✅ Secure token persistence using platform Keychain/Keystore
- ✅ CSRF protection via state validation
- ✅ Server-side token exchange (no client secrets exposed)
- ✅ Support for both iOS and Android

## Architecture

### Authentication Flow

```
App → API → Vercel → Web Callback → Deep Link → App
```

1. **App initiates OAuth**: Opens Vercel authorization in system browser
2. **User authorizes**: Grants permissions in Vercel's interface
3. **Vercel redirects**: Sends user to `https://auth.vctrl.io`
4. **Web callback processes**: Validates OAuth response
5. **Deep link triggers**: Redirects to `vctrl://auth/callback?...`
6. **App intercepts**: Captures OAuth parameters via deep linking
7. **Token exchange**: Backend exchanges code for access token
8. **Secure storage**: Tokens saved to SecureStore (Keychain/Keystore)

## The OAuth Callback Page

### What It Does

The HTML callback page (`auth.vctrl.io`) serves as a bridge between Vercel's HTTPS-only redirect requirement and your mobile app's custom URL scheme:

```html
<!-- Hosted at https://auth.vctrl.io -->
```

**Key responsibilities:**
- Receives OAuth callback from Vercel
- Extracts `code`, `state`, and `installation_id` parameters
- Immediately redirects to app via custom scheme
- Provides fallback UI for debugging and manual app switching

### Parameter Handling

The page handles multiple parameter variations for compatibility:

```javascript
const installationId = urlParams.get('installation_id') || 
                      urlParams.get('configurationId') || 
                      urlParams.get('config_id');
```

### Redirect Logic

```javascript
const appUrl = `vctrl://auth/callback?code=${code}&state=${state}&installation_id=${installationId}`;
window.location.href = appUrl;  // Opens mobile app
```

## Mobile App Implementation

### 1. OAuth Initiation (`authService.ts`)

```typescript
async initiateOAuth() {
  // Generate and store CSRF state
  const state = generateRandomState();
  await SecureStore.setItemAsync('oauth_state', state);
  
  // Open Vercel OAuth in browser
  const authUrl = `https://vercel.com/oauth/authorize?...&redirect_uri=https://auth.vctrl.io`;
  const result = await WebBrowser.openAuthSessionAsync(
    authUrl,
    'vctrl://auth/callback'
  );
  
  if (result.type === 'success') {
    await handleOAuthCallback(result.url);
  }
}
```

### 2. Callback Handling

```typescript
async handleOAuthCallback(url: string) {
  // Parse deep link
  const { code, state, installation_id } = parseUrl(url);
  
  // Validate state (CSRF protection)
  const storedState = await SecureStore.getItemAsync('oauth_state');
  if (state !== storedState) {
    throw new Error('Invalid state - potential CSRF attack');
  }
  
  // Exchange code for token (server-side)
  const { access_token, user_data } = await api.exchangeToken({
    code,
    installation_id
  });
  
  // Store securely
  await TokenManager.storeTokens(access_token, user_data);
}
```

### 3. Token Management (`tokenManager.ts`)

```typescript
class TokenManager {
  async storeTokens(accessToken: string, userData: any) {
    await SecureStore.setItemAsync('vercel_access_token', accessToken);
    await SecureStore.setItemAsync('vercel_user_data', JSON.stringify(userData));
  }
  
  async getAccessToken(): Promise<string | null> {
    return await SecureStore.getItemAsync('vercel_access_token');
  }
  
  async isAuthenticated(): Promise<boolean> {
    const token = await this.getAccessToken();
    return token !== null;
  }
  
  async clearTokens() {
    await SecureStore.deleteItemAsync('vercel_access_token');
    await SecureStore.deleteItemAsync('vercel_user_data');
  }
}
```

## Configuration

### iOS Setup (`app.json`)

```json
{
  "expo": {
    "scheme": "vctrl",
    "ios": {
      "bundleIdentifier": "com.yourcompany.vctrl",
      "infoPlist": {
        "CFBundleURLTypes": [
          {
            "CFBundleURLSchemes": ["vctrl"]
          }
        ]
      }
    }
  }
}
```

### Android Setup (`app.json`)

```json
{
  "expo": {
    "android": {
      "package": "com.yourcompany.vctrl",
      "intentFilters": [
        {
          "action": "VIEW",
          "data": [
            {
              "scheme": "vctrl",
              "host": "auth",
              "path": "/callback"
            }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

### Vercel OAuth Application

Configure your Vercel OAuth app with:
- **Redirect URI**: `https://auth.vctrl.io`
- **Client ID**: From Vercel integration settings
- **Client Secret**: Stored server-side only

## Security Features

### 1. CSRF Protection
- Random `state` parameter generated per session
- Validated on callback to prevent replay attacks
- Stored in SecureStore, never in URL or cookies

### 2. Secure Token Storage
- **iOS**: Tokens stored in Keychain with encryption
- **Android**: Tokens stored in Keystore with hardware backing
- **Never** stored in AsyncStorage or plain text

### 3. Server-Side Exchange
- Client secrets never exposed to mobile app
- Token exchange happens on backend
- Mobile app receives only the access token result

### 4. Installation ID Validation
Handles multiple parameter formats from Vercel:
```javascript
installation_id || configurationId || config_id
```

## Common Issues & Solutions

### Problem 1: App Doesn't Open After Authorization

**Symptoms:**
- Browser shows "Success" but app doesn't reopen
- Stuck on callback page

**Solutions:**
1. Verify custom URL scheme matches in all configs
2. Test deep link: `xcrun simctl openurl booted vctrl://auth/callback`
3. Check iOS Associated Domains or Android intent filters
4. Ensure `openAuthSessionAsync` uses correct redirect URL

### Problem 2: Tokens Lost After App Restart

**Symptoms:**
- User must re-authenticate every launch
- `isAuthenticated()` returns false unexpectedly

**Root Causes:**
1. Using AsyncStorage instead of SecureStore
2. Bundle ID change (resets Keychain access group)
3. App uninstall/reinstall on simulator
4. Keychain cleared during development

**Solution:**
```typescript
// Always use SecureStore
import * as SecureStore from 'expo-secure-store';

// Not AsyncStorage
import AsyncStorage from '@react-native-async-storage/async-storage'; // ❌
```

### Problem 3: State Validation Fails

**Symptoms:**
- "Invalid state" error on callback
- Authentication never completes

**Causes:**
- State not persisted before browser opens
- Multiple simultaneous auth attempts
- Race condition in state storage

**Solution:**
```typescript
// Store state BEFORE opening browser
await SecureStore.setItemAsync('oauth_state', state);
await new Promise(resolve => setTimeout(resolve, 100)); // Ensure write completes
const result = await WebBrowser.openAuthSessionAsync(...);
```

### Problem 4: Missing Installation ID

**Symptoms:**
- Callback succeeds but API calls fail
- Integration not properly linked

**Debug:**
```javascript
// Check all parameter variations
console.log('All URL params:', Object.fromEntries(urlParams.entries()));
```

**Solution:**
The callback page handles this with fallbacks, but ensure Vercel sends the parameter by checking your integration setup.

## Development vs Production

### Development
- Use `https://auth.vctrl.io` for testing
- Tokens stored in simulator Keychain (wiped on reset)
- Debug logging enabled in callback page

### Production
- Same callback URL (no changes needed)
- Tokens persist across updates and reboots
- Remove debug UI from callback page
- Add token refresh logic
- Implement proper session expiry handling

## Testing

### Test the Full Flow

1. **Initiate OAuth:**
```bash
# iOS Simulator
xcrun simctl openurl booted "vctrl://test"
```

2. **Test Deep Link:**
```bash
# With parameters
xcrun simctl openurl booted "vctrl://auth/callback?code=test&state=test&installation_id=test"
```

3. **Verify Token Storage:**
```typescript
const token = await SecureStore.getItemAsync('vercel_access_token');
console.log('Token stored:', token ? 'Yes' : 'No');
```

### Debug Mode

The callback page includes comprehensive debugging:
```javascript
const debugInfo = {
  code: code ? code.substring(0, 10) + '...' : 'missing',
  state: state ? state.substring(0, 10) + '...' : 'missing',
  installation_id: installationId || 'missing',
  allParams: Object.fromEntries(urlParams.entries())
};
```

## Why This Approach?

### The Problem
- Vercel requires HTTPS redirect URIs
- Mobile apps can't directly receive HTTPS redirects
- OAuth codes must be exchanged securely
- Tokens must persist across sessions

### The Solution
1. **Web bridge**: Hosted callback receives HTTPS redirect
2. **Deep linking**: Immediately hands off to app
3. **Server exchange**: Backend handles token exchange
4. **Secure storage**: Platform Keychain/Keystore persistence

### User Impact

**Without this system:**
- ❌ Users stuck on login screen
- ❌ Re-authentication after every restart
- ❌ Insecure token storage
- ❌ Unusable app experience

**With this system:**
- ✅ Seamless one-time login
- ✅ Persistent sessions
- ✅ Enterprise-grade security
- ✅ Professional UX

## API Reference

### Backend Endpoint

```typescript
POST /api/exchange-token
Content-Type: application/json

{
  "code": "oauth_code_from_vercel",
  "installation_id": "installation_12345"
}

Response:
{
  "access_token": "token_xyz",
  "user_data": {
    "user_id": "user_123",
    "email": "user@example.com",
    "team_id": "team_456"
  }
}
```

## Future Enhancements

- [ ] Token refresh logic
- [ ] Redis/database session storage
- [ ] Multi-account support
- [ ] Biometric re-authentication
- [ ] Offline mode handling
- [ ] Session expiry warnings


## Support

For issues or questions:
- Check troubleshooting section above
- Review Vercel OAuth documentation
- Test with debug mode enabled
- Verify all configuration matches exactly
