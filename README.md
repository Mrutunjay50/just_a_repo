# FIREBASE TO AWS MIGRATION - EXECUTIVE OVERVIEW
## Cricket Or Nothing Game

**Date:** November 5, 2025  
**Migration Timeline:** 2-3 weeks

---

## QUICK SUMMARY

### What We're Migrating

| Current (Firebase) | Moving To (AWS) |
|-------------------|----------------|
| Firebase Authentication | Amazon Cognito |
| Firebase Realtime Database | Amazon DynamoDB |
| Firebase CloudStorage | Amazon S3 |
---

## MIGRATION SCOPE

### Authentication (5 Methods)
- ✓ Google Sign-In → Cognito with Google IdP
- ✓ Facebook Login → Cognito with Facebook IdP
- ✓ Apple Sign-In → Cognito with Apple IdP
- ✓ WhatsApp/Phone OTP → Cognito SMS verification
- ✓ Guest Login → Cognito unauthenticated identities

### Database (2 Tables)
- ✓ User Profiles → DynamoDB Users table
- ✓ Coin Transactions → DynamoDB CoinTransactions table

### Files to Update (9 Critical)
1. `FirebaseManager.cs` - Initialize AWS SDK
2. `AuthManager.cs` - Cognito + DynamoDB operations
3. `GoogleLogin.cs` - Cognito Google auth
4. `FaceBookManager.cs` - Cognito Facebook auth
5. `AppleLogin.cs` - Cognito Apple auth
6. `WhatsAppLogin.cs` - Cognito SMS/Phone auth
7. `GuestLoginManager.cs` - Cognito anonymous auth
8. `EditProfileManager.cs` - DynamoDB CRUD
9. `CoinManager.cs` - DynamoDB atomic updates

---

## 2-3 WEEK TIMELINE

### Week 1: Infrastructure & Authentication
**Days 1-2: Infrastructure Setup**
- Create AWS account and IAM roles
- Set up Cognito User Pool + Identity Pool
- Create DynamoDB tables
- Configure social login providers
- Install AWS SDK in Unity

**Days 3-5: Authentication Migration**
- Implement Cognito authentication
- Update all 5 login methods (Google, Facebook, Apple, WhatsApp, Guest)
- Test authentication flows

**Deliverable:** AWS infrastructure ready + All logins working

### Week 2: Database Migration
**Days 1-3: Database Implementation**
- Implement DynamoDB operations
- Update user profile CRUD operations
- Update coin management system
- Implement real-time session monitoring

**Days 4-5: Data Migration & Testing**
- Migrate existing Firebase data to DynamoDB
- Enable dual-write (Firebase + DynamoDB)
- Comprehensive testing
- Validate data integrity

**Deliverable:** Database migrated and tested

### Week 3: Testing & Production Rollout
**Days 1-2: Final Testing**
- Load testing (100+ concurrent users)
- Edge case testing
- Bug fixes
- Deploy to UAT environment

**Days 3-5: Production Rollout**
- Deploy to production
- Gradual rollout: 10% → 50% → 100%
- Monitor metrics and errors
- Disable Firebase completely

**Deliverable:** Migration complete ✅

---

## CODE CHANGES REQUIRED

### File Changes Summary

#### 1. `FirebaseManager.cs` → `AWSManager.cs`
**Change:** Replace Firebase initialization with AWS SDK setup
```csharp
// REMOVE: FirebaseApp.CheckAndFixDependenciesAsync()
// ADD: AmazonDynamoDBClient, CognitoAWSCredentials initialization
```

#### 2. `AuthManager.cs`
**Change:** Replace Firebase RTDB operations with DynamoDB
```csharp
// REMOVE: dbRef.Child("users").Child(userId).GetValueAsync()
// ADD: dynamoClient.GetItemAsync(request)
// REMOVE: dbRef.SetRawJsonValueAsync(json)
// ADD: dynamoClient.PutItemAsync(putRequest)
```

#### 3. `GoogleLogin.cs`, `FaceBookManager.cs`, `AppleLogin.cs`
**Change:** Replace Firebase Auth with Cognito identity providers
```csharp
// REMOVE: FirebaseAuth.SignInWithCredentialAsync(credential)
// ADD: CognitoAWSCredentials.AddLogin("provider", token)
```

#### 4. `WhatsAppLogin.cs`
**Change:** Replace Firebase Phone Auth with Cognito SMS
```csharp
// REMOVE: FirebaseAuth.PhoneAuthProvider.VerifyPhoneNumber()
// ADD: Cognito SignUpRequest + ConfirmSignUpRequest
```

#### 5. `GuestLoginManager.cs`
**Change:** Replace anonymous auth with Cognito unauthenticated
```csharp
// REMOVE: FirebaseAuth.SignInAnonymouslyAsync()
// ADD: CognitoAWSCredentials.GetIdentityIdAsync() (unauthenticated)
```

