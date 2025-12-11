# Authentication Setup Guide

## Overview
The app now includes comprehensive authentication with:
- ✅ Email/Password Authentication
- ✅ Google Sign In
- ✅ Apple Sign In (iOS/Android only)
- ✅ Phone Verification (Cellular only, No VoIP)

## Google OAuth Setup

### Step 1: Create Google Cloud Project
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select an existing one
3. Enable the Google+ API

### Step 2: Create OAuth Credentials
1. Go to **APIs & Services** → **Credentials**
2. Click **Create Credentials** → **OAuth client ID**
3. Configure the OAuth consent screen if prompted
4. Create credentials for each platform:

#### Web Application
- Application type: **Web application**
- Authorized redirect URIs: Add your web URL

#### iOS Application  
- Application type: **iOS**
- Bundle ID: Your iOS bundle identifier (from app.json)

#### Android Application
- Application type: **Android**
- Package name: Your Android package name (from app.json)
- SHA-1 certificate: Run `keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android`

### Step 3: Update Client IDs
In both `app/login.tsx` and `app/signup.tsx`, replace:
```typescript
const [googleRequest, googleResponse, googlePromptAsync] = Google.useAuthRequest({
  clientId: 'YOUR_WEB_CLIENT_ID',  // Replace with Web client ID
  iosClientId: 'YOUR_IOS_CLIENT_ID',  // Replace with iOS client ID
  androidClientId: 'YOUR_ANDROID_CLIENT_ID',  // Replace with Android client ID
});
```

## Apple Sign In Setup

### Requirements
- Apple Developer Account ($99/year)
- iOS/Android devices (Not available on web)

### Step 1: Configure App Identifier
1. Go to [Apple Developer Portal](https://developer.apple.com/)
2. Certificates, Identifiers & Profiles → **Identifiers**
3. Select your App ID
4. Enable **Sign in with Apple** capability
5. Save changes

### Step 2: Update app.json
Add to your `app.json`:
```json
{
  "expo": {
    "ios": {
      "usesAppleSignIn": true
    }
  }
}
```

## Phone Verification Setup

### Current Implementation
The phone verification is set up with backend routes but needs a real SMS service.

### Integration Options

#### Option 1: Twilio (Recommended)
1. Sign up at [Twilio](https://www.twilio.com/)
2. Get Account SID and Auth Token
3. Install Twilio SDK in backend:
   ```bash
   bun add twilio
   ```
4. Update `backend/trpc/routes/auth/verify-phone/route.ts`:
   ```typescript
   import twilio from 'twilio';
   
   const client = twilio(
     process.env.TWILIO_ACCOUNT_SID,
     process.env.TWILIO_AUTH_TOKEN
   );
   
   // In the mutation:
   await client.messages.create({
     body: `Your verification code is: ${generatedCode}`,
     from: process.env.TWILIO_PHONE_NUMBER,
     to: input.phoneNumber
   });
   ```

#### Option 2: AWS SNS
1. Configure AWS SNS in your AWS Console
2. Install AWS SDK:
   ```bash
   bun add @aws-sdk/client-sns
   ```
3. Update the verify-phone route with SNS implementation

#### Option 3: Firebase Phone Auth
1. Set up Firebase project
2. Enable Phone Authentication
3. Install Firebase Admin SDK:
   ```bash
   bun add firebase-admin
   ```

### VoIP Detection
To block VoIP numbers, integrate services like:
- [Twilio Lookup API](https://www.twilio.com/docs/lookup/api) (carrier info)
- [Numverify](https://numverify.com/) (number validation)
- [ipqualityscore](https://www.ipqualityscore.com/) (fraud detection)

Example with Twilio Lookup:
```typescript
const lookup = await client.lookups.v2
  .phoneNumbers(phoneNumber)
  .fetch({ fields: 'line_type_intelligence' });

if (lookup.lineTypeIntelligence?.type === 'voip') {
  throw new Error('VoIP numbers are not allowed');
}
```

## Backend Implementation

### User Database Schema
You'll need to store:
```typescript
type User = {
  id: string;
  email: string;
  name: string;
  passwordHash?: string; // Only for email auth
  provider: 'email' | 'google' | 'apple';
  providerId?: string;
  phoneNumber?: string;
  phoneVerified: boolean;
  createdAt: Date;
}
```

### Update Backend Routes
The current routes (`backend/trpc/routes/auth/*`) are placeholders. You need to:

1. **Add Database** (PostgreSQL, MongoDB, etc.)
2. **Hash Passwords** using bcrypt:
   ```bash
   bun add bcrypt @types/bcrypt
   ```
3. **Generate JWT tokens** for session management:
   ```bash
   bun add jsonwebtoken @types/jsonwebtoken
   ```
4. **Store verification codes** temporarily (Redis recommended)

### Example Enhanced Signup Route:
```typescript
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

export const signupProcedure = publicProcedure
  .input(...)
  .mutation(async ({ input }) => {
    // Check if user exists
    const existingUser = await db.user.findUnique({
      where: { email: input.email }
    });
    
    if (existingUser) {
      throw new Error('User already exists');
    }
    
    // Hash password for email auth
    const passwordHash = input.provider === 'email'
      ? await bcrypt.hash(input.password, 10)
      : undefined;
    
    // Create user
    const user = await db.user.create({
      data: {
        email: input.email,
        name: input.name,
        passwordHash,
        provider: input.provider,
        providerId: input.providerId,
        phoneVerified: false,
      }
    });
    
    return {
      success: true,
      userId: user.id,
      needsPhoneVerification: true,
    };
  });
```

## Testing

### Local Testing
1. For Google: Use test accounts in Google Cloud Console
2. For Apple: Use Sandbox environment
3. For Phone: Use Twilio test credentials (won't send real SMS)

### Important Notes
- Apple Sign In requires production build or TestFlight
- Google Sign In works in Expo Go with proper configuration
- Phone verification requires real SMS service in production

## Environment Variables

Create a `.env` file:
```env
# Google OAuth
GOOGLE_CLIENT_ID=123882586844-7oqn4ggt59emqgignsvbqr84ruajtf0m.apps.googleusercontent.com
GOOGLE_IOS_CLIENT_ID=123882586844-efm7085v0gvuo2j2akfefmct250lihu2.apps.googleusercontent.com
GOOGLE_ANDROID_CLIENT_ID=your_android_client_id

# Twilio (or other SMS service)
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE_NUMBER=your_twilio_number

# JWT Secret
JWT_SECRET=your_secret_key_here

# Database
DATABASE_URL=your_database_connection_string
```

## Next Steps

1. **Set up Google OAuth credentials** and update client IDs
2. **Configure Apple Sign In** (if targeting iOS)
3. **Choose and integrate SMS service** (Twilio recommended)
4. **Add database** for user storage
5. **Implement JWT authentication** for session management
6. **Add VoIP detection** to verify-phone route
7. **Test authentication flow** end-to-end

## Support

For issues or questions about authentication:
- Google: [Expo AuthSession docs](https://docs.expo.dev/versions/latest/sdk/auth-session/)
- Apple: [Expo Apple Authentication docs](https://docs.expo.dev/versions/latest/sdk/apple-authentication/)
- Phone: Your SMS provider documentation
