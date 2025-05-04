# social_login_drf_and_react
login with socila link manually django rest framework and react app



Here's a **step-by-step documentation** for implementing **Google and Microsoft OAuth Login** in your Django + React project.

---

## 🔐 OAuth Login Integration Guide

### ✅ Prerequisites

- A **Django backend** with JWT authentication configured.
- A **React frontend** for redirecting users and capturing tokens.
- Registered OAuth applications for **Google** and **Microsoft**.

---

## 🌐 Google OAuth Login

### 🔧 1. **Register Google OAuth App**
Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials):
- Create OAuth 2.0 Client ID.
- Set **Authorized redirect URI** to:
  ```
  https://yourdomain/api/accounts/auth/google/callback/
  ```
- Copy:
  - `Client ID`
  - `Client Secret`

### ⚙️ 2. **Backend Settings**

In Django settings or a safe environment file:

```python
GOOGLE_CLIENT_ID = config('GOOGLE_CLIENT_ID')
GOOGLE_CLIENT_SECRET = config('GOOGLE_CLIENT_SECRET')
REDIRECT_URI = 'https://yourdomain/api/accounts/auth/google/callback/'
```

### 🧠 3. **Google Callback View**

Create `GoogleAuthCallbackView` in Django:

```python
class GoogleAuthCallbackView(APIView):
    permission_classes = [AllowAny]
    
    def get(self, request):
        code = request.GET.get('code')
        if not code:
            return Response({'error': 'Missing code'}, status=400)

        # Step 1: Exchange code for access token
        token_url = 'https://oauth2.googleapis.com/token'
        token_data = {
            'code': code,
            'client_id': GOOGLE_CLIENT_ID,
            'client_secret': GOOGLE_CLIENT_SECRET,
            'redirect_uri': REDIRECT_URI,
            'grant_type': 'authorization_code'
        }
        token_res = requests.post(token_url, data=token_data)
        if token_res.status_code != 200:
            return Response({'error': 'Failed to get token'}, status=400)
        token_json = token_res.json()
        id_token = token_json.get('id_token')

        # Step 2: Decode and verify ID token with Google
        id_info_url = f'https://oauth2.googleapis.com/tokeninfo?id_token={id_token}'
        userinfo_res = requests.get(id_info_url)
        if userinfo_res.status_code != 200:
            return Response({'error': 'Failed to verify ID token'}, status=400)
        user_info = userinfo_res.json()

        email = user_info.get('email')
        name = user_info.get('name')
        picture = user_info.get('picture')
        if not email:
            return Response({'error': 'Invalid Google response'}, status=400)

        # Step 3: Get or create user

        # Step 3: Get or create user - Modified to handle existing email
        try:
            user = User.objects.get(email=email)
            created = False
        except User.DoesNotExist:
            # Create user with a unique username if email exists
            username = email  # or generate a unique username
            user = User.objects.create_user(
                username=username,
                email=email,
                first_name=name
            )
            created = True

        # Get or create profile
        profile, profile_created = Profile.objects.get_or_create(user=user)
        
        if picture and profile:
            from django.core.files.base import ContentFile
            try:
                img_response = requests.get(picture)
                if img_response.status_code == 200:
                    profile.profile_image.save(
                        f'{user.username}_profile.jpg',
                        ContentFile(img_response.content),
                        save=True
                    )
            except Exception as e:
                print("Image download failed:", e)
        
        
        if profile:
            if profile.profile_image:
                image_url = profile.profile_image.url
            else:
                image_url = ''
        else:
            image_url = ''
        
        # Step 4: Issue JWT tokens
        refresh = RefreshToken.for_user(user)
        
        react_redirect_url = f"https://localhost:3000/?access={refresh.access_token}&refresh={refresh}&username={email}&first_name={name}&image={image_url}"
        return redirect(react_redirect_url)
```

---

### 💻 4. **Frontend - Google Button**