#### 6. `EditProfileManager.cs`
**Change:** Replace Firebase RTDB with DynamoDB operations
```csharp
// REMOVE: dbRef.Child("users").GetValueAsync()
// ADD: dynamoClient.GetItemAsync() / PutItemAsync()
// REMOVE: dbRef.Query for team name check
// ADD: dynamoClient.QueryAsync() with GSI
```

#### 7. `CoinManager.cs`
**Change:** Replace Firebase transactions with DynamoDB atomic updates
```csharp
// REMOVE: dbRef.RunTransaction()
// ADD: dynamoClient.UpdateItemAsync() with UpdateExpression
// REMOVE: dbRef.Child("coinHistory").Push()
// ADD: Generate transactionId manually (timestamp#guid)
```

### Before/After Examples

**Authentication (Guest Login):**
```csharp
// BEFORE: Firebase
FirebaseAuth.DefaultInstance.SignInAnonymouslyAsync()
    .ContinueWith(task => {
        string userId = task.Result.User.UserId;
    });

// AFTER: AWS Cognito
var credentials = new CognitoAWSCredentials(identityPoolId, RegionEndpoint.USEast1);
string userId = await credentials.GetIdentityIdAsync();
```

**Database Read (User Profile):**
```csharp
// BEFORE: Firebase RTDB
dbRef.Child("users").Child(userId).GetValueAsync()
    .ContinueWith(task => {
        var userData = JsonUtility.FromJson<UserProfile>(task.Result.GetRawJsonValue());
    });

// AFTER: AWS DynamoDB
var request = new GetItemRequest {
    TableName = "CricketGame_Users",
    Key = new Dictionary<string, AttributeValue> {
        {"userId", new AttributeValue {S = userId}}
    }
};
var response = await dynamoClient.GetItemAsync(request);
var userData = MapToUserProfile(response.Item);
```

**Database Write (Save User):**
```csharp
// BEFORE: Firebase RTDB
string json = JsonUtility.ToJson(userData);
dbRef.Child("users").Child(userId).SetRawJsonValueAsync(json);

// AFTER: AWS DynamoDB
var request = new PutItemRequest {
    TableName = "CricketGame_Users",
    Item = ConvertToDynamoDBItem(userData)
};
await dynamoClient.PutItemAsync(request);
```

**Atomic Coin Update:**
```csharp
// BEFORE: Firebase Transaction
dbRef.Child("coins").RunTransaction(mutableData => {
    mutableData.Value = (int)mutableData.Value + amount;
    return TransactionResult.Success(mutableData);
});

// AFTER: DynamoDB Atomic Update
var request = new UpdateItemRequest {
    TableName = "CricketGame_Users",
    Key = new Dictionary<string, AttributeValue> {
        {"userId", new AttributeValue {S = userId}}
    },
    UpdateExpression = "ADD coins :amount",
    ExpressionAttributeValues = new Dictionary<string, AttributeValue> {
        {":amount", new AttributeValue {N = amount.ToString()}}
    }
};
await dynamoClient.UpdateItemAsync(request);
```

**Real-time Listener (Session Token):**
```csharp
// BEFORE: Firebase ValueChanged
userRef.ValueChanged += HandleSessionChanged;

// AFTER: AWS AppSync Subscription (or polling)
await appSyncClient.Subscribe<User>(subscriptionQuery, (data) => {
    HandleSessionChanged(data.sessionToken);
});
```

---

## AWS SERVICES REQUIRED

### 1. Amazon Cognito
**Purpose:** User authentication  
**Setup:**
- User Pool: cricket-game-users
- Identity Pool: cricket-game-identity-pool
- Social providers: Google, Facebook, Apple
- Phone verification: Enabled

### 2. Amazon DynamoDB
**Purpose:** NoSQL database/ or Mongoose separately
**Tables:**
- `CricketGame_Users` - User profiles
- `CricketGame_CoinTransactions` - Transaction history
- `CricketGame_UAT_Users` - UAT environment
- `CricketGame_UAT_CoinTransactions` - UAT transactions

### 3. Amazon S3 (Future Use)
**Purpose:** File storage for profile pictures  
**Setup:**
- Bucket: cricket-game-assets
- CloudFront: CDN for faster delivery

---

## MIGRATION PHASES SUMMARY

```
Week 1: Infrastructure & Authentication
├─ Days 1-2: AWS infrastructure setup
└─ Days 3-5: Migrate all 5 login methods

Week 2: Database Migration
├─ Days 1-3: Implement DynamoDB operations
└─ Days 4-5: Data implementation & testing

Week 3: Testing
├─ Days 1-2: Final testing
```
---
