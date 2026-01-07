# Authentication API Documentation

## Overview

This API provides secure user authentication using Laravel Sanctum with token-based authentication. It supports user registration, login, logout, token refresh, and user profile retrieval.

**Base URL**: `http://127.0.0.1:8000/api`

---

## API Endpoints

### 1. Register

Create a new user account.

**Endpoint**: `POST /register`

**Authentication**: Not required

**Request Body**:
```json
{
  "username": "johndoe",
  "email": "john@example.com",
  "display_name": "John Doe",
  "phone_number": "1234567890",
  "password": "SecurePassword123!",
  "password_confirmation": "SecurePassword123!"
}
```

**Validation Rules**:
- `username`: Required, string, max 255 characters, must be unique
- `email`: Required, valid email format, max 255 characters, must be unique
- `display_name`: Required, string, max 255 characters
- `phone_number`: Optional, string, max 15 characters
- `password`: Required, must be confirmed, must meet password requirements
- `password_confirmation`: Required, must match password

**Success Response** (201 Created):
```json
{
  "message": "User registered successfully",
  "user": {
    "id": 1,
    "username": "johndoe",
    "email": "john@example.com",
    "display_name": "John Doe",
    "phone_number": "1234567890",
    "profile_picture_url": "",
    "status": "active",
    "created_at": "2024-01-08T10:30:00.000000Z",
    "updated_at": "2024-01-08T10:30:00.000000Z"
  },
  "access_token": "1|abcdefghijklmnopqrstuvwxyz1234567890",
  "token_type": "Bearer"
}
```

**Error Responses**:
- `422 Unprocessable Entity`: Validation errors
- `500 Internal Server Error`: Database or server error

---

### 2. Login

Authenticate a user and receive an access token.

**Endpoint**: `POST /login`

**Authentication**: Not required

**Request Body**:
```json
{
  "email": "john@example.com",
  "password": "SecurePassword123!"
}
```

**Success Response** (200 OK):
```json
{
  "message": "Login successful",
  "user": {
    "id": 1,
    "username": "johndoe",
    "email": "john@example.com",
    "display_name": "John Doe",
    "phone_number": "1234567890",
    "profile_picture_url": "",
    "status": "active",
    "created_at": "2024-01-08T10:30:00.000000Z",
    "updated_at": "2024-01-08T10:30:00.000000Z"
  },
  "access_token": "2|zyxwvutsrqponmlkjihgfedcba0987654321",
  "token_type": "Bearer"
}
```

**Error Responses**:
- `401 Unauthorized`: Invalid credentials
- `422 Unprocessable Entity`: Validation errors

---

### 3. Logout

Revoke the current user's access token.

**Endpoint**: `POST /logout`

**Authentication**: Required (Bearer Token)

**Headers**:
```
Authorization: Bearer {access_token}
```

**Request Body**: None

**Success Response** (200 OK):
```json
{
  "message": "Successfully logged out"
}
```

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token

---

### 4. Get User Profile

Retrieve the authenticated user's profile information.

**Endpoint**: `GET /user`

**Authentication**: Required (Bearer Token)

**Headers**:
```
Authorization: Bearer {access_token}
```

**Success Response** (200 OK):
```json
{
  "id": 1,
  "username": "johndoe",
  "email": "john@example.com",
  "display_name": "John Doe",
  "phone_number": "1234567890",
  "profile_picture_url": "",
  "bio": null,
  "status": "active",
  "last_seen": null,
  "created_at": "2024-01-08T10:30:00.000000Z",
  "updated_at": "2024-01-08T10:30:00.000000Z"
}
```

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token

---

### 5. Refresh Token

Generate a new access token and revoke all existing tokens.

**Endpoint**: `POST /refresh`

**Authentication**: Required (Bearer Token)

**Headers**:
```
Authorization: Bearer {access_token}
```

**Request Body**: None

**Success Response** (200 OK):
```json
{
  "access_token": "3|newTokenStringHere1234567890abcdefgh",
  "token_type": "Bearer"
}
```

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token

---

## React Native Implementation

### Setup

Install required dependencies:

```bash
npm install axios
# or
yarn add axios
```

### Create API Client

Create a file `src/services/api.js`:

```javascript
import axios from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';

const API_BASE_URL = 'http://127.0.0.1:8000/api';

// Create axios instance
const api = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

// Add token to requests
api.interceptors.request.use(
  async (config) => {
    const token = await AsyncStorage.getItem('access_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Handle token expiration
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Token expired or invalid - clear storage and redirect to login
      await AsyncStorage.removeItem('access_token');
      await AsyncStorage.removeItem('user');
      // Navigate to login screen here
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Create Authentication Service

Create a file `src/services/authService.js`:

```javascript
import api from './api';
import AsyncStorage from '@react-native-async-storage/async-storage';

