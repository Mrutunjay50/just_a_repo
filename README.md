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

## CODE CHANGES REQUIRED

### File Changes Summary

#### 1. `FirebaseManager.cs` → `AWSManager.cs`
**Change:** Replace Firebase initialization with AWS SDK setup
```csharp
// REMOVE: FirebaseApp.CheckAndFixDependenciesAsync()
// ADD: AmazonDynamoDBClient, CognitoAWSCredentials initialization
```

#### 2. `AuthManager.cs`
**Change:** Replace Firebase RTDB operations with DynamoDB + Real-time listener
```csharp
// REMOVE: dbRef.Child("users").Child(userId).GetValueAsync()
// ADD: dynamoClient.GetItemAsync(request)
// REMOVE: dbRef.SetRawJsonValueAsync(json)
// ADD: dynamoClient.PutItemAsync(putRequest)
// REMOVE: userRef.ValueChanged += HandleSessionChanged
// ADD: StartCoroutine(PollSessionToken(userId)) // or AppSync subscription
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

## HANDLING REAL-TIME FEATURES

Firebase RTDB's real-time listeners (`ValueChanged`, `ChildAdded`, etc.) need to be replaced. You have 3 options:

### Option 1: AWS AppSync (GraphQL Subscriptions)
Best for real-time requirements similar to Firebase.

**Setup:**
1. Create AppSync API with GraphQL schema
2. Connect to DynamoDB as data source
3. Use subscriptions for real-time updates

**Example - Session Token Monitoring:**
```graphql
# GraphQL Schema
type User {
  userId: ID!
  sessionToken: String!
}

type Mutation {
  updateSessionToken(userId: ID!, token: String!): User
}

type Subscription {
  onSessionTokenChanged(userId: ID!): User
    @aws_subscribe(mutations: ["updateSessionToken"])
}
```

**Unity Client:**
```csharp
using AWS.AppSync;

public class SessionMonitor : MonoBehaviour
{
    private AppSyncClient appSyncClient;
    
    public async void StartMonitoring(string userId)
    {
        string subscriptionQuery = $@"
            subscription {{
                onSessionTokenChanged(userId: ""{userId}"") {{
                    sessionToken
                }}
            }}
        ";
        
        await appSyncClient.Subscribe<SessionData>(subscriptionQuery, (data) => {
            string serverToken = data.sessionToken;
            string localToken = PlayerPrefs.GetString("sessionToken", "");
            
            if (serverToken != localToken && !string.IsNullOrEmpty(serverToken))
            {
                Debug.LogWarning("Session changed on another device!");
                ForceLogout();
            }
        });
    }
}
```

**Pros:**
- True real-time updates (similar to Firebase)
- Automatic reconnection handling
- Scales automatically

**Cons:**
- Additional service to configure
- Learning curve for GraphQL

---

### Option 2: DynamoDB Streams + Lambda + WebSocket API
For custom real-time requirements.

**Setup:**
1. Enable DynamoDB Streams on tables
2. Create Lambda function to process stream events
3. Use API Gateway WebSocket to push updates to clients

**Flow:**
```
DynamoDB Update → Stream Event → Lambda → WebSocket → Unity Client
```

**Unity Client:**
```csharp
using WebSocketSharp;

public class RealtimeManager : MonoBehaviour
{
    private WebSocket ws;
    
    public void ConnectWebSocket(string userId)
    {
        ws = new WebSocket("wss://your-api-id.execute-api.us-east-1.amazonaws.com/production");
        
        ws.OnMessage += (sender, e) => {
            var update = JsonUtility.FromJson<SessionUpdate>(e.Data);
            HandleSessionUpdate(update);
        };
        
        ws.Connect();
        
        // Subscribe to user-specific updates
        ws.Send(JsonUtility.ToJson(new { 
            action = "subscribe", 
            userId = userId 
        }));
    }
}
```

**Pros:**
- Full control over real-time logic
- Can filter events server-side
- Cost-effective for high volume

**Cons:**
- More complex setup
- Requires WebSocket management

---

### Option 3: Polling (Simple Alternative)
For less critical real-time features or during initial migration.

**Implementation:**
```csharp
public class PollingMonitor : MonoBehaviour
{
    private string userId;
    private bool isPolling = false;
    
    public void StartPolling(string uid)
    {
        userId = uid;
        isPolling = true;
        StartCoroutine(PollSessionToken());
    }
    
    IEnumerator PollSessionToken()
    {
        while (isPolling)
        {
            // Wait 30 seconds between checks
            yield return new WaitForSeconds(30f);
            
            try
            {
                // Check session token
                var user = await awsDbManager.GetUser(userId);
                string serverToken = user.sessionToken;
                string localToken = PlayerPrefs.GetString("sessionToken", "");
                
                if (serverToken != localToken && !string.IsNullOrEmpty(serverToken))
                {
                    Debug.LogWarning("Session changed - logging out!");
                    ForceLogout();
                    isPolling = false;
                }
            }
            catch (Exception ex)
            {
                Debug.LogError($"Polling error: {ex.Message}");
            }
        }
    }
    
    public void StopPolling()
    {
        isPolling = false;
        StopAllCoroutines();
    }
}
```

**Pros:**
- Simplest to implement
- No additional services needed
- Good for migration testing

**Cons:**
- Not truly real-time (delay up to polling interval)
- More DynamoDB read requests
- Battery drain on mobile

---

### Real-time Features in Your Code

**Current Firebase Listeners:**
```csharp
// AuthManager.cs - Line 381
userRef.ValueChanged += HandleSessionChanged;
```

**Migration:**
```csharp
// Replace with polling initially
StartCoroutine(PollSessionToken(userId));

// Later upgrade to AppSync if needed
// await SubscribeToSessionChanges(userId);
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

### 4. AWS AppSync (Optional - for Real-time)
**Purpose:** Real-time data synchronization (alternative to polling)  
**Setup:**
- Create GraphQL API
- Connect to DynamoDB
- Define subscriptions for real-time updates

**When to use:**
- If you need instant real-time updates
- For features like live leaderboards, chat, notifications
- Can be added later if polling is sufficient initially

---

## MIGRATION PHASES SUMMARY

```
Week 1: Infrastructure & Authentication
├─ Days 1-2: AWS infrastructure setup
└─ Days 3-5: Migrate all 5 login methods

Week 2: Database Migration
├─ Days 1-3: Implement DynamoDB operations
└─ Days 4-5: Data implementation & testing

Week 3: Real-time Logic & Testing
├─ Days 1-2: Implement/refine real-time solution (polling or AppSync)
│  └─ Replace Firebase listeners, test real-time features
└─ Days 3-5: Final testing
   └─ Edge case testing & bug fixes
```
---