```js
const handleGoogleLogin = () => {
  const clientId = 'your_id';
  const redirectUri = encodeURIComponent('https://yourdomain/api/accounts/auth/google/callback/');
  const scope = encodeURIComponent('openid email profile');
  const authUrl = `https://accounts.google.com/o/oauth2/v2/auth?client_id=${clientId}&redirect_uri=${redirectUri}&response_type=code&scope=${scope}&access_type=offline&prompt=consent`;
  window.location.href = authUrl;
};
```

---

## 🏢 Microsoft OAuth Login

### 🔧 1. **Register Microsoft App**

Go to [Microsoft Azure Portal](https://portal.azure.com):
- App Registrations → New Registration.
- Set redirect URI to:
  ```
  https://api.novel.mohuls.com/api/accounts/auth/microsoft/callback/
  ```
- Copy:
  - `Client ID`
  - `Client Secret`
  - `Tenant ID` (or use `common`)

---

### ⚙️ 2. **Backend Settings**

```python
MICROSOFT_CLIENT_ID = config('MICROSOFT_CLIENT_ID')
MICROSOFT_CLIENT_SECRET = config('MICROSOFT_CLIENT_SECRET')
MICROSOFT_REDIRECT_URI = 'https://yourdomain/api/accounts/auth/microsoft/callback/'
```

---

### 🧠 3. **Microsoft Callback View**

```python
class MicrosoftAuthCallbackView(APIView):
    permission_classes = [AllowAny]

    def get(self, request):
        code = request.GET.get('code')
        if not code:
            return Response({'error': 'Missing code'}, status=400)

        token_res = requests.post('https://login.microsoftonline.com/common/oauth2/v2.0/token', data={
            'client_id': MICROSOFT_CLIENT_ID,
            'client_secret': MICROSOFT_CLIENT_SECRET,
            'grant_type': 'authorization_code',
            'code': code,
            'redirect_uri': MICROSOFT_REDIRECT_URI,
        }, headers={'Content-Type': 'application/x-www-form-urlencoded'})

        if token_res.status_code != 200:
            return Response({'error': 'Token exchange failed'}, status=400)
        access_token = token_res.json().get('access_token')
        if not access_token:
            return Response({'error': 'Missing access token'}, status=400)

        # Fetch Microsoft user profile
        headers = {'Authorization': f'Bearer {access_token}'}
        profile_res = requests.get('https://graph.microsoft.com/v1.0/me?$select=displayName,mail,userPrincipalName', headers=headers)
        if profile_res.status_code != 200:
            return Response({'error': 'Failed to get profile'}, status=400)

        profile_data = profile_res.json()
        email = profile_data.get('mail') or profile_data.get('userPrincipalName')
        name = profile_data.get('displayName')
        user, _ = User.objects.get_or_create(username=email, defaults={'email': email, 'first_name': name})

        profile = UserProfile.objects.filter(user=user).first()
        # Optional photo
        img_res = requests.get('https://graph.microsoft.com/v1.0/me/photo/$value', headers=headers)
        if img_res.status_code == 200 and profile:
            profile.image.save(f'{user.username}_microsoft.jpg', ContentFile(img_res.content), save=True)

        image_url = profile.image.url if profile and profile.image else ''
        
        refresh = RefreshToken.for_user(user)
        return redirect(
            f"https://localhost:3000/?access={refresh.access_token}&refresh={refresh}&username={email}&first_name={name}&image={image_url}"
        )
```

---

### 💻 4. **Frontend - Microsoft Button**

```js
const handleMicrosoftLogin = () => {
  const clientId = '29bae955-c8e7-482c-a175-fa6e30f0c00f';
  const redirectUri = encodeURIComponent('https://api.novel.mohuls.com/api/accounts/auth/microsoft/callback/');
  const scope = encodeURIComponent('openid profile email User.Read');
  const authUrl = `https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=${clientId}&response_type=code&redirect_uri=${redirectUri}&response_mode=query&scope=${scope}&prompt=select_account`;
  window.location.href = authUrl;
};
```

---

## ✅ Final Notes

- Secure your secrets using environment variables or Django settings management tools (e.g., `python-decouple` or `django-environ`).
- Make sure CORS is configured properly if frontend/backend are on different domains.
- Tokens in the URL should be handled securely and removed after reading them on the React side.

---

Would you like a React example to **store tokens in localStorage** and show logged-in user info after redirect?