export const authService = {
  // Register new user
  register: async (userData) => {
    try {
      const response = await api.post('/register', userData);
      const { access_token, user } = response.data;
      
      // Save token and user data
      await AsyncStorage.setItem('access_token', access_token);
      await AsyncStorage.setItem('user', JSON.stringify(user));
      
      return response.data;
    } catch (error) {
      throw error.response?.data || error.message;
    }
  },

  // Login user
  login: async (email, password) => {
    try {
      const response = await api.post('/login', { email, password });
      const { access_token, user } = response.data;
      
      // Save token and user data
      await AsyncStorage.setItem('access_token', access_token);
      await AsyncStorage.setItem('user', JSON.stringify(user));
      
      return response.data;
    } catch (error) {
      throw error.response?.data || error.message;
    }
  },

  // Logout user
  logout: async () => {
    try {
      await api.post('/logout');
      
      // Clear local storage
      await AsyncStorage.removeItem('access_token');
      await AsyncStorage.removeItem('user');
      
      return true;
    } catch (error) {
      // Clear local storage even if API call fails
      await AsyncStorage.removeItem('access_token');
      await AsyncStorage.removeItem('user');
      throw error.response?.data || error.message;
    }
  },

  // Get current user
  getCurrentUser: async () => {
    try {
      const response = await api.get('/user');
      await AsyncStorage.setItem('user', JSON.stringify(response.data));
      return response.data;
    } catch (error) {
      throw error.response?.data || error.message;
    }
  },

  // Refresh token
  refreshToken: async () => {
    try {
      const response = await api.post('/refresh');
      const { access_token } = response.data;
      
      await AsyncStorage.setItem('access_token', access_token);
      
      return response.data;
    } catch (error) {
      throw error.response?.data || error.message;
    }
  },

  // Check if user is authenticated
  isAuthenticated: async () => {
    const token = await AsyncStorage.getItem('access_token');
    return !!token;
  },

  // Get stored user data
  getStoredUser: async () => {
    const userStr = await AsyncStorage.getItem('user');
    return userStr ? JSON.parse(userStr) : null;
  },
};
```

### Usage Examples

#### Register Screen Example

```javascript
import React, { useState } from 'react';
import { View, TextInput, Button, Alert } from 'react-native';
import { authService } from '../services/authService';

const RegisterScreen = ({ navigation }) => {
  const [formData, setFormData] = useState({
    username: '',
    email: '',
    display_name: '',
    phone_number: '',
    password: '',
    password_confirmation: '',
  });
  const [loading, setLoading] = useState(false);

  const handleRegister = async () => {
    setLoading(true);
    try {
      await authService.register(formData);
      Alert.alert('Success', 'Registration successful!');
      navigation.navigate('Home');
    } catch (error) {
      Alert.alert('Error', error.message || 'Registration failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View>
      <TextInput
        placeholder="Username"
        value={formData.username}
        onChangeText={(text) => setFormData({ ...formData, username: text })}
      />
      <TextInput
        placeholder="Email"
        value={formData.email}
        onChangeText={(text) => setFormData({ ...formData, email: text })}
        keyboardType="email-address"
        autoCapitalize="none"
      />
      <TextInput
        placeholder="Display Name"
        value={formData.display_name}
        onChangeText={(text) => setFormData({ ...formData, display_name: text })}
      />
      <TextInput
        placeholder="Phone Number (Optional)"
        value={formData.phone_number}
        onChangeText={(text) => setFormData({ ...formData, phone_number: text })}
        keyboardType="phone-pad"
      />
      <TextInput
        placeholder="Password"
        value={formData.password}
        onChangeText={(text) => setFormData({ ...formData, password: text })}
        secureTextEntry
      />
      <TextInput
        placeholder="Confirm Password"
        value={formData.password_confirmation}
        onChangeText={(text) => setFormData({ ...formData, password_confirmation: text })}
        secureTextEntry
      />
      <Button title="Register" onPress={handleRegister} disabled={loading} />
    </View>
  );
};

export default RegisterScreen;
```

#### Login Screen Example

```javascript
import React, { useState } from 'react';
import { View, TextInput, Button, Alert } from 'react-native';
import { authService } from '../services/authService';

const LoginScreen = ({ navigation }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    setLoading(true);
    try {
      await authService.login(email, password);
      navigation.navigate('Home');
    } catch (error) {
      Alert.alert('Error', error.message || 'Login failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View>
      <TextInput
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
      />
      <TextInput
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
      />
      <Button title="Login" onPress={handleLogin} disabled={loading} />
    </View>
  );
};

export default LoginScreen;
```

#### Protected Screen with Logout

```javascript
import React, { useEffect, useState } from 'react';
import { View, Text, Button } from 'react-native';
import { authService } from '../services/authService';

const HomeScreen = ({ navigation }) => {
  const [user, setUser] = useState(null);

  useEffect(() => {
    loadUser();
  }, []);

  const loadUser = async () => {
    try {
      const userData = await authService.getCurrentUser();
      setUser(userData);
    } catch (error) {
      console.error('Failed to load user:', error);
    }
  };

  const handleLogout = async () => {
    try {
      await authService.logout();
      navigation.navigate('Login');
    } catch (error) {
      console.error('Logout error:', error);
    }
  };

  return (
    <View>
      {user && (
        <>
          <Text>Welcome, {user.display_name}!</Text>
          <Text>Email: {user.email}</Text>
          <Text>Username: {user.username}</Text>
        </>
      )}
      <Button title="Logout" onPress={handleLogout} />
    </View>
  );
};

export default HomeScreen;
```

### Authentication Context (Optional but Recommended)

Create `src/contexts/AuthContext.js` for global state management:

```javascript
import React, { createContext, useState, useEffect, useContext } from 'react';
import { authService } from '../services/authService';

const AuthContext = createContext({});

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    checkAuth();
  }, []);

  const checkAuth = async () => {
    try {
      const isAuth = await authService.isAuthenticated();
      if (isAuth) {
        const userData = await authService.getStoredUser();
        setUser(userData);
      }
    } catch (error) {
      console.error('Auth check failed:', error);
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await authService.login(email, password);
    setUser(response.user);
    return response;
  };

  const register = async (userData) => {
    const response = await authService.register(userData);
    setUser(response.user);
    return response;
  };

  const logout = async () => {
    await authService.logout();
    setUser(null);
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        loading,
        login,
        register,
        logout,
        isAuthenticated: !!user,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

---

## Error Handling

Common error responses and how to handle them:

| Status Code | Meaning | Action |
|-------------|---------|--------|
| 401 | Unauthorized - Invalid or expired token | Redirect to login, clear stored token |
| 422 | Validation Error | Display field-specific errors to user |
| 500 | Server Error | Show generic error message, retry later |

Example error handling:

```javascript
try {
  await authService.login(email, password);
} catch (error) {
  if (error.errors) {
    // Validation errors (422)
    Object.keys(error.errors).forEach(key => {
      console.log(`${key}: ${error.errors[key].join(', ')}`);
    });
  } else if (error.message) {
    // General error
    console.log(error.message);
  }
}
```

---

## Security Best Practices

1. **Never log or display tokens** - Keep access tokens secure
2. **Use HTTPS in production** - Replace `http://` with `https://`
3. **Store tokens securely** - Use `@react-native-async-storage/async-storage` or secure storage
4. **Implement token refresh** - Refresh tokens before they expire
5. **Clear tokens on logout** - Always remove tokens from storage
6. **Handle 401 errors** - Automatically redirect to login on authentication failures
7. **Validate input** - Always validate user input on the client side
8. **Use secure passwords** - Enforce strong password requirements

---

## Testing with Postman

### Register Request
```
POST http://127.0.0.1:8000/api/register
Content-Type: application/json

{
  "username": "testuser",
  "email": "test@example.com",
  "display_name": "Test User",
  "phone_number": "1234567890",
  "password": "SecurePass123!",
  "password_confirmation": "SecurePass123!"
}
```

### Login Request
```
POST http://127.0.0.1:8000/api/login
Content-Type: application/json

{
  "email": "test@example.com",
  "password": "SecurePass123!"
}
```

### Logout Request
```
POST http://127.0.0.1:8000/api/logout
Authorization: Bearer {your_token_here}
```

### Get User Request
```
GET http://127.0.0.1:8000/api/user
Authorization: Bearer {your_token_here}
```

---

## Troubleshooting

### Common Issues

**Issue**: 500 Internal Server Error on logout
- **Solution**: Ensure you're sending the Bearer token in the Authorization header

**Issue**: 401 Unauthorized on protected routes
- **Solution**: Verify token is stored and included in request headers

**Issue**: CORS errors in development
- **Solution**: Configure CORS in Laravel (`config/cors.php`) to allow your React Native app's origin

**Issue**: Token not persisting
- **Solution**: Check AsyncStorage implementation and ensure proper await/async usage

---

## Additional Resources

- [Laravel Sanctum Documentation](https://laravel.com/docs/sanctum)
- [Axios Documentation](https://axios-http.com/docs/intro)
- [React Native AsyncStorage](https://react-native-async-storage.github.io/async-storage/)
